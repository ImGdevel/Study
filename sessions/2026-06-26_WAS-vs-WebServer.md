# 2026-06-26 세션: WAS와 WebServer 차이

## 질문

WAS와 WebServer의 차이점은 무엇인가 — 왜 정적 자원이 거의 없어도 WAS 앞에 WebServer(Nginx)를 두는지, Tomcat과 Nginx의 동시성 처리 모델이 어떻게 다른지, "WAS"/"웹서버" 용어 구분이 스펙에 정의되어 있는지.

## 결론

1. "정적 vs 동적"은 입문 수준 구분일 뿐, 실제 이유는 WAS의 유한한 스레드 풀(thread-per-request, `maxThreads` 기본 200)을 느린 클라이언트로부터 격리하는 것.
2. Tomcat 기본 커넥터도 소켓 레벨에서는 이미 NIO(논블로킹)다. Nginx와의 진짜 차이는 블로킹/논블로킹이 아니라 thread-per-request(Tomcat) vs event-loop 멀티플렉싱(Nginx epoll/kqueue)이라는 동시성 모델 자체.
3. Jakarta Servlet 스펙은 "WAS"/"웹서버" 용어를 구분해 정의하지 않는다. 실무 관례상 Apache/Nginx=순수 웹서버, Tomcat/Jetty=경량 서블릿 컨테이너, JBoss/WebLogic=풀스펙 Java EE WAS로 나누지만 이는 관례이며 스펙상 명문화된 경계는 아니다.

## 근거 / 출처

- [Apache Tomcat 10.1 HTTP Connector — Introduction](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html#Introduction) — thread-per-request, "Each incoming, non-asynchronous request requires a thread for the duration of that request."
- [Apache Tomcat 10.1 HTTP Connector — Common Attributes](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html#Common_Attributes) — 기본 커넥터가 Java NIO 기반
- [Jakarta Servlet Spec 6.0 §1.2](https://jakarta.ee/specifications/servlet/6.0/jakarta-servlet-spec-6.0.html#what-is-a-servlet-container) — 서블릿 컨테이너는 웹서버/WAS의 "일부"
- [Jakarta Servlet Spec 6.0 §2.3.3.3](https://jakarta.ee/specifications/servlet/6.0/jakarta-servlet-spec-6.0.html#asynchronous-processing) — 동기 처리 시 thread starvation 위험
- [nginx — Connection processing methods](https://nginx.org/en/docs/events.html) — epoll/kqueue 이벤트 기반 모델

## 검증 상태

- [x] 1차 출처 확인됨
- [ ] 코드/실측으로 직접 검증함
- [ ] 미검증 (추정만 함, 검증 방법: ___)

## 모르는 것 / 후속 과제

- Tomcat(경량 서블릿 컨테이너)과 JBoss/WebLogic(풀스펙 Java EE WAS)의 경계를 Jakarta EE Platform Specification이 컴포넌트 체크리스트로 명문화하는지 — 모름. 확인하려면 Jakarta EE Platform Spec에서 WAS가 구현해야 하는 컴포넌트(EJB, JTA 등) 목록을 직접 대조해야 함.
- 사용자가 공유한 ChatGPT 대화(`https://chatgpt.com/share/6a3e68e3-ae88-83ee-b25a-bb547a31b4ef`) 내용 — Chrome 확장 미연결로 확인 못함. 재시도 또는 사용자가 직접 내용을 붙여넣어야 확인 가능.

## 승격된 지식 노트

- [WAS-vs-WebServer](../네트워크/WAS-vs-WebServer.md)
