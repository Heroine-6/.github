# 🏠 부동부동 (BudongBudong)

> **경매 기반 부동산 직거래 플랫폼**  
> "중개 수수료 없이, 시장이 가격을 결정합니다"

<img width="2050" height="1387" alt="부동부동_메인페이지" src="https://github.com/user-attachments/assets/3d378b94-eb81-4d62-bdbc-efdedbad7a7a" />

---

## 📚 목차

- [✨ Overview](#-overview)
- [🕹️ 핵심 기능](#️-핵심-기능)
- [🏗️ 시스템 아키텍처](#️-시스템-아키텍처)
- [🛠️ 기술 스택](#️-기술-스택)
- [💡 기술적 의사결정](#-기술적-의사결정)
- [⚡ 트러블 슈팅](#-트러블-슈팅)
- [🎯 성능 개선](#-성능-개선)
- [📁 설계 문서](#-설계-문서)
- [🥇 팀원 소개](#-팀원-소개)

---

## ✨ Overview

### 서비스 소개

**부동부동**은 부동산 직거래 시 발생하는 높은 중개 수수료와 불투명한 가격 협상 구조를 개선하기 위해 기획된 **경매 기반 부동산 직거래 플랫폼**입니다.

사용자는 매물을 직접 등록하고, 구매자는 공개된 조건에서 입찰에 참여하며, 경매 결과를 통해 시장 참여 기반의 가격이 형성됩니다.

### 기획 배경

**기존 부동산 거래의 문제**

- 중개사를 통한 높은 수수료 부담
- 불투명한 가격 협상 구조로 인한 정보 비대칭
- 실거래가 기반의 합리적인 가격 비교 수단 부재

> 부동부동은 `실시간 경매 + 실거래가 비교 + 직거래`를 결합해 `부동산 거래의 투명성과 합리성`을 높입니다.

### 사용자 이용 흐름

<img width="4000" height="2250" alt="image" src="https://github.com/user-attachments/assets/51c32d2d-a1ad-495b-ae93-85fbd075650a" />

---

## 🕹️ 핵심 기능

### 🏢 스마트 매물 등록

- 법정동 주소 입력 시 실거래 데이터 기반으로 매물명, 주소, 가격, 전용면적, 건축연도 **자동 입력**
- 공공 데이터 포털 API를 활용한 실거래가 데이터 연동

### 🔎 다중 조건 매물 검색

- 가격, 건물명, 주소, 매물 타입, 입주 가능일 등 **다양한 조건 조합 검색**
- Elasticsearch 기반 고속 검색으로 밀리초 단위 응답
- 불필요한 결과를 줄이는 정밀 필터링

### ⏰ 실시간 경매 시스템

- 입찰 금액 및 최고가 **실시간 반영**
- 경쟁 상황을 즉시 확인 가능한 입찰 화면
- 경매 종료 시점 기준으로 엄격한 입찰 처리
- **RabbitMQ 기반 대용량 트래픽 처리**

### 📈 주변 시세 비교

- 입찰 시 매물 반경 **1km 내 실거래가 정보** 제공
- 손해 보지 않는 합리적인 입찰 가이드 제공

### 📩 실시간 채팅

- **WebSocket 기반** 실시간 채팅
- 읽음 / 안 읽음 상태 관리 및 안 읽은 메시지 수 표시
- 경매 종료 후에도 이전 문의 내역 조회 가능

### 💸 안전한 결제 처리

- 경매 상태 흐름과 강하게 결합된 거래 안전 장치
- 결제 요청과 승인 분리 구조로 외부 장애·중복 호출 대응
- **toss payments** 연동

### 🔔 카카오톡 알림

- 주요 이벤트 발생 즉시 카카오톡 알림 메시지 전송
    - 경매 시작 / 종료 임박 / 종료
    - 입찰 발생 및 최고가 갱신
    - 낙찰 이후 결제 기한 안내
    - 결제 완료

---

## 🏗️ 시스템 아키텍처

### Architecture Diagram

<img width="1858" height="1640" alt="v2_시스템아키텍쳐 drawio" src="https://github.com/user-attachments/assets/eb6a5d1f-5028-4a68-9676-a28e9fc25ee5" />

<details>
<summary><strong>Infrastructure</strong></summary>

| Component             | Technology              | Purpose            |
|-----------------------|-------------------------|--------------------|
| **Cloud Platform**    | AWS                     | 전체 인프라 호스팅         |
| **Container Runtime** | EC2                     | 서비스 실행 환경          |
| **Database**          | RDS (MySQL)             | 트랜잭션 데이터 저장        |
| **Object Storage**    | S3                      | 파일 저장              |
| **Cache**             | Redis                   | 캐싱 및 분산 락          |
| **Search Engine**     | Elasticsearch           | 고속 매물 검색           |
| **Message Queue**     | RabbitMQ                | 비동기 메시징 (경매 입찰 처리) |
| **Monitoring**        | Grafana, K6, nGrinder   | 성능 모니터링 및 부하 테스트   |
| **CI/CD**             | GitHub Actions + Docker | 빌드, 테스트, 배포 자동화    |

</details>

---

## 💡 기술적 의사결정

<details>
<summary><strong>MySQL ↔ Elasticsearch 동기화 전략</strong></summary>

### 배경

매물 등록/수정/삭제 시 MySQL과 Elasticsearch 간 데이터 정합성 유지 방식 선택 필요.

### 고려한 방법

| 방식                         | 특징                            | 선택 여부        |
|----------------------------|-------------------------------|--------------|
| **CDC (Debezium + Kafka)** | binlog 기반 자동 감지, 높은 신뢰성       | ❌ 인프라 복잡도 과도 |
| **주기적 배치 동기화**             | 구현 단순, 초기 적재에 적합              | ❌ 실시간 반영 불가  |
| **도메인 이벤트 기반**             | 비즈니스 이벤트 중심, 트랜잭션 커밋 후 비동기 처리 | ✅ 최종 선택      |

### 최종 선택 — 도메인 이벤트 기반 동기화

```
매물 변경 → 도메인 이벤트 발행 → 트랜잭션 커밋 후 비동기 처리 → Elasticsearch 반영
```

**선택 이유**

- Kafka 등 대규모 인프라 없이 Spring 이벤트 리스너만으로 통합 가능
- "무슨 데이터가 바뀌었는가"가 아닌 **"어떤 비즈니스 이벤트가 발생했는가"** 중심 설계
- 실시간성 확보 + 현재 프로젝트 규모에 적합한 복잡도

**Trade-off**

- 이벤트 처리 실패 시 재처리 전략 필요 (Retry / DLQ 설계)

---

</details>

<details>
<summary><strong>Refresh Token 전략</strong></summary>

### 배경

Access Token(JWT)의 짧은 만료 시간으로 인한 사용자 편의성 저하와, 긴 만료 시간으로 인한 **보안 취약** 사이의 균형을 맞추기 위해 Refresh Token 도입

### 인증 흐름

```
로그인 → AT + RT 발급 (RT는 Redis + HttpOnly 쿠키에 저장)

API 요청
  ├─ AT 유효 → 접근 허가
  └─ AT 만료 → RT 검증
          ├─ RT 유효 → AT 재발급 후 접근 허가
          └─ RT 무효 → 재로그인
```

### 전략 결정: RTR(Refresh Token Rotation) 채택

| 항목       | 선택            | 이유                         |
|----------|---------------|----------------------------|
| AT 만료 시간 | 60분           | 탈취 피해 최소화                  |
| RT 만료 시간 | 2주            | 사용자 편의성                    |
| 클라이언트 저장 | HttpOnly 쿠키   | JS 접근 불가 → XSS 방어          |
| 서버 저장    | Redis         | TTL 자동 관리, 빠른 조회           |
| RT 수명 정책 | **RTR (일회성)** | AT 재발급 시 RT도 함께 갱신 → 탈취 대응 |

- RT가 탈취되더라도 먼저 사용된 이후의 토큰은 자동으로 무효화되도록 설계

</details>

<details>
<summary><strong>위치 기반 검색 엔진 선정: Redis Geo vs Elasticsearch Geo</strong></summary>

### 배경

행정동 단위 검색의 한계를 극복하기 위해 좌표 기반 반경 검색 도입 필요.
기존 인프라에 Redis와 Elasticsearch가 모두 구축되어 있기에 엔진 선정 필요.

### 결정: Elasticsearch 채택

- Redis Geo는 좌표 조회 후 면적·타입 필터를 위해 DB를 재조회해야 하는 2중 오버헤드 발생.
- ES의 BKD-Tree 구조는 반경 + 다중 조건을 단일 쿼리로 처리 가능.
- 부동산 데이터는 정적 특성이 강해 인덱싱 비용보다 복합 필터링 성능이 더 중요.

### 추가 고도화: Redis Look-aside Cache 도입

인기 지역·특정 조건의 반복 검색 시 ES의 CPU 점유율이 상승하고 중복 연산이 발생하는 문제를 캐싱으로 해결

```
검색 요청 → Redis 캐시 확인 (Query DSL Hash 키)
  ├─ HIT  → 캐시 반환 (~10ms)
  └─ MISS → ES 조회 → Redis 저장 → 반환 (~200ms)
```

- TTL 120초 적용으로 데이터 최신성과 성능 간 균형 확보.
- 동일 조건 반복 검색 시 응답시간 **90% 이상 단축** (200ms → 10ms 미만).

</details>

<details>
<summary><strong>분산 락을 통한 동시성 제어 </strong></summary>

### 배경

입찰 기능은 마감 임박 구간에 동일 자원에 대한 동시 요청이 집중될 가능성이 있다.
서버 확장을 고려하고 있어 애플리케이션 레벨의 분산 락이 필요했다.

### 고려한 방법

**Lettuce 기반 Spin Lock**

락 획득 실패 시 일정 시간 대기 후 Redis를 반복 호출하는 구조.

```
tryLock 실패 → 일정 시간 대기 → Redis 재호출 → 반복
```

- 여러 스레드가 동시에 반복 요청 → Redis CPU 사용량 증가.
- 락 해제 시점과 무관하게 정해진 주기로 재시도 → 불필요한 네트워크 비용 발생.
- 입찰 특성상 동일 키에 경합이 집중되는 구조에서 부하 폭증 위험.

**Redisson 기반 Subscribe Lock**

락 획득 실패 시 Redis를 반복 호출하지 않고, 락 해제 이벤트를 구독해 재시도하는 구조.

```
tryLock 실패 → 대기 상태 진입 → 락 해제 알림 수신 → 재시도
```

- 이벤트 기반 재시도로 네트워크 호출 횟수 최소화.
- Watchdog을 통한 자동 락 연장, 재진입 락, 타임아웃 관리 내장.

### 결정: Redisson 채택

현재 구간별 `waitTime` 전략을 적용 중이다.

| 구간       | waitTime       |
|----------|----------------|
| 일반 구간    | 0ms            |
| 마감 임박 구간 | 200ms ~ 1000ms |

Spin Lock 구조에서는 대기 중에도 Redis에 반복 요청이 발생하지만, Redisson은 실제 대기 상태로 전환되므로 **waitTime이 길수록 Redisson의 부하 이점이 커진다.** 입찰처럼 경합이
특정 키에 집중되는 구조에서는 반복 polling이 없는 Redisson이 구조적으로 더 적합하다고 판단했다.

</details>

---

## ⚡ 트러블 슈팅

<details>
<summary><strong>결제 무한 재시도 이슈</strong></summary>

### 문제

Toss API 장애 지속 시 동일 결제 메시지가 무한 반복 consume. 100ms 동안 59회(초당 약 590회) 재시도 발생.

### 원인

Spring AMQP 기본값 `requeue = true` + DLX/DLQ 미설정으로 재시도 횟수 제한·간격·최종 실패 처리가 모두 없는 상태.

### 해결

예외 throw → requeue 방식 대신, 실패 시 **지연 큐(Delay Queue)에 명시적 재발행** + 최대 5회 제한.

[자세한 내용 보기](https://www.notion.so/2fe2dc3ef51480c3b56eee84050b407b)

</details>

<details>
<summary><strong>분산 락과 트랜잭션 분리 시 @Transactional 미적용 이슈</strong></summary>

### 문제

같은 클래스 내 메서드로 분리했을 때 `@Transactional`이 적용되지 않음.

### 원인

`@Transactional`은 AOP 프록시 기반이라 **같은 빈 내부 호출은 프록시를 우회**해 트랜잭션이 걸리지 않음.

### 해결

클래스 자체를 분리해 외부 호출 구조로 변경.

- `BidService` → 분산 락 획득/해제
- `BidTxService` → 입찰 생성 트랜잭션

[자세한 내용 보기](https://www.notion.so/Transactional-2f02dc3ef5148042a633c6a71d1f34e4)

</details>

<details>
<summary><strong>핵심 기능에서 알림 로직 분리</strong></summary>

### 문제

알림 로직이 트랜잭션 내부에 존재해 카카오 API 장애 시 **알림 실패 → 핵심 기능 롤백**으로 이어지는 구조.

### 해결

RabbitMQ 기반 비동기 처리로 책임 분리. 핵심 기능은 메시지 발행 후 즉시 응답, 알림은 별도 Consumer가 처리.

- 카카오 API 장애가 핵심 기능에 영향 없음
- 트랜잭션 내 외부 네트워크 호출 제거
- Consumer 추가만으로 알림 채널 확장 가능

[자세한 내용 보기](https://www.notion.so/3072dc3ef51480c988c8d066ff2a6d1d)

</details>

<details>
<summary><strong>실거래가 데이터 중복 적재 및 트랜잭션 롤백</strong></summary>

### 문제

배치 실행 시마다 데이터 기하급수적 증가(5만 → 20만 건), 최종적으로 `saveDeals()` 트랜잭션 롤백으로 후속 작업 전체 마비.

### 원인

- 파티셔닝 도입 과정에서 유니크 제약 조건 스크립트 누락 → 중복 INSERT 허용
- Job Parameter로 `currentTimeMillis()` 사용 → 실행마다 전체 재수집 반복
- 수집 단계 중복 제거 로직 부재, 수십만 건 단일 트랜잭션 처리

### 해결

- 유니크 제약 조건 복구 + `INSERT IGNORE + batchUpdate`로 중복 Skip 처리
- Job Parameter를 `runDate`로 표준화
- 500건 단위 Bulk Write로 전환
- 지오코딩 실패 전용 `geocodeRetryJob` 신설

[자세한 내용 보기](https://www.notion.so/2ff2dc3ef51480549299e68cda920df2)

</details>

---

## 🎯 성능 개선

<details>
<summary><strong>매물 검색 성능 개선 (Elasticsearch 도입)</strong></summary>

| 지표                | DB 검색 (MySQL) | Elasticsearch | 개선율      |
|-------------------|---------------|---------------|----------|
| filter 조건         | 150~300ms     | 80 ~ 150ms    | 약 49% 개선 |
| range + filter 조건 | 300 ~ 550ms   | 120 ~ 250ms   | 약 78% 개선 |
| contains 검색       | 1.5s~2.5s     | 150 ~ 300ms   | 약 89% 개선 |
| 복합 조건 검색          | 2.5s~4s       | 200 ~ 400ms   | 약 91% 개선 |

### 문제

100만 건 데이터 기준, `LIKE '%keyword%'` 검색이 포함된 복합 조건 쿼리는 복합 인덱스를 설계해도 Full Scan 발생. 조건 조합(타입, 가격, 면적, 경매 상태 등)이 매번 달라져 모든 경우의
복합 인덱스를 만드는 것도 구조적으로 불가능.

### 해결: Elasticsearch 도입

- **Inverted Index** 기반으로 부분 문자열 검색도 Full Scan 없이 처리
- `range`, `term`, `bool filter` 조합 시 인덱스 재설계 불필요 (복합 인덱스의 왼쪽 컬럼 순서 제약 없음)
- **nori 형태소 분석기** 적용으로 한국어 검색 품질 향상
- 정합성은 DB, 조회 성능은 ES로 역할 분리

### 인덱스 설계

| 필드                   | 타입             | 이유        |
|----------------------|----------------|-----------|
| name, address        | text (nori)    | 부분 문자열 검색 |
| type, auction.status | keyword        | 정확 일치 필터  |
| price, builtYear     | long / integer | range 쿼리  |

--- 

</details>

<details>
<summary><strong>지오코딩 속도 개선 (단일 스레드 → 파티셔닝)</strong></summary>

| 구분     | 개선 전        | 개선 후       | 개선율       |
|--------|-------------|------------|-----------|
| 처리 건수  | 48,973건     | 51,208건    | -         |
| 소요 시간  | 1시간 19분 55초 | 19분 45초    | 75% 단축    |
| 분당 처리량 | 613건/min    | 2,592건/min | 약 4.2배 향상 |

### 문제

`geocodeStep`에 `TaskExecutor` 설정이 없어 단일 스레드로 순차 처리. 네이버/카카오 외부 API 호출이 누적되어 전국 데이터 약 5만 건 처리에 1시간 20분 소요.

### 멀티스레드 vs 파티셔닝

| 방식    | 특징                                | 선택 |
|-------|-----------------------------------|----|
| 멀티스레드 | 설정 간단, Reader를 스레드 간 공유 → 동시성 이슈  | ❌  |
| 파티셔닝  | 파티션별 독립 Reader/Writer → 안정적 병렬 처리 | ✅  |

### 파티셔닝 선택 이유

- `PendingDealReader`가 thread-safe하지 않아 멀티스레드 시 중복 읽기 발생 가능
- 파티션 단위로 실패/성공 추적 → 실패 파티션만 재실행 가능
- 4개 스레드로 API 호출량을 초당 약 42건으로 제어, Rate Limit 범위 내 유지

--- 

</details>

<details>
<summary><strong>대용량 입찰 트래픽 처리 (RabbitMQ 도입)</strong></summary>

| 구분              | TPS     | 평균 응답 속도 |
|-----------------|---------|----------|
| 개선 전 (DB Lock)  | 144.0/s | 9.19s    |
| 개선 후 (RabbitMQ) | 521.1/s | 359ms    |

### 문제

경매 종료 직전 입찰 요청 폭주 시 비관적 락 방식으로 DB 커넥션 풀 고갈. VUser 2,500 기준 평균 응답 9초, Connection Timed out 오류 다수 발생.

### 해결:  RabbitMQ 기반 비동기 처리 도입

- 입찰 요청 수신 시 유효성 검사만 빠르게 수행 후 큐에 적재하고 즉시 응답.
- Consumer가 순차적으로 DB에 반영해 DB 부하 분산.
- Manual-Ack으로
  처리 완료 후 메시지 제거, DLQ로 실패 메시지 격리.

---

</details>

<details>
<summary><strong> 분산 락 wait 설정에 따른 성능 비교 (Redis Redisson)</strong></summary>

### 목적

경매 종료 직전 트래픽 집중 구간에서 분산 락 wait 설정이 응답 속도·안정성에 미치는 영향 검증.
부동산 경매 특성상 잠재 참여자 약 20,000명 중 동시 입찰 가능 인원(약 2,400명)을 기준으로 시나리오 설계.

### 테스트 조건

K6 / Duration 1s / 성공 기준 `201`, `409` / wait = 0s vs wait = 2s 비교

---

☑️ **100명 동시 요청**

| 항목       | wait = 0s | wait = 2s | 개선율  |
|----------|-----------|-----------|------|
| 평균 응답 시간 | 77.42ms   | 56.58ms   | 27%↓ |
| p95      | 407.92ms  | 354.47ms  | 13%↓ |
| 처리 요청 수  | 808건      | 957건      | 18%↑ |

저부하 상황에서도 wait 설정이 일정 수준 성능 개선에 기여함

---

☑️ **1,000명 동시 요청**

| 항목       | wait = 0s | wait = 2s | 개선율  |
|----------|-----------|-----------|------|
| 평균 응답 시간 | 488.18ms  | 396.03ms  | 19%↓ |
| p95      | 859.71ms  | 680.13ms  | 21%↓ |
| 처리 요청 수  | 2,246건    | 2,578건    | 15%↑ |

동시 요청이 증가하면서 대기 시간 설정의 효과가 더 분명하게 나타남

---

☑️ **2,000명 동시 요청**

| 항목       | wait = 0s | wait = 2s | 개선율  |
|----------|-----------|-----------|------|
| 평균 응답 시간 | 1.19s     | 719.07ms  | 40%↓ |
| p95      | 2.28s     | 1.07s     | 53%↓ |
| 처리 요청 수  | 3,100건    | 3,587건    | 16%↑ |

고부하 상황에서 wait 설정이 응답 일관성에 가장 큰 영향을 미침

### 결론

- 동시 요청이 증가할수록 wait 설정의 효과가 뚜렷해짐
- 단순 처리량 개선을 넘어 p95 기준 최대 53% 개선으로, 트래픽 집중 상황에서 응답 속도와 안정성을 동시에 확보

---

</details>

## 🛠️ 기술 스택

### Language

![Java](https://img.shields.io/badge/Java-007396?style=for-the-badge&logo=openjdk&logoColor=white)
![Vue.js](https://img.shields.io/badge/Vue.js-4FC08D?style=for-the-badge&logo=vue.js&logoColor=white)

### Back-end

![Spring Framework](https://img.shields.io/badge/Spring_Framework-6DB33F?style=for-the-badge&logo=spring&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)
![Spring Data JPA](https://img.shields.io/badge/Spring_Data_JPA-6DB33F?style=for-the-badge&logo=spring&logoColor=white)
![QueryDSL](https://img.shields.io/badge/QueryDSL-005C84?style=for-the-badge)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white)

### Security

![Spring Security](https://img.shields.io/badge/Spring_Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white)

### Batch

![Spring Batch](https://img.shields.io/badge/Spring_Batch-6DB33F?style=for-the-badge&logo=spring&logoColor=white)

### Database

![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)

### Infra & CI/CD

![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white)
![Amazon EC2](https://img.shields.io/badge/Amazon_EC2-FF9900?style=for-the-badge&logo=amazonec2&logoColor=white)
![Amazon RDS](https://img.shields.io/badge/Amazon_RDS-527FFF?style=for-the-badge&logo=amazonrds&logoColor=white)
![Amazon S3](https://img.shields.io/badge/Amazon_S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![API Gateway](https://img.shields.io/badge/API_Gateway-FF4F8B?style=for-the-badge&logo=amazonapigateway&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![Self-Hosted Runner](https://img.shields.io/badge/Self_Hosted_Runner-000000?style=for-the-badge&logo=github&logoColor=white)

### Monitoring

![k6](https://img.shields.io/badge/k6-7D64FF?style=for-the-badge&logo=k6&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-0A2B4C?style=for-the-badge)
![Alloy](https://img.shields.io/badge/Alloy-FF6F00?style=for-the-badge)
![nGrinder](https://img.shields.io/badge/nGrinder-03C75A?style=for-the-badge)

### Test

![Swagger](https://img.shields.io/badge/Swagger-85EA2D?style=for-the-badge&logo=swagger&logoColor=black)
![Postman](https://img.shields.io/badge/Postman-FF6C37?style=for-the-badge&logo=postman&logoColor=white)
![Mockito](https://img.shields.io/badge/Mockito-5C2D91?style=for-the-badge)
![JUnit5](https://img.shields.io/badge/JUnit5-25A162?style=for-the-badge&logo=junit5&logoColor=white)

### Open API

![Toss Payments](https://img.shields.io/badge/Toss_Payments-0064FF?style=for-the-badge)
![Kakao OAuth](https://img.shields.io/badge/Kakao_OAuth-FFCD00?style=for-the-badge&logo=kakao&logoColor=black)
![KakaoTalk Message](https://img.shields.io/badge/KakaoTalk_Message-FFCD00?style=for-the-badge&logo=kakao&logoColor=black)
![Kakao Maps](https://img.shields.io/badge/Kakao_Maps-FFCD00?style=for-the-badge&logo=kakao&logoColor=black)
![Naver Maps](https://img.shields.io/badge/Naver_Maps-03C75A?style=for-the-badge&logo=naver&logoColor=white)
![Public Data Portal](https://img.shields.io/badge/Public_Data_Portal-005BAC?style=for-the-badge)

### Collaboration

![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![Slack](https://img.shields.io/badge/Slack-4A154B?style=for-the-badge&logo=slack&logoColor=white)
![Notion](https://img.shields.io/badge/Notion-000000?style=for-the-badge&logo=notion&logoColor=white)
![Figma](https://img.shields.io/badge/Figma-F24E1E?style=for-the-badge&logo=figma&logoColor=white)
![Draw.io](https://img.shields.io/badge/Draw.io-F08705?style=for-the-badge)

## 📁 설계 문서

### 🌐 와이어프레임

<details>
  <summary><strong>V1</strong></summary>
<img width="1146" height="721" alt="V1_와이어프레임" src="https://github.com/user-attachments/assets/27744354-4c87-448d-9f90-aa24314ed556" />
</details>

<details>
  <summary><strong>V2</strong></summary>
    <img width="2124" height="1064" alt="V2_와이어프레임" src="https://github.com/user-attachments/assets/07f284d1-54a9-4f31-add1-a609b463f772" />

</details>

### 📎 ERD

<details>
  <summary><strong>V1</strong></summary>
  <img width="598" height="572" alt="v1_ERD" src="https://github.com/user-attachments/assets/a922dd97-c71b-4bf3-9a6c-b016b35e802d" />
</details>

<details>
  <summary><strong>V2</strong></summary>
  <img width="1405" height="1352" alt="v2_ERD drawio" src="https://github.com/user-attachments/assets/acb6073d-8864-41d6-bd8c-8ddb1344798f" />
</details>

### 🌐 API 명세서

- [V1_API 명세서](https://www.notion.so/V1_API-3002dc3ef51480a48185edbc06c02d06)
- [V2_API 명세서](https://www.notion.so/teamsparta/V2_API-3002dc3ef51480518458d64b2917631d?source=copy_link)

---

## 🥇 팀원 소개

<table>
<tr>
<td align="center" width="20%">
<a href="https://github.com/jheony">
<img src="https://github.com/jheony.png" width="120px" style="border-radius: 50%;" alt="윤지현"/>
<br/><b>윤지현</b><br/>
<sub>리더</sub>
</a>
</td>
<td align="center" width="20%">
<a href="https://github.com/catlejuyeon">
<img src="https://github.com/catlejuyeon.png" width="120px" style="border-radius: 50%;" alt="성주연"/>
<br/><b>성주연</b><br/>
<sub>부리더</sub>
</a>
</td>
<td align="center" width="20%">
<a href="https://github.com/ziy0ung1234">
<img src="https://github.com/ziy0ung1234.png" width="120px" style="border-radius: 50%;" alt="한지영"/>
<br/><b>한지영</b><br/>
<sub>팀원</sub>
</a>
</td>
<td align="center" width="20%">
<a href="https://github.com/i-am-dahee">
<img src="https://github.com/i-am-dahee.png" width="120px" style="border-radius: 50%;" alt="이다희"/>
<br/><b>이다희</b><br/>
<sub>팀원</sub>
</a>
</td>
</tr>
</table>

<br/>

---

## 🚀 Quick Start

### Prerequisites

```bash
- Java 17+
- Docker & Docker Compose
```

### Local Development

**1. 저장소 클론**

```bash
git clone https://github.com/your-org/budong-budong.git
cd budong-budong
```

**2. 인프라 실행**

```bash
docker-compose up -d
```

**3. 서비스 실행**

```bash
./gradlew bootRun --args='--spring.profiles.active=local'
```

### Environment Variables

```properties
# Database
DB_URL=jdbc:mysql://localhost:3306/budongbudong
DB_USERNAME=root
DB_PASSWORD=password
# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
# Elasticsearch
ES_HOST=localhost
ES_PORT=9200
# RabbitMQ
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
# OAuth
KAKAO_CLIENT_ID=your_kakao_client_id
GOOGLE_CLIENT_ID=your_google_client_id
# Payment
TOSS_PAYMENTS_SECRET_KEY=your_toss_secret_key
# Map API
NAVER_MAPS_CLIENT_ID=your_naver_client_id
KAKAO_MAPS_KEY=your_kakao_maps_key
```

---

## 📄 License

This project is licensed under the MIT License.

---

<div align="center">

**🏠 부동부동 - 여걸식스 팀**

*"중개 수수료 없이, 시장이 가격을 결정합니다"*

</div>
