---
title: "Redis 실무 정리: 캐시를 넘어 세션·락·큐·레이트리밋까지 (Spring 기준)"
date: 2026-01-16
categories: [Architecture]
tags: [redis, cache, spring, performance, consistency]
layout: single
toc: false
classes: wide
---

Redis는 “빠른 캐시”로 시작하지만, 운영에서는 **데이터 정합성**, **동시성 제어**, **트래픽 방어**, **장애 대응**이 핵심이다.  
이 글은 단순 문법이 아니라 **실무에서 Redis를 어디에, 어떤 방식으로, 어떤 함정까지 고려해서 쓰는지**를 기준으로 정리한다.

---

## 0. Redis를 한 문장으로
Redis는 **인메모리 기반의 Key-Value 데이터 구조 서버**이며, 캐시뿐 아니라 **세션/락/레이트리밋/이벤트 스트림/간단한 큐** 같은 “고속 상태 저장소” 역할을 한다.

다만 운영에서 중요한 전제는:

- Redis는 기본적으로 **메모리 기반**이다 (즉, 메모리/eviction/영속성 정책이 설계의 일부)
- 네트워크 I/O가 있다 (즉, “무조건 빠르다”가 아니라 **연결/파이프라인/커맨드 수**가 성능을 좌우)
- **정합성 요구가 높은 데이터의 단독 저장소**로 쓰면 위험해질 수 있다 (RDB/AOF/replica/cluster의 특성 이해 필요)

---

## 1. Redis 아키텍처 핵심 개념

### 1.1 싱글 스레드? 멀티 스레드?
Redis는 커맨드 실행은 기본적으로 단일 스레드(이벤트 루프) 모델로 유명하다.  
이 설계 덕분에 락 경합 없이 **원자적(atomic) 처리**가 강점이지만, 아래가 중요하다.

- 단일 인스턴스에서 “한 번에 한 커맨드”이므로
  - 커맨드가 느리면 전체가 느려짐 (예: 큰 키 스캔, 큰 리스트/해시 조작)
- 네트워크 I/O는 최근 버전에서 멀티 스레드 옵션이 있으나, 실무적으로는 **커맨드 패턴 최적화**가 먼저다.

**운영 교훈:**  
Redis 성능은 “TPS”보다 **커맨드당 작업량**이 결정한다.

---

## 2. 데이터 타입과 실무 패턴

Redis가 강한 이유는 단순 key-value가 아니라 **자료구조**가 있기 때문이다.

### 2.1 String
가장 기본. 캐시(HTML, JSON), 토큰, 플래그 등에 사용.

- `SET key value EX 60` (TTL 60초)
- `SETNX` (락/중복 처리에 활용)
- `INCR/DECR` (카운터, 레이트리밋)

**패턴:** “idempotency key” 저장  
결제/주문 API에서 동일 요청 중복 방지:

- 요청 해시를 키로 `SET key requestId NX EX 300`
- NX 성공이면 처리 진행, 실패면 중복 요청으로 판단

---

### 2.2 Hash
객체 단위 캐시에 자주 쓰인다.

- `HSET user:123 name "wanhi" age 30`
- 장점: 일부 필드만 갱신 가능
- 단점: 필드 수가 너무 많거나 value가 커지면 메모리 부담

**패턴:** 사용자 세션 / 프로필 캐시  
`user:{id}` 에 주요 필드만 저장 + TTL

---

### 2.3 List
간단한 큐로 활용 가능.

- `LPUSH queue item`
- `RPOP queue`
- `BRPOP queue 5` (blocking pop)

**주의:** List 기반 큐는 “간단한” 용도에 적합.  
**정확히 한 번(Exactly-once)** 같은 보장이 필요하면 Redis Streams/Kafka 등 고려.

---

### 2.4 Set
중복 제거 + membership 체크.

- `SADD coupon:issued userId`
- `SISMEMBER coupon:issued userId`

**패턴:** 쿠폰/이벤트 중복 참여 방지  
단, “최종 정합성”은 DB로 귀결시키는 게 안전.

---

### 2.5 Sorted Set (ZSET)
점수(score) 기반 정렬이 핵심.

- 랭킹, 우선순위 큐, 지연 큐(딜레이 작업)에 유용
- `ZADD rank 100 userA`
- `ZRANGE rank 0 9 WITHSCORES`

**패턴:** 지연 큐(Delay Queue) 흉내
- score에 “실행 시각(epoch ms)”를 넣고
- 주기적으로 `ZRANGEBYSCORE`로 꺼내 처리

---

### 2.6 Stream
Redis가 제공하는 로그/스트림 구조(카프카 유사 일부 기능).

- 컨슈머 그룹 지원
- 재처리/ACK 등 큐보다 “조금 더” 신뢰성

**패턴:** 주문 이벤트(“주문 생성됨”, “결제 승인됨”) 같은 이벤트 버스 흉내  
다만 대규모/장기 운영이면 Kafka 같은 전문 도구가 관리 편하다.

---

## 3. TTL과 캐시 설계의 핵심

Redis를 캐시로 쓸 때 가장 중요한 건 **TTL과 무효화 전략**이다.

### 3.1 TTL 기본
- TTL이 없는 캐시는 운영에서 “언젠가 터지는 부채”
- TTL은 시스템 특성(변경 주기/허용 stale)에 맞춘다.

예:
- 상품 상세: 1~10분 (변경 빈도 낮으면 더 길게)
- 재고/가격: 짧게(수초~수십초) 또는 **캐시보다 이벤트 기반 무효화**
- 세션: 로그인 정책에 따라 30분~2시간 등

---

### 3.2 Cache Stampede (캐시 폭주) / Dogpile 문제
동일 키가 동시에 만료되면 DB로 트래픽이 몰린다.

**해결 패턴**
1) TTL에 jitter(랜덤) 추가  
- 60초 TTL이라면 50~70초로 랜덤 분산

2) 단일 비행(SingleFlight) / 뮤텍스 키  
- 캐시 미스 시 “한 요청만 DB 조회”
- 나머지는 잠깐 대기하거나 stale 반환

3) Stale-While-Revalidate (SWR)
- 만료 직전/만료 후 짧은 기간 stale을 제공하고 백그라운드 갱신

---

### 3.3 Write-Through / Write-Behind / Cache Aside
실무에서 많이 쓰는 3가지 패턴:

- Cache Aside(가장 흔함)
  - 읽기: 캐시 조회 → 미스면 DB 조회 후 캐시 put
  - 쓰기: DB 업데이트 후 캐시 삭제/갱신
  - 장점: 단순
  - 단점: 갱신 타이밍 실수로 stale 발생

- Write-Through
  - 쓰기 시 캐시와 DB를 동시에 갱신
  - 장점: 읽기 일관성 좋음
  - 단점: 쓰기 비용 증가

- Write-Behind(비동기)
  - 캐시에 쓰고 DB는 나중에 반영
  - 장점: 쓰기 TPS 최고
  - 단점: 정합성/장애 복구가 어려워 커머스/결제엔 보통 비추천

**커머스/결제 결론:**  
정합성 민감한 도메인은 “캐시가 정답 저장소”가 되면 위험하다.  
캐시는 **조회 가속** 역할로 한정하는 게 대부분 안전하다.

---

## 4. 동시성 제어: 분산 락(Distributed Lock)

### 4.1 가장 기본: SET NX EX (분산 락의 출발점)

Redis에서 분산 락을 구현할 때 가장 기본이 되는 패턴은 다음 한 줄이다.

```bash
SET lock:order:123 token NX EX 5
```

* **NX**: key가 없을 때만 생성 (이미 있으면 실패)
* **EX**: TTL 지정 → 데드락 방지
* **token**: 락 소유자 식별용 값 (UUID 등)

여기서 중요한 건 **락 해제 방식**이다.
단순히 `DEL key`로 해제하면 다른 인스턴스가 만든 락까지 지워버릴 수 있다.

그래서 보통 Lua 스크립트를 사용해 **원자적으로** 해제한다.

```lua
-- unlock.lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

운영에서 이 token 비교를 빼먹으면
락 경쟁 상황에서 아주 애매한 버그가 발생한다.

---

### 4.2 실무에서 락이 필요한 대표 상황

커머스/결제 시스템 기준으로 Redis 락이 자주 등장하는 케이스는 다음과 같다.

* 쿠폰 선착순 발급 (동시 요청 폭주)
* 동일 주문번호 중복 결제 방지
* 특정 배치 작업 “한 번만” 실행 보장
* 재고 감소 로직 동시성 완화

다만 여기서 중요한 포인트는:

> Redis 락은 DB 트랜잭션을 대체하지 않는다.

락은 **경쟁을 줄여주는 장치**일 뿐이고,
최종 정합성은 DB에서 보장해야 한다.

예:

* `UPDATE stock SET qty = qty - 1 WHERE qty > 0`
* 영향 row 수 체크

이 패턴이 Redis 락만 쓰는 것보다 안전한 경우도 많다.

---

## 5. 영속성: RDB vs AOF (운영에서 반드시 알아야 하는 차이)

Redis를 단순 캐시가 아니라
세션/큐/락 같은 상태 저장소로 쓰기 시작하면
영속성 전략을 이해해야 한다.

### 5.1 RDB (Snapshot)

특정 시점 메모리를 통째로 저장한다.

장점:

* 복구 속도 빠름
* 파일 용량 작음

단점:

* 마지막 스냅샷 이후 데이터 유실 가능

캐시 전용이면 보통 RDB만 쓰거나 최소 설정으로 둔다.

---

### 5.2 AOF (Append Only File)

모든 write 명령을 로그로 남긴다.

장점:

* 데이터 유실 최소화
* 복구 시점 제어 가능

단점:

* 파일 크기 증가
* rewrite 관리 필요

결제/세션 같은 상태성 데이터가 있다면
AOF 고려가 필요하지만 운영 복잡도는 증가한다.

---

## 6. 고가용성: Replica / Sentinel / Cluster

### 6.1 Replica (복제)

* Master → Replica 복제
* 읽기 분산 가능
* Replication lag 존재 (즉시 일관성 아님)

---

### 6.2 Sentinel

* 장애 감지
* 자동 failover
* 애플리케이션은 sentinel 통해 master 조회

운영 난이도 대비 효과가 좋아
단일 인스턴스 다음 단계로 많이 사용한다.

---

### 6.3 Cluster (샤딩)

데이터를 슬롯 단위로 여러 노드에 분산.

주의점:

* 멀티키 연산 제한
* 키 해시태그 사용 가능

예:

```text
{user}:profile
{user}:orders
```

같은 슬롯으로 묶을 수 있다.

---

## 7. 메모리와 Eviction 전략

Redis 장애의 상당수는 메모리 정책에서 시작된다.

### 7.1 maxmemory

물리 메모리 100%를 쓰지 않는다.

* OS 버퍼
* 연결 버퍼
* Redis 오버헤드

까지 고려해야 한다.

---

### 7.2 eviction 정책

대표 옵션:

* noeviction → 메모리 꽉 차면 write 실패
* allkeys-lru → 전체 key 중 LRU 제거
* volatile-lru → TTL 있는 key만 제거

세션/락처럼 사라지면 안 되는 데이터가 있다면
eviction 정책을 신중하게 선택해야 한다.

---

## 8. 성능 최적화: 커맨드 수 줄이기

Redis는 네트워크 왕복 비용이 있다.

### 8.1 Pipeline

여러 명령을 한 번에 전송해 latency 감소.

### 8.2 MGET / HMGET

가능하면 batch 조회로 합친다.

### 8.3 Big Key 금지

value가 크거나
hash/set 원소가 너무 많으면
Redis 전체 지연이 발생할 수 있다.

---

## 9. Spring에서 Redis 사용할 때 실무 포인트

### 9.1 Lettuce vs Jedis

Spring Boot에서는 Lettuce가 기본.

* 논블로킹
* connection pooling 필요 없음(대부분 상황에서)

하지만 timeout 설정은 반드시 점검해야 한다.

---

### 9.2 직렬화 전략

운영에서 자주 겪는 문제:

* JDK Serialization → 디버깅 어려움
* JSON → 용량 증가

실무에서는:

* String / Hash 구조로 단순화
* JSON + 압축(필요 시)

패턴을 많이 사용한다.

---

### 9.3 @Cacheable 사용 시 주의

편하지만 숨겨진 비용이 있다.

* TTL 제어 어려움
* stampede 대응 없음
* key 설계 통제 어려움

커머스/결제 핵심 로직에서는
cache abstraction 남용을 피하는 편이 안전하다.

---

## 10. Redis를 어디까지 믿을 것인가

### 잘하는 영역

* 조회 성능 개선
* 레이트리밋
* 분산 락
* 세션/토큰
* 간단한 큐

### 위험해지는 영역

* 정답 데이터를 Redis에만 저장
* eviction/TTL 전략 없이 운영
* 장애 fallback 없는 구조

Redis는 DB를 대체하는 게 아니라
**DB를 보호하는 레이어**에 가깝다.

---

## 11. 운영 체크리스트

* 모든 캐시 키에 TTL이 있는가?
* 캐시 만료 시 DB 폭주 대비했는가?
* key 네이밍 규칙이 있는가?
* big key 모니터링 하고 있는가?
* 락 해제는 Lua 기반인가?
* maxmemory 정책이 의도대로인가?

---

## 마무리

Redis는 단순히 빠른 캐시가 아니라
운영 환경에서 **트래픽과 동시성을 제어하는 도구**다.

특히 주문·결제 시스템에서는
트랜잭션 설계(DB)와 함께 Redis 전략을 같이 설계해야
안정성과 성능을 동시에 얻을 수 있다.

