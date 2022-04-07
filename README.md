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
 
@FeignClient(name="pay", url="http://localhost:8082")
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
 
 # CQRS
+ order 서비스(8081)와 orderList 서비스(8082)를 각각 실행

```
cd order
mvn spring-boot:run
```

```
cd orderList
mvn spring-boot:run
```

+ 맥도날드에 대한 order 요청

```sql
http localhost:8081/orders orderId=1 orderNum="상하이버거세트"
```

```sql
HTTP/1.1 201
Content-Type: application/json;charset=UTF-8
Date: Thu, 07 Apr 2022 05:41:22 GMT
Location: http://localhost:8081/orders/1
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "orderId": 1,
    "orderNum": "상하이버거세트",
}
```

+ 카프카 consumer 이벤트 모니터링

```
/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic shopmall --from-beginning
```

```sql
{"eventType":"Ordered","timestamp":"20220329041223","id":1,"orderId":1,"orderNum":"상하이버거세트","me":true}
{"eventType":"OrderConfirmed","timestamp":"20220329041223","id":1,"orderId":1,"orderId":1,"orderNum":"상하이버거세트","me":true}
```

+ orderView 서비스를 실행

```
cd orderView
mvn spring-boot:run

```

+ orderView의 Query Model을 통해 Order상태와 OrderConfirm상태를 `통합조회`

- Query Model 은 발생한 모든 이벤트를 수신하여 자신만의 `View`로 데이터를 통합 조회 가능하게 함

```
http localhost:8090/orderStatuses
```

```
HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 29 Mar 2022 04:13:00 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "orderStatuses": [
            {
                "_links": {
                    "orderStatus": {
                        "href": "http://localhost:8090/orderStatuses/1"
                    },
                    "self": {
                        "href": "http://localhost:8090/orderStatuses/1"
                    }
                },
                "orderListId": 1,
                "orderListStatus": "OrderConfirmed",
                "orderStatus": "Ordered",
                "orderId": 1,
                "orderNum": "상하이버거세트",
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8090/profile/orderStatuses"
        },
        "search": {
            "href": "http://localhost:8090/orderStatuses/search"
        },
        "self": {
            "href": "http://localhost:8090/orderStatuses{?page,size,sort}",
            "templated": true
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```

+ orderView 에서 order, orderList, order 상태를 통합 조회 가능함
+ Compensation Transaction 테스트(cancel order)
+ Order 취소

```
http DELETE localhost:8081/orders/1
```

```
HTTP/1.1 204
Date: Thu, 07 Apr 2022 05:54:44 GMT
```

+ order상태와 orderList상태 값을 확인

```
http localhost:8090/orderStatuses
```

```
HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8
Date: Thu, 07 Apr 2022 05:55:54 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "orderStatuses": [
            {
                "_links": {
                    "orderStatus": {
                        "href": "http://localhost:8090/orderStatuses/1"
                    },
                    "self": {
                        "href": "http://localhost:8090/orderStatuses/1"
                    }
                },
                "orderListId": 1,
                "orderListStatus": "OrderConfirmCancelled",
                "orderStatus": "OrderCancelled",
                "orderId": 1,
                "orderNum": "상하이버거세트",
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8090/profile/orderStatuses"
        },
        "search": {
            "href": "http://localhost:8090/orderStatuses/search"
        },
        "self": {
            "href": "http://localhost:8090/orderStatuses{?page,size,sort}",
            "templated": true
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```

+ order cancel 정보가 orderView에 전달되어 `orderStatus`, `orderListStatus` 모두 cancelled 로 상태 변경 된 것을 통합 조회 가능함
 

# Correlation / Compensation
## Correlation Id

+ Correlation Id를 생성하는 로직은 common-module로 구성하였다. 해당 로직은, 모든 컴포넌트에 동일하게 적용하고 컴포넌트 간의 통신은 Json 기반의 Http request를 받았을 때, Filter 에서 생성

```java

public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        CorrelationHttpHeaderHelper.prepareCorrelationParams(request);
        CorrelationLoggerUtil.updateCorrelation();
        filterChain.doFilter(request, response);
        CorrelationLoggerUtil.clear();
    }
 }
```

+ Filter에서는, 요청받은 request 를 확인하여, Correlation-Id가 존재할 경우, 해당 데이터를 식별자로 사용하고, 존재하지 않을 경우에는, 신규 Correlation Id를 생성한다. 관련 로직은 다음과 같다.

```java

public class CorrelationHttpHeaderHelper {

    public static void prepareCorrelationParams(HttpServletRequest httpServletRequest) {
        String currentCorrelationId = prepareCorrelationId(httpServletRequest);
        setCorrelations(httpServletRequest, currentCorrelationId);
        log.debug("Request Correlation Parameters : ");
        CorrelationHeaderField[] headerFields = CorrelationHeaderField.values();
        for (CorrelationHeaderField field : headerFields) {
            String value = CorrelationHeaderUtil.get(field);
            log.debug("{} : {}", field.getValue(), value);
        }
    }

    private static String prepareCorrelationId(HttpServletRequest httpServletRequest) {
        String currentCorrelationId = httpServletRequest.getHeader(CorrelationHeaderField.CORRELATION_ID.getValue());
        if (currentCorrelationId == null) {
            currentCorrelationId = CorrelationContext.generateId();
            log.trace("Generated Correlation Id: {}", currentCorrelationId);
        } else {
            log.trace("Incoming Correlation Id: {}", currentCorrelationId);
        }
        return currentCorrelationId;
    }
} 
```

## Compensation

+ `Correlation Id` 정보를 기반으로 kafka를 이용한 비동기방식의 Compensation Transaction 처리
```java

@Component
  public class MessageProducer {
    @Autowired
    private KafkaTemplate<String, Message> messageKafkaTemplate;

    @Value(value = "${message.topic.name}")
    private String messageTopicName;

    public void sendMessage(Message message) {
        ListenableFuture<SendResult<String, Message>> future = messageKafkaTemplate.send(messageTopicName, message);

        future.addCallback(new ListenableFutureCallback<SendResult<String, Message>>() {
            @Override
            public void onSuccess(SendResult<String, Message> result) {
                Message g = result.getProducerRecord().value();
                System.out.println("Sent message=[" + g.toString() + "] with offset=[" + result.getRecordMetadata().offset() + "]");
            }

            @Override
            public void onFailure(Throwable ex) {
                // needed to do compensation transaction.
                System.out.println( "Unable to send message=[" + message.toString() + "] due to : " + ex.getMessage());
            }
        });
    }
}
```

```java


@Component
public class MessageConsumer {

    @KafkaListener(topics = "${message.topic.name}", containerFactory = "messageKafkaListenerContainerFactory")
    public void messageListener(Message message, Acknowledgment ack) {
        try {
            System.out.println("----Received Message----");
            System.out.println("id: " + message.getName());
            System.out.println("act: " + message.getMsg());

            ack.acknowledge();
        } catch (Exception e) {
            // 에러 처리
        }
    }
}

```

```
// Producer Log
2022-04-07 06:36:21.665  INFO 15382 --- [nio-8081-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-04-07 06:36:21.665  INFO 15382 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-04-07 06:36:21.668  INFO 15382 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 3 ms
2022-04-07 06:36:07.604  INFO 15382 --- [nio-8081-exec-4] o.a.k.clients.producer.ProducerConfig    : ProducerConfig values: 
	...
022-04-07 06:37:07.625  INFO 15382 --- [nio-8081-exec-4] o.a.kafka.common.utils.AppInfoParser     : Kafka startTimeMs: 1648493227624
2022-04-07 06:37:07.689  INFO 15382 --- [ad | producer-1] org.apache.kafka.clients.Metadata        : [Producer clientId=producer-1] Cluster ID: PrON0srhTnuKsswe92XNA
Sent message=[test, 2022040711111] with offset=[10]

```

```
// Consumer Log
----Received Message----
id: 2022040711111
act: test
```
 


# Req & Res
* Feign Client

* `Interface 선언`을 통해 자동으로 Http Client 생성
* 선언적 Http Client란, Annotation만으로 Http Client를 만들수 있고, 이를 통해서 원격의 Http API호출이 가능
 
+ Dependency 추가

```java
    
    /** feign client*/
    <dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    ...

```

+ Controller
```java

@RestController
@RequiredArgsConstructor
public class MacDeliveryFeignController {

    private final MacDeliveryFeignService MacDeliveryFeignService;

    @GetMapping(value = "/v1/github/{owner}/{repo}")
    public List<Contributor> getMacDeliveryContributors(@PathVariable String owner , @PathVariable String repo){
        return MacDeliveryFeignService.getContributor(owner,repo);
    }
}

```

+ Service
```java
@Service
public class MacDeliveryFeignService {

  @Autowired
  private MacDeliveryFeignClient macDeliveryFeignClient;

  public List<Contributor> getContributor(String owner, String repo) {
    List<Contributor> contributors = macDeliveryFeignClient.getContributor(owner, repo);
    return contributors;
  }
}

```

+ FeignClient Interface
```java

@FeignClient(name="feign", url="https://api.github.com/repos",configuration = Config.class)
public interface MacDeliveryFeignClient {
    @RequestMapping(method = RequestMethod.GET , value = "/{owner}/{repo}/contributors")
    List<Contributor> getContributor(@PathVariable("owner") String owner, @PathVariable("repo") String repo);
}


```

+ DTO
```java

@Data
public class Contributor {
    String login;
    String id;
    String type;
    String site_admin;
}	
```
	
	
+ `@EnableFeignClients` Set
```java

@EnableFeignClients
@SpringBootApplication
public class ApiTestApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiTestApplication.class, args);
    }

}

```
