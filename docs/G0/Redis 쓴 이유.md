> G0에서 Redis는 “캐시”가 아니라, **실시간 경로(핫패스)를 DB/저장소로부터 분리하고, 이후 scale-out에서도 같은 구조를 유지하기 위한 공통 인메모리 계층**이다.
> 

---

# 1) G0에서 Redis가 담당하는 것

G0 요구사항을 기능으로 쪼개면 Redis 역할이 3개로 딱 떨어진다.

## A. 실시간 메시지 fan-out (Redis Pub/Sub)

- 목적: **메시지 전파의 허브를 “서버 밖”으로 빼기**
- G0에서는 단일 서버라도, 메시지 흐름을 **미리 “멀티 노드 기준”으로 고정**해둠

## B. Presence / Viewer count (SET, INCR)

- 목적: **초고빈도 상태값을 메모리에서 원자적으로 처리**
- viewer count 같은 값은 DB에 `UPDATE viewer=viewer+1` 하면 **hot row 락 경합**이 바로 생김
    
    → 그래서 Redis INCR로 “락 경합 회피 + 지연 최소화”
    

## C. (옵션) 실시간 지표 집계의 원천 (stream_metrics의 재료)

- 목적: 10초마다 집계할 때 **DB를 쿼리로 긁지 말고** Redis에서 빠르게 읽어오기
- “집계는 주기 작업”이니까, Redis가 **가벼운 실시간 상태 저장소**가 됨

---

# 2) “scale=1인데도” Redis Pub/Sub을 붙이는 이유

## 이유 1 — 메시지 흐름을 “멀티 노드 기준”으로 고정해두기

G0에서 simple broker로 만들면 흐름이 이렇게 됨:

```
SEND → (서버 메모리 브로드캐스트) → topic
```

G0.5에서 Redis로 바꾸면 흐름이 이렇게 바뀜:

```
SEND → publish → subscribe → broadcast → topic
```

즉, **메시지 경로가 통째로 바뀜**.

이 상태에서 버그가 생기면:

- “구독 문제인지”
- “브로드캐스트 문제인지”
- “Redis 문제인지”
- “listener 문제인지”

원인이 섞여서 디버깅이 어려워진다.

✅ 그래서 G0에서부터 Redis fan-out을 붙이면

**G0 → G0.5에서 코드 흐름이 안 바뀌고, 인스턴스 수만 늘리면 됨**.

> 결론: 지금 붙이는 건 성능 때문이 아니라 **변경 비용/리스크를 없애기 위한 설계 고정**이다.
> 

---

## 이유 2 — “핫패스/콜드패스 분리”를 설계 원칙으로 유지

G0 목표:

- 핫패스: **브로드캐스트/실시간**
- 콜드패스: **버킷 버퍼 → Mongo bulk 저장**

이 분리를 지키려면,

핫패스는 “외부 저장소 지연”에 영향을 받으면 안 됨.

Redis Pub/Sub을 쓰면:

- 실시간 전달은 Redis 중심으로 “가볍게”
- 저장은 별도의 콜드패스에서 “비동기/배치”

즉, Redis는 **핫패스를 유지하는 경계선**이 된다.

---

## 이유 3 — “분산 시스템 기본기”를 설명할 수 있음

scale=1이어도, Redis Pub/Sub을 넣으면 바로 말할 수 있는 포인트가 생김:

- “서버 내부 브로커 대신 외부 메시지 허브를 사용”
- “노드가 여러 개여도 동일한 fan-out 구조 유지”
- “단, Pub/Sub은 유실 가능해서 유실 허용 영역에만 사용”

---

# 3) G0에서 Redis를 안 쓰면 어떤 문제가 생기나?

## ① Presence/viewer를 DB로 하면

```
UPDATE broadcastSET viewer= viewer+1
```

동접이 늘면:

- 같은 row를 계속 갱신 → **hot row**
- row lock 경합 → TPS 하락, p95 상승

✅ Redis INCR은:

- 메모리 원자 연산
- 락 경합 회피
- 지연 최소화

---

## ② fan-out을 simple broker로 하면

G0에서는 되지만, G0.5부터 구조가 깨짐:

- 서버 A 접속자만 수신
- 서버 B 접속자는 수신 불가

✅ Redis Pub/Sub은:

- 어느 서버에서 publish해도
- 모든 서버가 subscribe로 받아
- 각자 연결된 클라이언트에게 broadcast

---

# 4) Redis 한계 

Redis Pub/Sub은 큐가 아니다.

```
메시지 유실 가능
재전송 없음
영속 저장 없음
ack 없음
```

그래서 원칙:

```
✅ 채팅/알림/presence/카운터: 사용 가능 (유실 허용)
❌ 주문/결제/재고/쿠폰 확정: 절대 사용 금지
```

> Redis Pub/Sub은 유실 가능성이 있어 채팅처럼 유실 허용 영역에만 적용했고, 정합성이 필요한 결제/주문은 DB 트랜잭션과 멱등 처리로 분리했습니다.
> 

---

# 5) G0에서 Redis 적용 위치 요약

## fan-out

- publish: `@MessageMapping`에서 `convertAndSend()`
- subscribe: `RedisMessageListenerContainer` → `SimpMessagingTemplate.broadcast`

## presence / viewer

- `room:{id}:users` → `SET`
- `room:{id}:viewer` → `INCR/DECR`
- (선택) heartbeat TTL로 유령 세션 정리

---

> G0에서 Redis는 성능 최적화 목적이 아니라, 실시간 경로를 저장소로부터 분리하고(핫/콜드 분리), fan-out과 presence 같은 실시간 상태를 인메모리 원자 연산으로 처리하기 위한 공통 계층으로 도입했습니다.
> 
> 
> 또한 이후 G0.5 scale-out에서도 메시지 전파 구조를 바꾸지 않도록, 단일 인스턴스 단계부터 Redis Pub/Sub 기반 fan-out 흐름을 고정했습니다.
> 
> 단, Redis Pub/Sub은 유실 가능성이 있어 채팅/알림 등 유실 허용 영역에만 사용합니다.
>
