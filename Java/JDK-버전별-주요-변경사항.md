# JDK 버전별 주요 변경사항

> 태그: `#java` `#jdk` `#lts`<br>
> 작성일: 2026-06-27<br>
> 최종 수정일: 2026-06-27

## 정의

JDK는 8/11/17/21/25를 LTS(Long-Term Support)로, 그 사이 버전(9~10, 12~16, 18~20, 22~24, 26...)을 6개월 주기 비LTS로 배포한다. 실무 의사결정 단위는 비LTS 개별 버전이 아니라 LTS 간 누적 변경이다 — 비LTS는 그 다음 LTS에 들어갈 기능이 preview/final로 가는 중간 과정으로 보는 게 정확하다.

## 특징 / 상세

핵심 키워드: `LTS`, `JEP`, `Virtual Thread`, `Preview Feature`, `release cadence`

### 1. 릴리즈 주기

- JDK 8(2014) 이후 한동안 비정기 릴리즈였다가, JDK 9(2017)부터 6개월 주기 고정 릴리즈로 전환됐다.
- 21까지는 LTS가 3년마다(11→17→21), 21 이후로는 2년마다(21→25→29)로 주기가 바뀌었다.
- 다음 LTS는 29, 2027년 9월 예정.

### 2. 8 → 11

- 모듈 시스템(JPMS) 도입 — JDK 9, [JEP 261](https://openjdk.org/jeps/261).
- 표준 HTTP Client 추가 — JDK 11, [JEP 321](https://openjdk.org/jeps/321) (JDK 9 incubator → 11 표준화).
- Java EE/CORBA 모듈 제거 — JDK 11, [JEP 320](https://openjdk.org/jeps/320).
- `var`(지역 변수 타입 추론) — JDK 10, [JEP 286](https://openjdk.org/jeps/286).

### 3. 11 → 17

- Record 표준화 — JDK 16, [JEP 395](https://openjdk.org/jeps/395).
- Sealed Classes 표준화 — JDK 17, [JEP 409](https://openjdk.org/jeps/409). 서브타입을 제한해 switch의 exhaustiveness 검사를 컴파일러가 보장할 수 있게 한다.
- Pattern Matching for switch — JDK 17 preview([JEP 406](https://openjdk.org/jeps/406)), Sealed Classes 위에서 동작.
- JDK 내부 API 강한 캡슐화를 기본값으로 — JDK 17, [JEP 403](https://openjdk.org/jeps/403).

### 4. 17 → 21 — 가장 큰 전환점

- **Virtual Thread 표준화** — JDK 21, [JEP 444](https://openjdk.org/jeps/444). JDK 19 preview([JEP 425](https://openjdk.org/jeps/425)) → JDK 20 2차 preview([JEP 436](https://openjdk.org/jeps/436)) → JDK 21 표준화 순서로 안정화됐다.
  - 21 표준화 시점에는 `synchronized` 블록·일부 native 호출 안에서 블로킹하면 virtual thread가 carrier(OS) thread에 고정(pinning)되는 제약이 남아 있었다.
- Pattern Matching for switch 표준화 — JDK 21, [JEP 441](https://openjdk.org/jeps/441).
- Record Pattern 표준화 — JDK 21, [JEP 440](https://openjdk.org/jeps/440).
- Sequenced Collections — JDK 21, [JEP 431](https://openjdk.org/jeps/431).
- Generational ZGC — JDK 21, [JEP 439](https://openjdk.org/jeps/439).

### 5. 21 → 25

- **Virtual Thread pinning 해소** — JDK 24, [JEP 491](https://openjdk.org/jeps/491) (25가 아니라 **24**다 — 이전에 25로 잘못 말한 부분을 정정). `synchronized` 안에서 블로킹해도 virtual thread가 carrier를 점유한 채 멈추지 않고, monitor 소유권을 virtual thread 단위로 추적해 mount/unmount 시점마다 갱신하는 방식으로 구현됐다.
- Scoped Values 표준화 — JDK 25, [JEP 506](https://openjdk.org/jeps/506).
- Compact Object Headers — JDK 25, [JEP 519](https://openjdk.org/jeps/519). 객체 헤더 크기를 줄여 메모리 사용량과 캐시 효율을 개선.
- Structured Concurrency — JDK 25에서도 아직 5차 preview([JEP 505](https://openjdk.org/jeps/505)), GA 아님. `--enable-preview` 필요하고 API가 버전마다 바뀌어왔다 — 프로덕션에 쓰기 전 해당 버전 공식 문서 확인 필수.
- JDK 25 전체 변경 목록(1차 출처): [JEPs in JDK 25 integrated since JDK 21](https://openjdk.org/projects/jdk/25/jeps-since-jdk-21).

### 6. 26 (비LTS, 2026.03 출시)

- HTTP/3 지원 — HTTP Client API, [JEP 517](https://openjdk.org/jeps/517).
- Applet API 제거 — [JEP 504](https://openjdk.org/jeps/504) (17부터 deprecated).
- G1 GC 동기화 오버헤드 감소 — [JEP 522](https://openjdk.org/jeps/522).
- 비LTS이므로 프로덕션 표준 채택 대상은 아니고, 다음 LTS(29)에 들어갈 기능을 먼저 보는 용도로 보는 게 맞다.

## 트레이드오프

LTS만 추적할지, 비LTS까지 따라갈지에 대한 트레이드오프:

| 항목 | LTS만 채택 | 비LTS까지 추적 |
|---|---|---|
| 일관성 | 높음 — 2~3년 단위로만 마이그레이션, 코드베이스 안정 | 낮음 — 6개월마다 preview API가 바뀔 수 있어 코드가 흔들림 |
| 가용성 | 영향 없음(런타임 가용성과 직접 관련 없음) | 영향 없음 |
| 지연 | 최신 GC/런타임 개선(예: Generational ZGC, Compact Object Headers)을 1~2년 늦게 받음 | 최신 개선을 가장 먼저 받음 |
| 비용 | 마이그레이션 비용이 낮은 빈도로 집중됨 | 마이그레이션 비용이 6개월마다 분산 발생 |
| 운영부담 | 벤더(Oracle/Temurin 등) LTS 지원 기간 내 패치 받기 용이 | 비LTS는 다음 버전 나오면 패치 지원 종료 — 운영 중 강제 업그레이드 압박 |

규모 가정 없음 — 이 표는 버전 채택 정책 자체의 구조적 트레이드오프이며 QPS/데이터 크기와 무관하다.

## 실무 경험

해당 없음

## 참고

- [JDK 21 — OpenJDK](https://openjdk.org/projects/jdk/21/) — 확인일 2026-06-27
- [JEPs in JDK 25 integrated since JDK 21 — OpenJDK](https://openjdk.org/projects/jdk/25/jeps-since-jdk-21) — 확인일 2026-06-27
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444) — 확인일 2026-06-27
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491) — 확인일 2026-06-27
  - "synchronized no longer pins virtual threads to their carriers" — JDK 24에서 구현됨
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506) — 확인일 2026-06-27
- [JEP 519: Compact Object Headers](https://openjdk.org/jeps/519) — 확인일 2026-06-27
- [JEP 505: Structured Concurrency (Fifth Preview)](https://openjdk.org/jeps/505) — 확인일 2026-06-27
- [JEP 409: Sealed Classes](https://openjdk.org/jeps/409) — 확인일 2026-06-27
- [JEP 321: HTTP Client (Standard)](https://openjdk.org/jeps/321) — 확인일 2026-06-27
- [JEP 320: Remove the Java EE and CORBA Modules](https://openjdk.org/jeps/320) — 확인일 2026-06-27
- [JEP 261: Module System](https://openjdk.org/jeps/261) — 확인일 2026-06-27
- [JEP 517: HTTP/3 for the HTTP Client API](https://openjdk.org/jeps/517) — 확인일 2026-06-27
- 보조 출처(2차, 교차 확인용): [Java 25, the Next LTS Release, Delivers Finalized Features — InfoQ](https://www.infoq.com/news/2025/09/java25-released/), [Java 26: Better Language, Better APIs, Better Runtime — Inside.java](https://inside.java/2026/05/19/javaone-better-jdk26/)

## 관련 내용

- [가비지-컬렉션](./가비지-컬렉션.md) — Generational ZGC(21)와 연결
- [Coroutine](../운영체제/Coroutine.md) — Virtual Thread와 코루틴의 개념적 차이 비교 시 참고
