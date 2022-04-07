# MSA_Capstone
MSA 캡스톤


맥도날드 주문 앱 따라잡기
============
-----

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

*전반적인 어플리케이션의 구조 및 흐름을 인지한 상태에서 실시한 이벤트 스토밍과정으로, 기초적인 이벤트 도출이나, Aggregation 작업은 `Bounded Context`를 먼저 선정하고 진행*
*Pub/Sub 연결*

  
 ![image](https://user-images.githubusercontent.com/24773549/162122778-2c6f21f8-c3ef-4f9a-9f98-6aae30a45214.png)

