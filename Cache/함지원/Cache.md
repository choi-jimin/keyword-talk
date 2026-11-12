## Cache란?
Cache란 **데이터나 계산 결과를 저장**해 두고, **이후 같은 요청에 재사용하여 접근 시간과 처리 비용을 줄이는 저장 공간 또는 기법**이다.

영속성 컨텍스트로 예를 들어보면
```java
Member memberA = em.find(Member.class, 1L); // 첫 번째 조회 - 캐시가 없으므로 DB 조회
Member memberB = em.find(Member.class, 1L); // 두 번째 조회 - 1차 캐시가 존재하므로 return
```

| 메모리 계층 | 용량 | 접근 지연(CPU 사이클) |
| --- | --- | --- |
| Register | 약 1KB | 1-2 cycle |
| L1 Cache | 약 100KB | 2-3 cycle |
| L2 Cache | 약 500KB | 3-5 cycle |
| L3 Cache | 약 10-15MB | 30-50 cycle |
| RAM(주기억장치) | 약 10GB | 50-200 cycle |
| SSD/HDD(보조기억장치) | 약 1,000GB | 50,000 cycle |

CPU 캐시는 일반적으로 RAM보다 용량이 작고 단위 용량당 비용이 높지만, 접근 속도가 빠르다.  
애플리케이션에서도 DB 조회나 계산 결과를 캐시에 저장하여 반복 작업을 줄일 수 있다.

## Cache를 도입하면 좋은 경우
- Read 연산이 Write 연산보다 압도적으로 많은 경우
  - 자주 조회되지만 데이터 변경이 드문 경우
- 데이터 처리에 오랜 시간이 걸리는 경우
  - 복잡한 쿼리 결과, 외부 API 호출 결과, 대형 연산 데이터
- 동일한 데이터에 대한 반복 요청이 많은 경우
  - 특정 페이지나 API가 짧은 시간 동안 집중적으로 호출 될 때
 
즉, 반복적으로 데이터를 불러오는 경우에, 지속적으로 DB 또는 서버에 요청하는 것이 아니라 Memory에 데이터를 저장했다가 불러 쓰는 것이다.

## 용어 정리
### `Cache Hit`
요청한 데이터를 캐시에서 찾은 경우

### `Cache Miss`
캐시에 데이터가 없어 DB에서 조회해야 하는 경우

### `Hit Rate`
전체 요청 중 캐시에서 처리한 비율

```text
전체 요청: 1,000건
Cache Hit: 900건

Hit Rate: 90%
```
`Hit Rate`가 너무 낮다면 캐시 효과가 작고, DB 부하가 높아질 수 있다.  
다만 높다고 무조건 좋은 것은 아니다. 거의 사용하지 않는 데이터까지 캐싱하면 메모리가 낭비될 수 있다.

### `TTL`
`Time To Live`로, 캐시 데이터의 유효기간
```text
product:100 의 TTL = 10m
⟶ 10분 후 Redis에서 자동 삭제
```

### `Stale Data`
```text
DB: 15,000원
Cache: 10,000원
```
직역하면 **신선하지 않은 데이터**라고 하고, 위의 예시 처럼 원본 DB와 내용이 달라진 오래된 캐시 데이터이다.

### `Local Cache`
애플리케이션 서버 내부 메모리
- Caffeine
  - 속도는 매우 빠름.
  - 서버 간 공유는 어려움
  - 서버 재시작 하면 데이터 소멸
  - 일관성 관리 어려움

### `Remote Cache`
별도 캐시 서버
- Redis
  - 속도는 네트워크 통신 필요
  - 서버 간 공유는 가능
  - 서버 재시작 해도 별도로 유지 가능
  - 일관성 관리 상대적 쉬움

## Cache의 삭제 주기
<img width="538" height="157" alt="image" src="https://github.com/user-attachments/assets/dd8029f2-8cd4-4ce4-95d7-474f1793aae8" />

우선 브라우저마다 캐시 삭제 주기는 달라 잘 모른다.

**그렇다면 캐시 용량이 가득 찼다면?**  
캐시 용량이 가득 찼을 때 데이터가 삭제되는 순서는 **캐시 교체 알고리즘(Cache Replacement Algorithm)** 에 의해 결정된다.
- LRU(Least Recently Used) - 가장 많이 쓰임
  - 사용한지 **가장 오래된 데이터**부터 삭제
- FIFO(First In First Out)
  - 캐시에 **가장 먼저 들어온 데이터**부터 삭제
- LFU(Least Frequently Used)
  - 사용된 **횟수가 가장 적은 데이터**부터 삭제
- MRU(Mose Recently Used)
  - 사용한지 **가장 최근인 데이터**부터 삭제
- Random
  - 랜덤으로 삭제
 
## 구조는 어떻게 진화했나?
>- user server + remote cache ⟶ gateway + user server ⟶ gateway + passport
>- ⟶ gateway + local cache + sticky session ⟶ gateway + cassandra

1. user server와 redis를 합쳐서 사용했다.
  - DB 조회 비용 감소
2. gateway와 user server를 썼다.
  - 사용자 확인과 공통 처리를 모으기 위해
3. Netflix의 오픈 소스인 passport 방식을 토스도 만들어 쓰고 있다.
  - 같은 요청에서 유저 정보 조회를 반복하지 않게
4. gateway랑 local cache를 쓰면서 캐시 유실을 막기 위해 sticky session을 사용하고 있다.
  - local cache의 단점을 보완하기 위해 sticky session을 사용
5. NoSQL인 cassandra랑 gateway를 쓰고 있다.
  - 특정 gateway에 묶이지 않고 데이터를 공유하기 위해

## Remote-Local 정합성은?
```
외부 저장소: 김〇〇
Gateway 1 로컬 캐시: 김〇〇
Gateway 2 로컬 캐시: 김〇〇

외부 저장소: 이〇〇  ⟵ 최신 데이터
Gateway 1: 김〇〇  ⟵ 오래된 데이터(stale)
Gateway 2: 김〇〇  ⟵ 오래된 데이터(stale)
```
이렇게 **서로 다른 값을 갖는 것을 Remote-Local 정합성**이라고 한다.

### pub/sub
pub/sub은 변경 사실을 서버들에 알려주는 방식이다.
1. 사용자 정보를 변경
2. “사용자 A의 정보가 바뀌었다”라는 이벤트를 발행
3. 이벤트를 구독하는 Gateway들이 A의 로컬 캐시를 삭제
4. 다음 요청에서 최신 정보를 가져와 다시 캐시

### stream
stream은 변경 이벤트를 기록해 순서대로 읽게 하는 방식이다.  
일반적인 pub/sub은 서버가 잠시 연결을 끊으면 알림을 놓칠 수 있지만, 이벤트를 보관하는 stream은 보관 기간과 소비 위치 관리에 따라 놓친 이벤트를 나중에 처리할 수 있다.

## 메모리가 크다면?
>- bitmap, hotkey 등 자료구조 성택, Binary 포맷

### Bitmap
`Bitmap`은 각 항목의 상태를 `0`과 `1`로 저장하는 자료구조이다.

예를 들어보면,
|ID|출석날짜|출석시각|
|---|---|---|
|1|2026-11-11|09:00|
|2|2026-11-11|09:00|
|3|2026-11-11|09:00|
|4|2026-11-11|09:00|
|5|2026-11-11|09:02|
|6|2026-11-11|09:04|
|7|2026-11-11|09:04|

DB에 `attendance`테이블이 있고, 서비스에서 "이 유저 오늘 출석 했어?"를 자주 확인한다면 redis와 같은 cache에 저장할 수 있다.

```text
  key:  attendance:2026-11-11:user:1
value:  1

  key:  attendance:2026-11-11:user:3
value:  1

  key:  attendance:2026-11-11:user:4
value:  1

  key:  attendance:2026-11-11:user:7
value:  1
```

이 방식이 메모리가 너무 커서 Bitmap을 사용한다고 했을 때,
```text
Redis Key:  attendance:2026-11-11

비트위치:  0  1  2  3  4  5  6  7
저장비트:  0  1  0  1  1  0  0  1

최종
  key: attendance:2026-11-11
value: [0][1][0][1][1][0][0][1]
```

**그럼 갖고올 때는 어떻게?**

```redis
GETBIT attendance:2026-11-11 3
```
```text
return: 1 ⟶ 출석
```
키와 비트위치를 전달하면 Redis가 해당 비트를 읽어서 반환을 한다.

### HotKey
유독 요청이 많이 몰리는 Redis 키를 말한다.

### Binary Format
```json
{
  "userId":3,
  "attended":true
}
```
라는 JSON이 있을 때, 정해진 규칙에 따라 바이트로 표현하는 방식이다. 목적은 저장 공간과 전송량을 줄일 때 사용한다.

앞에 있는 JSON 형식의 "같은 구조의 데이터가 대량으로 오가는가?"라는 방향이 있을 때 사용하면 좋다.
