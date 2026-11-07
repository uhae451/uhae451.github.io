---
title: "BFS(Breadth First Search)"
date: 2026-10-19 09:00:00 +0900
categories: ["Algorithm", "Graph"]
tags: ["algorithm", "bfs", "graph", "queue", "java"]
description: "너비 우선 탐색(BFS)의 동작 원리와 큐를 사용하는 이유, 그래프 탐색 구현 및 코딩 테스트 활용 방법을 정리한다."
toc: true
---

BFS는 **시작 지점에서 가까운 정점부터 차례대로 탐색하는 그래프 탐색 알고리즘**이다.

한국어로는 **너비 우선 탐색**이라고 한다.

코딩테스트에서는 단순히 그래프를 탐색하는 용도뿐만 아니라 다음과 같은 문제에서 자주 사용한다.

- 최단 거리
- 최소 이동 횟수
- 미로 탐색
- 연결된 영역 탐색
- 특정 거리의 노드 찾기
- 여러 시작점에서 동시에 탐색

BFS에서 가장 중요한 특징은 **Queue를 사용한다는 것**이다.

---

## 1. BFS의 핵심 개념

다음과 같은 그래프가 있다고 가정한다.

![](https://velog.velcdn.com/images/uhae451/post/abb08760-f97a-4ada-be39-5a77dee3201d/image.png)


1번 노드에서 BFS 탐색을 시작하면 방문 순서는 다음과 같다.

```text
1, 2, 3, 4, 5, 6
```

먼저 시작점과 가까운 노드를 모두 방문한 뒤 그다음 거리의 노드를 탐색한다.

| 거리 | 방문 노드 |
|---|---|
| 0 | 1 |
| 1 | 2, 3 |
| 2 | 4, 5, 6 |

이처럼 BFS는 같은 거리의 노드를 먼저 처리한다.

이 성질 때문에 **가중치가 없는 그래프에서 최단 거리**를 구하는 문제에 특히 적합하다.

---

## 2. BFS는 왜 Queue를 사용하는가

BFS에서는 먼저 발견한 노드를 먼저 탐색해야 한다.

Queue는 **FIFO(First In First Out)** 구조를 가진다.

```text
입력 순서

1
2
3
4

처리 순서

1
2
3
4
```

현재 노드에서 연결된 노드를 Queue에 넣고, Queue에서 가장 먼저 들어온 노드부터 꺼내면서 탐색한다.

따라서 자연스럽게 시작점에서 가까운 노드부터 처리된다.

Java에서는 코딩테스트에서 보통 `ArrayDeque`를 사용한다.

```java
Queue<Integer> queue = new ArrayDeque<>();
```

`LinkedList`도 Queue 구현체이지만 단순 Queue 용도라면 일반적으로 `ArrayDeque`를 사용하는 것이 좋다.

---

## 3. BFS의 기본 동작

BFS는 크게 다음 과정으로 구현한다.

1. 시작 노드를 Queue에 넣는다.
2. 시작 노드를 방문 처리한다.
3. Queue에서 노드를 하나 꺼낸다.
4. 해당 노드와 연결된 노드를 확인한다.
5. 아직 방문하지 않은 노드를 방문 처리하고 Queue에 넣는다.
6. Queue가 빌 때까지 반복한다.

핵심 구성 요소는 세 가지다.

```java
Queue
visited
반복문
```

---

## 4. 가장 기본적인 BFS 구현

그래프가 인접 리스트 형태로 주어졌다고 가정한다.

```java
import java.util.*;

public class Main {

    static List<Integer>[] graph;
    static boolean[] visited;

    static void bfs(int start) {

        Queue<Integer> queue = new ArrayDeque<>();

        queue.offer(start);
        visited[start] = true;

        while (!queue.isEmpty()) {

            int current = queue.poll();

            System.out.println(current);

            for (int next : graph[current]) {

                if (visited[next]) {
                    continue;
                }

                visited[next] = true;
                queue.offer(next);
            }
        }
    }
}
```

BFS 문제에서 가장 기본이 되는 형태다.

특히 다음 세 줄의 의미를 정확하게 이해하는 것이 중요하다.

```java
queue.offer(start);
visited[start] = true;
```

시작점을 Queue에 넣으면서 방문 처리한다.

그리고 반복문에서

```java
int current = queue.poll();
```

현재 탐색할 노드를 꺼낸다.

이후

```java
for (int next : graph[current])
```

현재 노드와 연결된 노드를 검사한다.

---

## 5. visited가 필요한 이유

그래프에서는 서로 연결된 노드를 다시 방문할 수 있다.

예를 들어 다음과 같은 그래프가 있다고 생각해보자.
![](https://velog.velcdn.com/images/uhae451/post/e8b9d8cc-3ee1-46ab-a0dd-fce6e3a9643c/image.png)


방문 여부를 확인하지 않으면 다음과 같은 상황이 반복될 수 있다.

```text
1에서 2 방문
2에서 3 방문
3에서 4 방문
4에서 다시 1 방문
```

그래서 일반적인 BFS에서는 반드시 방문 여부를 관리한다.

```java
boolean[] visited = new boolean[n + 1];
```

방문하지 않은 노드만 Queue에 넣는다.

```java
if (!visited[next]) {
    visited[next] = true;
    queue.offer(next);
}
```

---

## 6. 방문 처리는 언제 해야 하는가

BFS에서 자주 실수하는 부분이다.

방문 처리는 일반적으로 **Queue에 넣을 때** 한다.

```java
visited[next] = true;
queue.offer(next);
```

다음처럼 Queue에서 꺼낼 때 방문 처리하는 것은 피하는 것이 좋다.

```java
int current = queue.poll();
visited[current] = true;
```

이렇게 하면 하나의 노드가 여러 경로를 통해 Queue에 중복으로 들어갈 수 있다.

예를 들어 2번과 3번 노드가 모두 4번과 연결되어 있다면

```text
2가 4 발견
3도 4 발견
```

4가 아직 Queue에서 나오지 않았기 때문에 두 번 들어갈 수 있다.

따라서 기본적으로 다음 순서를 기억하면 된다.

```java
방문 여부 확인
방문 처리
Queue 삽입
```

---

## 7. BFS와 최단 거리

BFS의 가장 중요한 활용 중 하나다.

**모든 간선의 이동 비용이 동일하다면 BFS로 최단 거리를 구할 수 있다.**

예를 들어 그래프에서

```text
1번 노드에서 각 노드까지 몇 번 이동해야 하는가?
```

를 구한다고 해보자.

거리 배열을 만든다.

```java
int[] distance = new int[n + 1];
Arrays.fill(distance, -1);
```

`-1`을 아직 방문하지 않은 상태로 사용할 수 있다.

```java
static void bfs(int start) {

    Queue<Integer> queue = new ArrayDeque<>();

    queue.offer(start);
    distance[start] = 0;

    while (!queue.isEmpty()) {

        int current = queue.poll();

        for (int next : graph[current]) {

            if (distance[next] != -1) {
                continue;
            }

            distance[next] = distance[current] + 1;
            queue.offer(next);
        }
    }
}
```

이 경우 `distance` 자체가 방문 배열 역할도 한다.

예를 들어

```java
distance[5] = 3;
```

이라면 시작점에서 5번 노드까지 최소 3번 이동해야 한다는 의미다.

---

## 8. 미로 문제에서 BFS 사용하기

코딩테스트에서 BFS가 가장 많이 등장하는 형태 중 하나가 **2차원 배열 탐색**이다.

예를 들어 다음과 같은 미로가 있다고 가정한다.
![](https://velog.velcdn.com/images/uhae451/post/00b102ba-2426-48e0-8050-66a2625ed1aa/image.png)


조건은 다음과 같다.

```text
1 : 이동 가능
0 : 이동 불가능
```

상하좌우로 이동해야 한다면 방향 배열을 사용한다.

```java
static int[] dx = {-1, 1, 0, 0};
static int[] dy = {0, 0, -1, 1};
```

각각

```text
상
하
좌
우
```

이동을 의미한다.

---

### 8.1 좌표를 Queue에 저장하기

2차원 BFS에서는 Queue에 노드 번호 대신 좌표를 저장한다.

```java
static class Point {

    int x;
    int y;

    Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

Queue는 다음처럼 만든다.

```java
Queue<Point> queue = new ArrayDeque<>();
```

---

### 8.2 2차원 BFS 기본 코드

```java
static void bfs(int startX, int startY) {

    Queue<Point> queue = new ArrayDeque<>();

    queue.offer(new Point(startX, startY));
    visited[startX][startY] = true;

    while (!queue.isEmpty()) {

        Point current = queue.poll();

        for (int i = 0; i < 4; i++) {

            int nx = current.x + dx[i];
            int ny = current.y + dy[i];

            if (nx < 0 || nx >= n || ny < 0 || ny >= m) {
                continue;
            }

            if (map[nx][ny] == 0) {
                continue;
            }

            if (visited[nx][ny]) {
                continue;
            }

            visited[nx][ny] = true;
            queue.offer(new Point(nx, ny));
        }
    }
}
```

2차원 BFS 문제는 대부분 이 구조를 기반으로 조건만 변경된다.

---

## 9. 2차원 BFS에서 반드시 확인할 조건

격자 문제에서는 보통 다음 세 가지를 검사한다.

### 1. 배열 범위를 벗어나는가

```java
if (nx < 0 || nx >= n || ny < 0 || ny >= m) {
    continue;
}
```

### 2. 이동할 수 있는 칸인가

```java
if (map[nx][ny] == 0) {
    continue;
}
```

### 3. 이미 방문한 칸인가

```java
if (visited[nx][ny]) {
    continue;
}
```

세 조건을 통과하면 이동할 수 있다.

```java
visited[nx][ny] = true;
queue.offer(new Point(nx, ny));
```

---

## 10. 2차원 최단 거리 구하기

미로의 최단 거리를 구한다면 `visited` 대신 거리 배열을 사용할 수도 있다.

```java
int[][] distance = new int[n][m];
```

여기서는 시작 칸도 포함한 **방문 칸 수**를 거리로 기록한다. 이동 횟수가 필요하면 도착지의 값에서 1을 빼거나, 미방문 값을 -1로 초기화하고 시작 거리를 0으로 둔다.

```java
distance[startX][startY] = 1;
```

다음 위치의 거리는 현재 거리 + 1이다.

```java
distance[nx][ny] = distance[current.x][current.y] + 1;
```

전체 코드는 다음 형태가 된다.

```java
static void bfs(int startX, int startY) {

    Queue<Point> queue = new ArrayDeque<>();

    queue.offer(new Point(startX, startY));
    distance[startX][startY] = 1;

    while (!queue.isEmpty()) {

        Point current = queue.poll();

        for (int i = 0; i < 4; i++) {

            int nx = current.x + dx[i];
            int ny = current.y + dy[i];

            if (nx < 0 || nx >= n || ny < 0 || ny >= m) {
                continue;
            }

            if (map[nx][ny] == 0) {
                continue;
            }

            if (distance[nx][ny] != 0) {
                continue;
            }

            distance[nx][ny]
                    = distance[current.x][current.y] + 1;

            queue.offer(new Point(nx, ny));
        }
    }
}
```

---

## 11. BFS가 자주 사용되는 문제

코딩테스트에서 다음과 같은 표현이 나온다면 BFS를 고려할 수 있다.

| 문제 표현 | BFS 활용 |
|---|---|
| 최소 몇 번 | 최단 거리 |
| 가장 빠르게 | 최단 이동 |
| 몇 번 이동해야 하는가 | 거리 계산 |
| 미로 탈출 | 2차원 BFS |
| 연결된 영역의 개수 | 영역 탐색 |
| 섬의 개수 | 영역 탐색 |
| 특정 거리의 도시 | 거리 계산 |
| 바이러스 확산 | 확산 시뮬레이션 |
| 토마토가 익는 날짜 | 다중 시작점 BFS |

다만 **최단 거리라는 단어만 보고 무조건 BFS를 사용하면 안 된다.**

BFS로 일반적인 최단 거리를 구할 수 있는 조건은 보통

```text
모든 이동 비용이 동일한 경우
```

이다.

이동 비용이 서로 다르면 다익스트라 등의 알고리즘을 고려해야 한다.

---

## 12. 연결된 영역 개수 구하기

BFS는 최단 거리뿐만 아니라 **연결된 영역의 개수**를 구하는 문제에서도 자주 사용된다.

예를 들어
![](https://velog.velcdn.com/images/uhae451/post/28f3b345-5d99-4917-b6b4-d328488068f6/image.png)


에서 연결된 `1` 영역의 개수를 구한다고 해보자.

전체 배열을 순회한다.

```java
for (int i = 0; i < n; i++) {

    for (int j = 0; j < m; j++) {

        if (map[i][j] == 1 && !visited[i][j]) {

            bfs(i, j);
            count++;
        }
    }
}
```

핵심은

```java
새로운 방문하지 않은 영역 발견
```

시 BFS를 한 번 실행한다는 것이다.

BFS 한 번이 하나의 연결된 영역 전체를 방문하므로 BFS가 실행된 횟수가 영역의 개수가 된다.

---

## 13. 여러 시작점에서 동시에 BFS하기

BFS에서 상당히 중요한 패턴이다.

대표적으로 **백준 토마토 문제**처럼 여러 위치에서 동시에 확산되는 문제가 있다.

시작점이 하나라면

```java
queue.offer(start);
```

하나만 넣는다.

하지만 시작점이 여러 개라면 처음부터 모두 Queue에 넣는다.

```java
for (...) {

    if (map[i][j] == START) {
        visited[i][j] = true;
        queue.offer(new Point(i, j));
    }
}
```

그 이후 BFS를 한 번 실행한다.

이것을 **Multi-source BFS**라고 한다.

여러 시작점에서 각각 BFS를 따로 실행하는 것이 아니다.

```text
잘못된 방식

시작점 A BFS
시작점 B BFS
시작점 C BFS
```

처음부터 하나의 Queue에 모두 넣는다.

```java
Queue<Point> queue = new ArrayDeque<>();

queue.offer(startA);
queue.offer(startB);
queue.offer(startC);
```

그러면 모든 시작점에서 동시에 한 칸씩 퍼지는 효과를 만들 수 있다.

---

## 14. BFS 시간 복잡도

그래프를 인접 리스트로 구현했다면 BFS의 시간 복잡도는 일반적으로

```text
O(V + E)
```

이다.

- `V`: 정점의 개수
- `E`: 간선의 개수

각 정점을 한 번 방문하고 각 간선을 확인하기 때문이다.

2차원 배열이 `N × M` 크기라면 일반적으로

```text
O(N × M)
```

으로 생각할 수 있다.

각 칸을 최대 한 번 방문하기 때문이다.

---

## 15. 코딩테스트에서 자주 하는 실수

### 15.1 Queue에 넣고 방문 처리를 하지 않는 경우

```java
queue.offer(next);
```

만 하고 방문 처리를 나중에 하면 같은 노드가 Queue에 여러 번 들어갈 수 있다.

따라서 보통 다음 두 작업은 같이 한다.

```java
visited[next] = true;
queue.offer(next);
```

---

### 15.2 시작점을 방문 처리하지 않는 경우

다음처럼 Queue에만 넣는 경우가 있다.

```java
queue.offer(start);
```

시작점도 반드시 방문 처리한다.

```java
queue.offer(start);
visited[start] = true;
```

---

### 15.3 배열 범위 검사를 하지 않는 경우

2차원 BFS에서 가장 많이 발생하는 실수 중 하나다.

```java
map[nx][ny]
```

를 확인하기 전에 반드시 범위부터 검사해야 한다.

```java
if (nx < 0 || nx >= n || ny < 0 || ny >= m) {
    continue;
}
```

---

### 15.4 행과 열을 반대로 사용하는 경우

2차원 배열에서는 보통

```java
map[row][column]
```

형태다.

즉,

```java
map[x][y]
```

라고 작성했다면 코드 전체에서 `x`, `y`의 의미를 일관되게 유지하는 것이 중요하다.

---

### 15.5 BFS인데 Stack을 사용하는 경우

BFS의 핵심 자료구조는 Queue다.

```java
Queue<Integer> queue = new ArrayDeque<>();
```

Stack을 사용하면 BFS가 아니라 DFS 형태의 탐색이 된다.

---

## 16. BFS 문제를 볼 때 판단 기준

문제를 읽자마자 코드를 작성하기보다 먼저 다음을 확인하는 것이 좋다.

### 1. 탐색 대상이 무엇인가

```text
그래프의 정점
2차원 배열의 칸
상태
```

### 2. 이동 가능한 조건은 무엇인가

```text
연결된 간선
상하좌우
특정 조건을 만족하는 상태
```

### 3. 이미 방문한 상태를 어떻게 구분할 것인가

```java
boolean[] visited
boolean[][] visited
int[] distance
int[][] distance
```

### 4. 시작점은 몇 개인가

```text
하나
여러 개
```

### 5. 최단 거리가 필요한가

필요하다면 BFS의 거리 단위를 어떻게 기록할지 결정한다.

이 다섯 가지를 먼저 결정하면 대부분의 BFS 구현 구조가 자연스럽게 나온다.

---

## 17. BFS 구현 템플릿

그래프 BFS의 기본 형태는 다음 정도는 익숙하게 작성할 수 있는 것이 좋다.

```java
static void bfs(int start) {

    Queue<Integer> queue = new ArrayDeque<>();

    queue.offer(start);
    visited[start] = true;

    while (!queue.isEmpty()) {

        int current = queue.poll();

        for (int next : graph[current]) {

            if (visited[next]) {
                continue;
            }

            visited[next] = true;
            queue.offer(next);
        }
    }
}
```

2차원 BFS는 다음 형태를 기본 틀로 잡을 수 있다.

```java
static void bfs(int x, int y) {

    Queue<Point> queue = new ArrayDeque<>();

    queue.offer(new Point(x, y));
    visited[x][y] = true;

    while (!queue.isEmpty()) {

        Point current = queue.poll();

        for (int i = 0; i < 4; i++) {

            int nx = current.x + dx[i];
            int ny = current.y + dy[i];

            if (nx < 0 || nx >= n || ny < 0 || ny >= m) {
                continue;
            }

            if (visited[nx][ny]) {
                continue;
            }

            if (!canMove(nx, ny)) {
                continue;
            }

            visited[nx][ny] = true;
            queue.offer(new Point(nx, ny));
        }
    }
}
```

문제마다 달라지는 부분은 대부분 `canMove()`에 해당하는 **이동 조건**이다.

BFS 자체의 구조는 크게 달라지지 않는다.

---

## 18. BFS 문제 풀이 팁

### Queue에는 필요한 정보만 저장한다

단순 그래프 탐색이라면

```java
Queue<Integer>
```

좌표가 필요하다면

```java
Queue<Point>
```

거리까지 상태 자체에 포함해야 한다면

```java
class Node {
    int x;
    int y;
    int distance;
}
```

처럼 문제에 필요한 정보만 관리한다.

---

### visited와 distance를 따로 만들 필요가 없는 경우도 있다

최단 거리 문제에서는

```java
int[][] distance
```

를 방문 여부와 거리 기록에 동시에 사용할 수 있다.

```java
if (distance[nx][ny] != -1) {
    continue;
}
```

`-1`을 미방문 값으로 사용할 때는 탐색 전에 각 행을 `Arrays.fill(row, -1)`로 초기화해야 한다. 앞의 10절처럼 0을 미방문 값으로 사용한다면 검사 조건도 `!= 0`으로 맞춘다. 별도의 `visited` 배열이 필요 없어 코드가 단순해진다.

---

### 목적지에 처음 도착했을 때가 최단 거리다

모든 이동 비용이 동일한 BFS에서는 가까운 거리부터 탐색한다.

따라서 목적지를 처음 발견한 순간의 거리가 최단 거리다.

```java
if (nx == targetX && ny == targetY) {
    return distance[nx][ny];
}
```

모든 영역을 탐색할 필요가 없는 문제라면 이를 이용해 탐색을 조기에 종료할 수도 있다.

---

### 문제의 핵심은 BFS 코드보다 상태 정의다

BFS 구현 자체는 대부분 비슷하다.

어려운 BFS 문제는 보통 다음 부분에서 난도가 올라간다.

```text
무엇을 하나의 상태로 볼 것인가?
```

단순한 미로라면

```text
(x, y)
```

만 있으면 된다.

하지만 벽을 한 번 부술 수 있다면

```text
(x, y, 벽을 부쉈는지 여부)
```

까지 상태가 된다.

따라서 방문 배열 역시

```java
visited[x][y][broken]
```

처럼 관리해야 할 수 있다.

결국 BFS 문제에서 중요한 것은 Queue 자체보다 **동일한 상태를 어떻게 정의하고 중복 방문을 어떻게 막을 것인가**다.

---

## 19. BFS를 사용하기 좋은 경우

BFS를 우선적으로 고려하기 좋은 조건은 다음과 같다.

| 조건 | BFS |
|---|---|
| 그래프 전체 탐색 | 가능 |
| 연결된 영역 탐색 | 적합 |
| 가중치 없는 최단 거리 | 매우 적합 |
| 최소 이동 횟수 | 매우 적합 |
| 여러 지점에서 동시에 확산 | 매우 적합 |
| 서로 다른 가중치의 최단 거리 | 일반 BFS 부적합 |

특히 문제에서

```text
최소 횟수
최소 이동
몇 단계
몇 초 후
몇 칸
```

같은 표현이 나오고, 한 번의 이동 비용이 동일하다면 BFS를 먼저 떠올려볼 수 있다.

---

## 20. 정리

BFS는 **Queue를 이용하여 시작점에서 가까운 위치부터 탐색하는 알고리즘**이다.

코딩테스트에서는 단순히 BFS 코드를 외우는 것보다 다음 내용을 이해하는 것이 중요하다.

```text
1. 탐색할 상태를 정의한다.
2. Queue에 시작 상태를 넣는다.
3. 시작 상태를 방문 처리한다.
4. Queue에서 상태를 하나 꺼낸다.
5. 다음으로 이동 가능한 상태를 검사한다.
6. 방문하지 않았다면 방문 처리 후 Queue에 넣는다.
7. Queue가 빌 때까지 반복한다.
```

특히 기억해야 할 부분은 다음과 같다.

- BFS는 Queue를 사용한다.
- 방문 처리는 일반적으로 Queue에 넣을 때 한다.
- 가중치가 동일한 그래프의 최단 거리를 구할 수 있다.
- 2차원 배열에서는 방향 배열을 활용하면 구현이 단순해진다.
- 여러 시작점에서 동시에 탐색할 때는 시작점을 처음부터 모두 Queue에 넣는다.
- 어려운 BFS 문제일수록 Queue 구현보다 **상태와 방문 기준을 정의하는 것**이 중요하다.

BFS의 기본 코드는 크게 변하지 않는다.

결국 코딩테스트에서 필요한 능력은 **문제를 보고 이것을 BFS 탐색 문제로 모델링할 수 있는지 판단하는 것**이다.
