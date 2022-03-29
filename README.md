H Taxi
============
카카오택시 따라잡기
-----
<img src = "https://t1.kakaocdn.net/kakaomobility/company_website/contents/v2/10-taxi-sub-4.jpg" width = "700">

# 평가항목
  * 분석설계
  * SAGA
  * CQRS
  * Correlation / Compensation
  * Req / Resp
  * Gateway
  * Deploy / Pipeline
  * Circuit Breaker
  * Autoscale(HPA)
  * Self-healing(Liveness Probe)
  * Zero-downtime deploy(Readiness Probe)
  * Config Map / Persustemce Volume
  * Polyglot
   
----

# 분석설계
  + Step1<p>
    *전반적인 어플리케이션의 구조 및 흐름을 인지한 상태에서 실시한 이벤트 스토밍과정으로, 기초적인 이벤트 도출이나, Aggregation 작업은 `Bounded Context`를 먼저 선정하고 진행*
    <img src = '/images/Screen Shot 2022-03-28 at 14.42.26.png'>

  + Step2<p>
    *Pub/Sub연결*
    <img src = '/images/Screen Shot 2022-03-28 at 15.18.42.png'>

  + Step3<p>
   *완성본 대한 기능 검증*
    <img src = '/images/Screen Shot 2022-03-28 at 15.30.42.png'>

    ```
    - 기능요소
      - 사용자가 배차를 `요청`한다 (OK)
      - 사용자가 `결제`한다 (OK)
      - 결제가 완료되면 택시기사에게 `배차` 요청정보가 전달된다 (OK)
      - 택시기사가 배차를 확정하면 서비스가 시작되고 배차상태가 변경된다 (OK)
    - 비기능요소
      - 마이크로 서비스를 넘나드느 시나리오에 대한 트랜잭션 처리 (OK)
      - 고객 결제처리 : 결제가 완료되지 않은 요청은 `ACID` 트랜잭션 적용(Request/Response 방식처리) (OK)
      - 결제가 완료되면 택시기사에게 배차 요청정보가 전달된다 (OK)
    ```

# SAGA
    + 구현
    서비스를 Local에서 아래와 같은 방법으로 서비스별로 개별적으로 실행한다.
    ```
    cd app
    mvn spring-boot:run

    cd pay
    mvn spring-boot:run 

    cd store
    mvn spring-boot:run  

    cd customer
    python policy-handler.py 

    ```
     
