# Uhae451 개발 블로그

[![Blog](https://img.shields.io/badge/Blog-uhae451.github.io-2ea44f)](https://uhae451.github.io)

Java와 Spring을 중심으로 백엔드 개발을 공부하고, 직접 구현하며 알게 된 내용을 기록하는 기술 블로그입니다. 개념을 코드와 실행 결과에 연결하고, 문제를 만났을 때 원인을 확인하고 해결한 과정도 함께 정리합니다.

## 주요 주제

| 분야 | 다루는 내용 |
| --- | --- |
| Java · 객체지향 | SOLID 원칙, 디자인 패턴 |
| Spring | Bean, DI, AOP, MVC 요청 흐름, 검증, 예외 처리, 트랜잭션 |
| JPA | 영속성 컨텍스트, 엔티티 매핑, Repository, 쿼리 메서드, 성능 최적화 |
| 테스트 | JUnit 5, Mockito, `@DataJpaTest`, `@WebMvcTest` |
| Troubleshooting | 테스트 설정 오류와 쿼리 문제를 재현하고 원인 및 해결 과정을 기록 |
| 프로젝트 · DevOps | 프로젝트 개발 기록, Docker와 PostgreSQL 구성 |

전체 글은 [블로그](https://uhae451.github.io)에서 확인할 수 있습니다. 주제별 글은 [카테고리](https://uhae451.github.io/categories/)에서, 키워드별 글은 [태그](https://uhae451.github.io/tags/)에서 찾아볼 수 있습니다.

## 저장소 구성

```text
_posts/       블로그 게시글
_tabs/        About, Categories, Tags, Archives 페이지
_config.yml  사이트 이름, 주소, 언어, 시간대, 테마 설정
assets/       이미지, 스타일, 스크립트 및 정적 리소스
```

게시글은 `_posts`에 `YYYY-MM-DD-제목.md` 형식으로 작성합니다. 각 글의 맨 위에는 제목, 날짜, 카테고리, 태그, 설명을 YAML front matter로 지정합니다.

```yaml
---
title: "글 제목"
date: 2026-11-08 09:00:00 +0900
categories: ["Troubleshooting"]
tags: ["spring", "testing"]
description: "글에서 다루는 내용을 한 문장으로 설명합니다."
toc: true
---
```

## 로컬에서 실행

Ruby와 Bundler를 설치한 뒤 의존성을 설치하고 Jekyll 서버를 실행합니다.

```bash
bundle install
bundle exec jekyll serve
```

브라우저에서 `http://127.0.0.1:4000`을 열면 로컬 미리보기를 볼 수 있습니다.

## 기술 스택

- Jekyll · Ruby
- Chirpy 테마
- GitHub Pages

게시글을 수정하거나 추가한 뒤 `master` 브랜치에 푸시하면 GitHub Actions가 사이트를 빌드하고 배포합니다.
