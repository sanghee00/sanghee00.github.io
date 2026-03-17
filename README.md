# 한상희 포트폴리오

> 🔗 **포트폴리오 보기 → [sanghee00.github.io](https://sanghee00.github.io/)**

---

## 프로필

**"빠르다는 느낌이 아닌, 수치로 검증하는 백엔드 개발자"**
성능 병목을 수치로 분석하고 해결하는 것에 관심이 많은 백엔드 개발자입니다.

| | |
|---|---|
| 이메일 | hansanghee112@gmail.com |
| GitHub | [github.com/sanghee00](https://github.com/sanghee00) |
| 연락처 | 010-9934-8459 |
| Blog | [velog.io/@tkdgml82](https://velog.io/@tkdgml82/posts) |

**핵심 역량** `Java / Spring Boot` `MySQL / Redis` `AWS 인프라` `성능 최적화` `k6 부하테스트` `CI/CD`

---

## 프로젝트

### LoL_Pedia

> 프로게이머 선수 정보 및 경기 기록 조회 사이트

- **기간:** 2026.01 ~ 2026.03
- **규모:** 1인 개발
- **스택:** Java 21, Spring Boot 3.5.7
- **GitHub:** [github.com/sanghee00/LOL_Pedia](https://github.com/sanghee00/LOL_Pedia)

#### 사용기술

| 기술 | 선택 이유 |
|---|---|
| Java 21 | LTS 버전으로 장기 지원이 보장되며, Virtual Threads(Project Loom)를 정식 지원해 높은 동시성 처리에 유리하여 사용했습니다. |
| Spring Boot 3.5.7 | Java 21과 최적 호환되며, 간단한 설정만으로 Virtual Threads를 네이티브로 지원하여 사용했습니다. |
| MySQL | 정형화된 경기 데이터의 복잡한 연관관계를 JOIN으로 효율적으로 처리하기 위해 사용했습니다. |
| EC2 — NAT 인스턴스 | AWS NAT Gateway 대신 직접 구축하여 Private Subnet의 외부 통신 비용을 대폭 절감. |
| EC2 — 스프링 서버 2대 | 목표 트래픽 TPS 1,000 이상을 안정적으로 수용하기 위해 Scale-out 구성. |
| RDS | Private Subnet에 배치해 외부 직접 접근을 원천 차단하여 보안을 높임. |
| Bastion Host | 외부망에서 Private Subnet 내부에 안전하게 접근하기 위해 사용. |
| ElastiCache (Redis) | 빈번하게 조회되는 경기 데이터를 캐싱하여 DB 부하를 줄이고 응답 속도를 대폭 개선. |
| S3 / SSM / ECR / CodeDeploy | CI/CD 파이프라인 구성. SSM으로 민감정보 분리, ECR로 이미지 관리, CodeDeploy로 다중 EC2 자동 배포. |
| CloudWatch | 서버 자원(CPU, 메모리)를 실시간으로 모니터링하여 트래픽 병목 현상을 추적. |

#### 트러블슈팅

---

**TS 1 — Java 21 Virtual Thread 도입 시 성능 저하 발견 → Carrier Thread 한계 분석 후 롤백, L1/L2 캐시로 전환해 throughput 3배 달성**

**문제 원인**
1. Redis 캐싱 후 p95 개선 목적으로 Virtual Thread 도입했으나 avg 158ms → 201ms로 성능 악화.
2. 제한된 Carrier Thread(vCPU 2개) 환경에 500 VU의 요청이 집중되어 스케줄링 오버헤드 폭증 병목 확인.
3. Redis 응답 1~2ms로 너무 빨라 mount/unmount 비용이 이득보다 큼.

**해결 과정**
1. Lettuce 핀닝 이슈(synchronized → ReentrantLock) 확인 → 이미 v6.6.0에서 수정됨.
2. 하드웨어 한계임을 확인 후 Virtual Thread 비활성화로 롤백.
3. 플랫폼 스레드(기본 200) 유지, L1/L2 캐시로 방향 전환.

**결과**

| 단계 | avg 응답시간 | throughput | 비고 |
|---|---|---|---|
| 기본 스레드 | 158ms | 2,516/s | 베이스라인 |
| Virtual Thread 적용 | 201ms (↑27%) | 2,199/s (↓12.6%) | ❌ 성능 악화 |
| **롤백 + L1/L2 캐시** | **64ms** | **5,990/s (+138%)** | ✅ 최종 |

> 💡 배운 점: I/O가 빠른 환경에서 VT는 고사양 서버에서 유효한 기술. 도입 전 하드웨어 환경 검증 필수.

---

**TS 2 — 팀/선수 조회 API DB 부하 → Redis + Caffeine 캐시 도입으로 throughput 122/s에서 5,990/s (+4,800%) 개선**

**문제 원인**
1. 부하테스트 시 팀/선수 조회 API에 동일 데이터 반복 요청으로 DB 부하 급증.
2. Redis 도입 후에도 매 요청마다 네트워크 왕복 비용(~2ms) 누적.
3. 500 VU 동시 요청 시 Redis 연결 경합 발생.

**해결 과정**
1. Redis 캐싱 1차 적용 → RPS 122/s에서 7,190/s 달성, avg 153ms → 57ms 개선.
2. 잔존 네트워크 비용 해결을 위해 Caffeine(L1) + Redis(L2) 계층 캐시 직접 구현.
   - key 공간 분석 → 좁은 캐시(top8)만 L1 적용, 넓은 캐시(match)는 L2 전용 유지.
   - L1 TTL 30초 설정 → 만료 시 500 VU 동시 Redis 호출로 p99 742ms 스파이크 발생.
   - 원인 분석 후 TTL 3분으로 조정 → 스파이크 제거.

**결과**

| 단계 | avg | throughput | 비고 |
|---|---|---|---|
| Redis 단독 | 57ms | 2,516/s | |
| L1/L2 (TTL 30초) | 128ms | 3,081/s | p99 스파이크 발생 |
| **L1/L2 (TTL 3분)** | **64ms** | **5,990/s (+138%)** | ✅ 최종 |

> 💡 배운 점: L1 캐시 효과는 key 공간과 TTL 설계가 핵심. TTL 설정 한 줄로 p99 742ms → 227ms 개선.

---

**TS 3 — YEAR() 함수 및 OR 조건으로 인한 인덱스 풀스캔 문제 → 범위 조건 변경 및 쿼리 분리로 스캔 rows 98.3% 감소**

**문제 원인**
1. `YEAR(match_date)` → 컬럼에 함수 적용으로 인덱스 scan 불가 (rows 88,074).
2. `team_a_id = ? OR team_b_id = ?` → index_merge + filesort 발생 (rows 8,771).

**해결 과정**
1. YEAR() 제거 → BETWEEN 범위 조건으로 변경 → range scan 활성화.
2. OR 단일 쿼리 → teamA/teamB 쿼리 분리 → FK 인덱스 직접 사용, filesort 제거.
3. 두 결과를 `Stream.concat` → matchDate DESC 정렬.

**결과**

| 개선 항목 | Before (rows) | After (rows) | 감소율 |
|---|---|---|---|
| YEAR() 제거 | 88,074 | 1,481 | **98.3% ↓** |
| OR 조건 분리 | 8,771 | 117 | **98.7% ↓** |

> 💡 배운 점: 컬럼에 함수를 씌우거나 OR 조건은 인덱스를 무력화한다.

---

**TS 4 — L1/L2 계층 캐시로 Redis 삭제 구간 흡수, 에러율 0% · 응답시간 변화 0.5ms 이내 검증**

**문제 원인**
1. 캐시 만료 순간 다수의 요청이 동시에 cache miss.
2. 모든 요청이 DB로 직접 도달 → DB 부하 폭증 (Cache Stampede).
3. `@Cacheable` 기본 설정은 동시 DB 호출 방어 메커니즘 없음.

**해결 과정**
1. L1(Caffeine TTL 3분) + L2(Redis TTL 4h) 계층 구조로 방어막 이중화.
2. Redis 키를 50ms마다 강제 삭제하는 `flush-stampede.sh` 직접 제작.
3. k6 3단계 시나리오로 검증:
   - **pre(30s):** 캐시 정상 HIT 구간
   - **stampede(20s):** Redis 키 50ms마다 삭제
   - **post(30s):** 캐시 회복 확인

**결과**

| 구간 | avg | p99 | 에러율 |
|---|---|---|---|
| pre (캐시 HIT) | 57.59ms | 176ms | 0% |
| stampede (Redis 삭제 중) | 58.05ms | 186ms | 0% |
| **post (회복 후)** | **57.30ms** | **173ms** | **0%** |

📊 Redis 단독 vs L1/L2 스탬피드 방어 비교: 동일 스탬피드 조건에서 Redis 단독 시 throughput 3,211 RPS까지 저하된 반면, L1/L2 적용 시 4,866 RPS 유지 → L1 캐시 방어로 **51% 높은 throughput 유지**.

> 💡 핵심: L1 TTL(3분)이 살아있는 한 Redis 삭제는 서비스에 영향 없음. 에러율 0% 달성 (500 VU, 20초간 Redis 키 삭제 중에도 무중단).

---

**TS 5 — GitHub Actions + AWS CodeDeploy CI/CD 파이프라인 구축 및 보안 강화**

**문제 원인**
1. 코드 변경 시마다 수동 빌드 → EC2 접속 → 배포 반복으로 휴먼 에러 위험.
2. 환경변수, DB 접속 정보 등 민감정보를 코드에 직접 관리.

**해결 과정**
1. GitHub Actions → Gradle 빌드 → Docker 이미지 생성 → ECR push → CodeDeploy 배포 자동화.
2. AWS SSM Parameter Store에 application.yml, .env 분리 저장 → 배포 시점에 SSM에서 pull하여 주입.

**결과**
1. main 브랜치 push 시 EC2 자동 배포.
2. 소스코드 내 시크릿 노출 위험 제거.

---

**TS 6 — Jackson 역직렬화 취약점(CVE) 발견 후 allowlist 적용으로 임의 클래스 실행 차단**

**문제 원인**
1. Jackson에서 `getPolymorphicTypeValidator`를 사용함.
2. 기본적으로 허용 객체를 설정하지 않으면 `LaissezFaireSubTypeValidator`를 사용.
3. 해당 객체는 내부적으로 모든 타입 허용 → 공격자가 악의적인 클래스를 JSON에 심어 역직렬화 시점에 임의 코드 실행 가능 (CVE 취약점).

📝 [관련 블로그 글 보기](https://velog.io/@tkdgml82/Project-Redis-%EC%A7%81%EB%A0%AC%ED%99%94-%EB%B3%B4%EC%95%88-%EC%84%A4%EC%A0%95)

**해결 과정**
1. `BasicPolymorphicTypeValidator` allowlist 방식으로 교체.
2. DTO, java.util.*, java.time.* 등 허용 패키지만 명시적 등록.

**결과**
1. 허용되지 않은 클래스 역직렬화 시도 시 예외 발생으로 차단.
2. 신뢰할 수 있는 타입만 역직렬화 허용.

---

### 이음

> AI 의존으로 저하된 글쓰기 능력 향상을 위해, 사용자가 주제를 선택해 글을 쓰면 AI가 피드백과 점수를 제공하는 플랫폼 · 구름 딥다이브 최종 프로젝트

- **기간:** 2025.08 ~ 2025.09
- **규모:** Frontend 1, Backend 4, PM 1, Designer 1
- **GitHub:** [github.com/Team-i-can-do-it/api-server](https://github.com/Team-i-can-do-it/api-server)

#### 기여한 부분

| 분야 | 내용 |
|---|---|
| 성능 | 사용자가 랜덤으로 글쓰기를 선택 시 해당 주제를 제공하는 **API** 성능 병목 발견 및 최적화 (응답속도 137ms → 42ms, 70% 단축) 및 Self-Invocation 문제 직접 발견 후 해결. |
| 기능 | 글쓰기, AI 피드백, 랜덤 주제 제공 API 구현 (백엔드 4명 중 핵심 API 담당). |
| 협업 | Swagger 기반 API 문서화로 프론트엔드 협업 효율 향상. GitHub PR '최소 2인 승인' 규칙 도입 주도. |

#### 사용기술

| 기술 | 선택 이유 |
|---|---|
| Spring Boot | 팀 공통 기술 스택. |
| QueryDSL | `ORDER BY RAND()` 풀스캔을 타입 안전한 동적 쿼리로 대체. |
| EhCache | 외부 캐시 서버 없이 JVM 레벨에서 반복 조회 캐싱. |
| Swagger | 프론트엔드와의 API 명세 공유로 협업 효율 향상. |

#### 트러블슈팅

---

**TS 1 — 글쓰기 플랫폼의 랜덤 주제 제공 API — @Cacheable Self-Invocation 미적용으로 캐싱 무효화 발견 및 해결**

**문제 원인**
1. 캐싱을 하지 않고 매 요청마다 DB에서 랜덤 주제를 조회하는 방식으로 동작.
2. `@Cacheable`을 Service 계층 내부에서 호출하면 Spring 프록시 객체를 거치지 않아 캐싱이 적용되지 않는 Self-Invocation 문제 발생.

**해결 과정**
1. `TopicProxy` 클래스를 별도 생성하여 캐싱 로직을 외부 Bean으로 분리.
2. 프록시 객체를 통해 호출하도록 변경 → `@Cacheable` 정상 적용.

**결과**

| 지표 | Before | After | 개선율 |
|---|---|---|---|
| 평균 응답시간 (avg) | 23.14ms | 15.08ms | **34.8% ↓** |
| p95 응답시간 | 31.82ms | 21.6ms | **32.1% ↓** |

> 💡 배운 점: Spring AOP 프록시 기반 캐싱은 같은 클래스 내 자기 호출 시 무효화됨. 캐싱 로직은 별도 Bean으로 분리해야 함.

---

**TS 2 — 글쓰기 플랫폼의 랜덤 주제 제공 API — ORDER BY RAND() 풀스캔 제거로 응답속도 137ms → 42ms 개선**

**문제 원인**
1. 캐싱의 한계: 데이터가 많아질수록 메모리 사용량 급증 우려. 로컬 캐시는 오히려 성능 저하 가능성 있다고 판단.
2. `ORDER BY RAND()`는 인덱스를 타지 못해 전체 테이블 풀스캔 발생.

**해결 과정**
1. **1차 시도:** `COUNT(*)`으로 카테고리별 개수를 구한 뒤 `random()`과 OFFSET 서브쿼리 적용 → JPA에서 OFFSET절에 서브쿼리 사용 불가 문제로 실패.
2. **2차 시도(최종):** `MIN(ID)`, `MAX(ID)`로 PK 범위 파악 후 Java 코드 단에서 랜덤 ID를 생성하여 단건 조회로 변경 → 기본 인덱스 활용.
3. QueryDSL로 마이그레이션하여 타입 안전한 동적 쿼리로 전환.

**결과**

| 단계 | 방식 | 응답시간 | 이전 대비 |
|---|---|---|---|
| 원본 | ORDER BY RAND() | 137ms | — |
| 1차 | COUNT + OFFSET 서브쿼리 | 82ms | 40.1% ↓ |
| **2차 (최종)** | **MIN/MAX + Java 랜덤** | **42ms** | **원본 대비 69.3% ↓** |

> 💡 배운 점: ORM의 제약(JPA OFFSET 서브쿼리 불가)을 파악하고 Java 코드 단으로 로직을 이동하는 것도 유효한 해결책.
