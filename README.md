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
+ Step.1<p>

*전반적인 어플리케이션의 구조 및 흐름을 인지한 상태에서 실시한 이벤트 스토밍과정으로, 기초적인 이벤트 도출이나, Aggregation 작업은 `Bounded Context`를 먼저 선정하고 진행*

*Pub/Sub 연결*

  
 ![image](https://user-images.githubusercontent.com/24773549/162122778-2c6f21f8-c3ef-4f9a-9f98-6aae30a45214.png)

+ Step.2<p>
 
 *완성본 대한 기능 검증*
 
![image](https://user-images.githubusercontent.com/24773549/162123202-dd3c9a48-289e-4dca-bd1f-e00554c8c79d.png)

'''
 - 기능요소
    - 고객이 메뉴를 선택하여 주문한다 (ok)
    - 고객이 결제한다 (ok)
    - 주문이 되면 주문 내역이 맥도날드 상점에게 전달된다 (ok)
    - 상점주인이 확인하여 햄버거 만들고 배달 출발한다 (ok)
    - 고객이 주문을 취소할 수 있다 (ok) 
    - 주문이 취소되면 배달이 취소 된다 (ok) 
    - 고객이 주문상태를 상시 조회 한다 (ok) 
 
 
 - 비기능요소
    - 마이크로 서비스를 넘나드느 시나리오에 대한 트랜잭션 처리 (OK)
    - 고객 결제처리 : 결제가 완료되지 않은 요청은 `ACID` 트랜잭션 적용(Request/Response 방식처리) (OK)
    - 결제가 완료되면 택시기사에게 배차 요청정보가 전달된다 (OK)
'''
 

# SAGA
+ 구현<p>
    서비스를 Local에서 아래와 같은 방법으로 서비스별로 개별적으로 실행한다.
   
```
    cd app
    mvn spring-boot:run
```
```
    cd pay
    mvn spring-boot:run 
```
```
    cd store
    mvn spring-boot:run  
```
```
    cd customer
    python policy-handler.py 
```
 
+ DDD적용<p>
    3개의 도메인으로 관리되고 있으며 `주문(Order)`, `결제(Pay)`, `주문관리(OrderList)`으로 구성된다.
 
 
 
```java
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String item;
    private Integer qty;
    private String status;
    private String macstore;
    private Long price;

    @PostPersist
    public void onPostPersist(){


        MacDelivery.external.OrderList OrderList = new MacDelivery.external.OrderList();

        OrderList.setOrderId(String.valueOf(getId()));
        if(getprice()!=null)
            OrderList.setPrice(Double.valueOf(getPrice()));

        Application.applicationContext.getBean(MacDelivery.external.OrderListService.class).pay(OrderList);


    } 
 
```
 
+ 서비스 호출흐름(Sync)<p>
`주문(Order)` -> `결제(Pay)`간 호출은 동기식으로 일관성을 유지하는 트랜젝션으로 처리
* 고객이 메뉴를 선택하여 주문 요청한다.
* 결제서비스를 호출하기위해 FeinClient를 이용하여 인터페이스(Proxy)를 구현한다.
* 주문 요청을 받은 직후(`@PostPersist`) 결제(Pay)를 요청하도록 처리한다.

 
```java 
 
@FeignClient(name="pay", url="http://localhost:8082")//, fallback = OrderListServiceFallback.class)
public interface OrderListService {

    @RequestMapping(method= RequestMethod.POST, path="/OrderLists")
    public void pay(@RequestBody OrderList OrderList);

 

```
  
+ 서비스 호출흐름(Async)<p>
* 결제가 완료되면 주문 수락시 주문 내용(메뉴, 가격, 정보 등) 맥도날드 상인에게 전달하는 행위는 비동기식으로 처리되, `주문 상태의 변경이 블로킹 되지 않도록 처리`
* 이를 위해 결제과정에서 기록을 남기고 승인정보를 `Kafka`로 전달한다.
   
 
```java

@Entity
@Table(name="Payment_table")
public class Payment {

 
   @PrePersist
    public void onPrePersist(){
      	 PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();
    }

}

```

* 주문관리(OrderList)에서는 결제 승인 Event를 수신해 PolicyHandler에서 후행 작업을 처리한다.
* 맥도날드 상인은 주문정보를 수락하고 요리를 하고 배달한다.

```java

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_ConfirmAllocation(@Payload PaymentApproved paymentApproved){

        if(!paymentApproved.validate()) return;

        System.out.println("주문 접수 완료  : " + paymentApproved.toJson() + "\n\n");
  
  }   
```

 
