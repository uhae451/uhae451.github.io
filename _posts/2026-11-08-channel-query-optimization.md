---
title: "채널 목록 조회 개선: 2N + 2번의 쿼리를 3번으로"
date: 2026-11-07 23:00:00 +0900
categories: ["Spring", "JPA", "Performance"]
tags: ["spring-data-jpa", "query-optimization", "n-plus-one", "entity-graph"]
description: "채널 목록 조회에서 반복되던 참여자와 마지막 메시지 조회를 일괄 처리하고 SQL 로그로 쿼리 수를 확인한다."
toc: true
---

# 채널 목록 조회 개선: 2N + 2번의 쿼리를 3번으로

채팅 서비스의 채널 목록 API를 구현하고 코드 리뷰를 받았습니다. 전체 채널을 조회한 뒤 자바에서 필터링하고, DTO를 만들 때 채널마다 추가 조회가 발생한다는 내용이었습니다.

응답은 정상적으로 나왔지만, 코드를 따라가 보니 `toDto()` 안에서 DB를 조회하고 있었습니다. 반복 조회를 일괄 조회로 바꾸고 SQL 로그로 확인한 과정을 정리합니다.

## 1. 문제 상황: DTO 변환 한 번에 조회 두 번

기존 서비스는 사용자의 참여 채널을 확인하고 전체 채널을 조회한 뒤, 공개 채널이거나 참여한 채널이면 DTO로 변환했습니다.

```java
// 기존 findAllByUserId()의 핵심 부분
for (Channel channel : channelRepository.findAll()) {
  boolean isPublic = ChannelType.PUBLIC.equals(channel.getType());
  boolean isParticipant = joinedChannelIds.contains(channel.getId());

  if (isPublic || isParticipant) {
    result.add(channelMapper.toDto(channel));
  }
}
```

문제는 `channelMapper.toDto(channel)` 안에 있었습니다.

```java
@Mapping(target = "participants",
    expression = "java(findParticipants(channel))")
@Mapping(target = "lastMessageAt",
    expression = "java(findLastMessageAt(channel))")
public abstract ChannelDto toDto(Channel channel);
```

`findParticipants()`는 참여자를, `findLastMessageAt()`은 마지막 메시지를 조회합니다. Mapper를 한 번 호출할 때 SQL 두 개가 실행되는 구조입니다.

```text
채널 A → 참여자 조회 + 마지막 메시지 조회
채널 B → 참여자 조회 + 마지막 메시지 조회
채널 C → 참여자 조회 + 마지막 메시지 조회
```

응답 채널이 N개라면 기본 조회 횟수는 다음과 같습니다.

```text
사용자의 참여 채널 확인 1회 + 전체 채널 조회 1회
+ 참여자 조회 N회 + 마지막 메시지 조회 N회
= 2N + 2회
```

**MapStruct가 자동으로 쿼리를 반복하는 것이 아닙니다. 직접 작성한 Repository 호출이 DTO 변환마다 실행되는 것이 원인입니다.**

## 2. 수정: 필요한 데이터를 먼저 묶어서 조회

조회 흐름을 다음처럼 변경했습니다. 아래 코드는 변경한 핵심 부분만 발췌했습니다.

```text
표시할 채널 조회 → 참여자 일괄 조회 → 마지막 메시지 시각 일괄 조회
→ 준비된 데이터로 DTO 변환
```

### 표시할 채널만 조회

전체 조회 후 필터링하던 조건을 Repository로 옮겼습니다.

```java
@Query("""
    select c from Channel c
    where c.type = :publicType
       or exists (
           select r.id from ReadStatus r
           where r.channel = c and r.user.id = :userId
       )
    """)
List<Channel> findVisibleChannels(
    @Param("userId") UUID userId,
    @Param("publicType") ChannelType publicType
);
```

공개 채널 전체와 자신이 참여한 비공개 채널을 가져옵니다. 참여 여부만 필요하므로 `exists`로 확인합니다.

### 참여자와 마지막 메시지 시각 일괄 조회

참여자는 채널 ID 목록으로 한 번에 조회합니다. DTO 변환에 필요한 사용자·프로필·접속 상태도 함께 가져옵니다.

```java
@EntityGraph(attributePaths = {"user", "user.profile", "user.status"})
List<ReadStatus> findAllByChannelIdIn(List<UUID> channelIds);
```

마지막 메시지는 내용이 아니라 시각만 필요합니다. 조회용 DTO에 채널별 `MAX(createdAt)`을 담았습니다.

```java
public record ChannelLastMessageAt(UUID channelId, Instant lastMessageAt) {}

// MessageRepository
@Query("""
    select new com.sprint.mission.discodeit.dto.ChannelLastMessageAt(
        m.channel.id, max(m.createdAt)
    )
    from Message m
    where m.channel.id in :channelIds
    group by m.channel.id
    """)
List<ChannelLastMessageAt> findLastMessageAtByChannelIds(
    @Param("channelIds") List<UUID> channelIds
);
```

### Mapper에는 조회한 값 전달

서비스에서 조회 결과를 채널 ID별 `Map`으로 묶고, 변환할 때 전달했습니다.

```java
// 일괄 조회 결과를 담는 Map
Map<UUID, List<UserDto>> participantsByChannelId = new HashMap<>();
Map<UUID, Instant> lastMessageAtByChannelId = new HashMap<>();

// 중간의 조회 및 Map 구성 코드는 생략
for (Channel channel : channels) {
  result.add(channelMapper.toDto(
      channel,
      participantsByChannelId.get(channel.getId()),
      lastMessageAtByChannelId.get(channel.getId())
  ));
}
```

새 Mapper 메서드는 전달받은 값으로 DTO만 만듭니다.

```java
@Mapping(target = "participants", source = "participants")
@Mapping(target = "lastMessageAt", source = "lastMessageAt")
public abstract ChannelDto toDto(
    Channel channel, List<UserDto> participants, Instant lastMessageAt
);
```

참여자가 없으면 빈 목록, 메시지가 없으면 `null`을 전달하도록 처리했습니다. 기존 단건용 `toDto(channel)`는 유지하고 목록 조회에만 새 메서드를 사용했습니다.

**Mapper는 여전히 N번 호출하지만, 이제 목록 변환 반복문 안에서는 SQL이 실행되지 않습니다.**

## 3. 실제 로그 확인

공개 채널 3개를 생성한 뒤 `BasicChannelService.findAllByUserId()` 한 번의 트랜잭션을 확인했습니다. 아래는 실제 로그에서 공통 접두사와 일부 컬럼·JOIN을 생략한 내용입니다.

```sql
-- 17:43:15.327: 표시할 채널 조회
select ...
from channels c1_0
where c1_0.type=?
   or exists (
       select rs1_0.id from read_statuses rs1_0
       where rs1_0.channel_id=c1_0.id and rs1_0.user_id=?
   )

-- 17:43:15.331: 참여자 일괄 조회 (JOIN 부분 생략)
select ...
from read_statuses rs1_0
...
where rs1_0.channel_id in (?, ?, ?)

-- 17:43:15.349: 마지막 메시지 시각 일괄 조회
select m1_0.channel_id, max(m1_0.created_at)
from messages m1_0
where m1_0.channel_id in (?, ?, ?)
group by m1_0.channel_id
```

`IN (?, ?, ?)`에 채널 ID 3개가 전달됩니다. 채널별로 따로 조회하지 않고 한 번에 처리하는 것을 확인할 수 있습니다.

채널을 하나씩 생성한 직후의 목록 조회도 비교했습니다.

| 목록 조회 시작 시각 | 채널 수 | SQL 횟수 |
|---|---:|---:|
| 17:42:59.056 | 1개 | 3회 |
| 17:43:05.046 | 2개 | 3회 |
| 17:43:08.157 | 3개 | 3회 |

기존 코드에서 계산한 **2N + 2회 구조를, 실제 로그 기준 3회 조회로 개선**했습니다. 채널이 없으면 첫 조회 후 바로 반환하므로 1회입니다.

프런트엔드가 3초마다 목록을 요청해도 요청당 기본 SQL은 3개입니다. 다만 채널·참여자가 많아지면 데이터 양은 늘어납니다. 이번에는 쿼리 횟수를 확인했으며, 대량 데이터의 응답 시간까지 측정한 것은 아닙니다.

## 4. 정리

- DTO 변환 메서드 안에도 DB 조회가 숨어 있을 수 있습니다.
- 목록에 필요한 데이터는 일괄 조회하고, 반복문에서는 준비된 값으로 DTO를 만듭니다.
- SQL 로그는 전체 개수가 아니라 **목록 요청 한 번의 실행 범위**를 구분해서 확인합니다.

**오늘 배운 것 한 줄 요약:** 채널별로 반복하던 조회를 일괄 조회로 바꾸고, 실제 로그에서 채널 수가 늘어도 SQL 3회가 유지되는 것을 확인했습니다.
