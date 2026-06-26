# WAS vs WebServer

> 태그: `#network` `#was` `#webserver` `#tomcat` `#nginx` `#concurrency`<br>
> 작성일: 2026-06-26<br>
> 최종 수정일: 2026-06-26

## 정의

WAS와 WebServer의 차이는 "정적 vs 동적 처리"로 단순화되지만, 실무에서 둘을 같이 두는 진짜 이유는 동시성 처리 모델의 차이(스레드당 요청 vs 이벤트 루프)와 WAS의 유한한 스레드 자원을 느린 클라이언트로부터 보호하는 데 있으며, "WAS/WebServer"라는 용어 구분 자체는 스펙이 정의한 경계가 아니라 업계 관례다.

## 특징 / 상세

### "정적 vs 동적" 구분의 한계

`리버스-프록시.md`/`WAS.md`에 있는 한 줄 정의("웹서버=정적, WAS=동적")는 입문 수준에서는 맞지만, "그럼 왜 정적 파일 서빙만 하는 게 아니라 항상 WAS 앞에 WebServer를 두는가?"라는 질문에는 답하지 못한다. 정적 파일이 거의 없는 API 전용 서비스에서도 Nginx를 WAS 앞에 두는 경우가 많은 이유는 아래 두 가지(스레드 보호, 동시성 모델)에 있다.

### 왜 WAS 앞에 WebServer를 두는가 — 스레드 자원 보호

Tomcat은 기본적으로 요청 하나당 스레드 하나를 점유하는 thread-per-request 모델이고, 스레드 풀 크기는 유한하다(`maxThreads` 기본값 200).

> "Each incoming, non-asynchronous request requires a thread for the duration of that request."
> — [Apache Tomcat 10.1 HTTP Connector — Introduction](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html#Introduction)

느린 클라이언트(저속 네트워크, 의도적 슬로우 공격 등)가 WAS와 직접 연결되면, 응답을 다 받아갈 때까지 그 요청을 처리한 스레드가 점유된 채로 묶인다. 동시 접속한 느린 클라이언트 수가 `maxThreads`에 가까워지면 스레드 풀이 고갈되어 다른 정상 요청까지 대기하게 된다(thread starvation).

Nginx를 앞에 두면 Nginx가 클라이언트와의 느린 송수신을 떠맡고, WAS와는 내부망에서 빠르게 응답을 주고받기 때문에 WAS 스레드가 느린 클라이언트 때문에 오래 점유되지 않는다. 즉 핵심은 "정적 파일 서빙"이 아니라 "유한한 WAS 스레드 자원을 느린 클라이언트로부터 격리"하는 것이다.

### 동시성 모델 차이 — thread-per-request vs event-driven

"Tomcat = blocking I/O, Nginx = non-blocking I/O"는 부정확하다. Tomcat의 기본 커넥터도 소켓 레벨에서는 이미 논블로킹 NIO를 쓴다.

> "The default value is `HTTP/1.1` which uses a Java NIO based connector."
> — [Apache Tomcat 10.1 HTTP Connector — Common Attributes (protocol)](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html#Common_Attributes)

실제 차이는 블로킹/논블로킹이 아니라 **요청 처리 단위를 스레드에 묶는지 여부**다.

- Tomcat: 비동기 서블릿(Async Servlet)을 쓰지 않는 한, 요청을 받아 컨트롤러 로직을 실행하는 동안 그 요청은 스레드 하나를 계속 점유한다. 동시 처리 가능한 요청 수는 결국 스레드 풀 크기에 묶인다.
  > "Waiting within the servlet is an inefficient operation as it is a blocking operation that consumes a thread... can cause thread starvation and poor quality of service for an entire web container."
  > — [Jakarta Servlet Spec 6.0 §2.3.3.3 Asynchronous processing](https://jakarta.ee/specifications/servlet/6.0/jakarta-servlet-spec-6.0.html#asynchronous-processing)
- Nginx: 이벤트 루프(epoll/kqueue) 기반으로, 연결 수가 스레드/프로세스 수에 비례하지 않는다. 적은 워커 프로세스로도 수천~수만 연결을 동시에 멀티플렉싱할 수 있다.
  > epoll/kqueue를 "efficient method"로 명시.
  > — [nginx — Connection processing methods](https://nginx.org/en/docs/events.html)

정적 파일 서빙·SSL Termination·로드밸런싱 같은 I/O 대기가 많은 작업을 이벤트 루프 모델인 Nginx가 흡수하고, CPU 바운드인 비즈니스 로직만 thread-per-request 모델인 WAS가 처리하도록 역할을 나누는 것이 합리적이다.

### "WAS"/"웹서버" 용어 구분 — 스펙은 경계를 정의하지 않는다

Jakarta Servlet 스펙은 서블릿 컨테이너를 웹서버/애플리케이션 서버의 "일부"로 정의할 뿐, "WAS"와 "웹서버"를 구분하는 용어나 기준을 별도로 정의하지 않는다.

> "The servlet container is a part of a web server or application server that provides the network services over which requests and responses are sent, decodes MIME-based requests, and formats MIME-based responses."
> — [Jakarta Servlet Spec 6.0 §1.2 What is a Servlet Container?](https://jakarta.ee/specifications/servlet/6.0/jakarta-servlet-spec-6.0.html#what-is-a-servlet-container)

실무에서 통용되는 구분은 스펙이 아니라 관례다.

| 구분 | 예시 | 특징 |
|---|---|---|
| 순수 웹서버 | Apache HTTP Server, Nginx | 서블릿 컨테이너 없음. 정적 파일/리버스 프록시/로드밸런싱 전담 |
| 경량 서블릿 컨테이너 (관례상 "WAS") | Tomcat, Jetty, Undertow | 서블릿/JSP 컨테이너만 제공. EJB·JTA 등 풀스펙 Java EE 기능 없음 |
| 풀스펙 Java EE WAS | JBoss(WildFly), WebLogic | 서블릿 컨테이너 + EJB 컨테이너 + JTA 등 Java EE 전체 스펙 구현 |

Tomcat을 "WAS"라고 부르는 것은 틀린 말은 아니지만, 더 정확히는 "경량 서블릿/JSP 컨테이너"이고, JBoss/WebLogic 같은 풀스펙 Java EE WAS와는 제공 기능 범위가 다르다. 이 경계를 스펙에서 수치나 체크리스트로 정의하는지는 **모름** — 추가로 확인하려면 Java EE/Jakarta EE 플랫폼 스펙(Jakarta EE Platform Specification)에서 WAS가 구현해야 하는 컴포넌트 목록을 직접 대조해야 한다.

## 트레이드오프

가정: 일반적인 웹 서비스, 정적 자원 비중이 일부 있음, 수백~수천 RPS 규모. API 전용·정적 자원이 거의 없는 서비스는 결론이 달라질 수 있다(아래 실무 경험 참고).

| 항목 | WAS 단독 (Tomcat만 직접 노출) | WebServer + WAS (Nginx → Tomcat) |
|---|---|---|
| 일관성 | 영향 없음 (구성과 무관) | 영향 없음 |
| 가용성 | 느린 클라이언트가 스레드 풀을 고갈시키면 전체 서비스 영향 가능 | Nginx가 느린 클라이언트를 흡수해 WAS 스레드 고갈 위험 낮춤. 단 Nginx 자체가 단일 장애점이 되므로 보통 이중화 필요 |
| 지연 | 정적 파일도 WAS가 처리 → 불필요한 지연 | 정적 파일은 Nginx가 즉시 응답, 동적 요청만 WAS로 전달 |
| 비용 | 구성 요소 적음 | Nginx 인스턴스/이중화 비용 추가 |
| 운영부담 | 단일 구성 요소라 단순 | Nginx 설정·인증서·이중화 관리 부담 추가. Nginx~WAS 구간 평문 통신에 대한 내부망 보안 정책 필요 |

## 실무 경험

해당 없음

## 참고

- [Apache Tomcat 10.1 HTTP Connector](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html) — 확인일 2026-06-26
  - Introduction: thread-per-request 모델, `maxThreads` 기본값 200
  - Common Attributes: 기본 커넥터가 Java NIO 기반임
- [Jakarta Servlet Spec 6.0](https://jakarta.ee/specifications/servlet/6.0/jakarta-servlet-spec-6.0.html) — 확인일 2026-06-26
  - §1.2 What is a Servlet Container? — 서블릿 컨테이너는 웹서버/WAS의 "일부"로 정의, 용어 경계는 스펙이 정의하지 않음
  - §2.3.3.3 Asynchronous processing — 동기 처리 시 스레드 점유로 인한 thread starvation 위험
- [nginx — Connection processing methods](https://nginx.org/en/docs/events.html) — 확인일 2026-06-26

## 관련 내용

- [WAS](WAS.md)
- [리버스-프록시](리버스-프록시.md)
