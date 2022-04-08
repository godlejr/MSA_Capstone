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
    "orderNum": "상하이버거세트",
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
Date: Thu, 07 Apr Mar 2022 05:19:00 GMT
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

# Compensation

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
 


# Req & Resp
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

# Gateway
+ gateway 및 virtualService 생성

	
```yaml
root@labs--970387545:/home/project/personal/istio-1.11.3# kubectl apply -f - << EOF
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: mac-delivery-order
> spec:
>   hosts:
>     - "*"
>   gateways:
>   - mac-delivery-order
>   http:
>   - match:
>     - uri:
>         prefix: /mac-delivery-order
>     route:
>     - destination:
>         host: mac-delivery-order
>         port:
>           number: 8080
> EOF
virtualservice.networking.istio.io/mac-delivery-order created
```
	
```yaml
root@labs--970387545:/home/project/personal/istio-1.11.3# kubectl apply -f - << EOF
> apiVersion: networking.istio.io/v1alpha3
> kind: Gateway
> metadata:
>   name: mac-delivery-order
> spec:
>   selector:
>     istio: ingressgateway # use istio default controller
>   servers:
>   - port:
>       number: 80
>       name: http
>       protocol: HTTP
>     hosts:
>     - "*"
> EOF
gateway.networking.istio.io/mac-delivery-order created
```

- 서비스 호출 및 VirtualService가 정상적으로 서비스 되고 있음을 확인

```sql
root@labs--970387545:/home/project/personal/istio-1.11.3# kubectl -n istio-system get service/istio-ingressgateway
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.100.110.183   adf3a4a5deebf44f780686b6433420ff-1731053437.ca-central-1.elb.amazonaws.com   15021:30340/TCP,80:30720/TCP,443:32149/TCP,31400:30068/TCP,15443:30860/TCP   5m57s
```

	
# Deploy / Pipeline
## Deploy
	
+ Kubernetes용 service.yaml, deployment.yaml 을 작성하고 Kubernetes에 Deploy

- service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
  labels:
    app: app
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: app
```

- deployment.yaml
	
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: 979050235289.dkr.ecr.ca-central-1.amazonaws.com/user16-app:v1
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
```



## PipeLine 
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD는 buildspec.yml을 이용한 AWS codebuild를 사용

- CodeBuild 프로젝트를 생성하고 AWS_ACCOUNT_ID, KUBE_URL, KUBE_TOKEN 환경 변수 세팅
#### CodeBuild 에서 EKS 연결

- Service Account 생성

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
EOF

```

-   ClusterRoleBinding 생성

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
EOF
```

- SA로 EKS 접속 토큰 가져오기

```sql
kubectl -n kube-system describe secret eks-admin
```
![image](https://user-images.githubusercontent.com/24773549/162341509-2249b567-6379-49ab-8621-d14b8e8c3cd8.png)
	
```yaml
#buildspec.yml 

version: 0.2

env:
  variables:
    _PROJECT_NAME1: "user16-app"
    _PROJECT_NAME2: "user16-customer"
    _PROJECT_NAME3: "user16-pay"
    _PROJECT_NAME4: "user16-store"
    _PROJECT_NAME1_1: "app"
    _PROJECT_NAME2_1: "customer"
    _PROJECT_NAME3_1: "pay"
    _PROJECT_NAME4_1: "store"

phases:
  install:
    runtime-versions:
      java: corretto8
      docker: 18
    commands:
      - echo install kubectl
      - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin/kubectl
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - echo $_PROJECT_NAME
      - echo $AWS_ACCOUNT_ID
      - echo $AWS_DEFAULT_REGION
      - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo start command
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...

      - cd uber/$_PROJECT_NAME1_1
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME1:$CODEBUILD_RESOLVED_SOURCE_VERSION  .

      - cd ../$_PROJECT_NAME2_1
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME2:$CODEBUILD_RESOLVED_SOURCE_VERSION  .

      - cd ../$_PROJECT_NAME3_1
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME3:$CODEBUILD_RESOLVED_SOURCE_VERSION  .

      - cd ../$_PROJECT_NAME4_1
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME4:$CODEBUILD_RESOLVED_SOURCE_VERSION  .

      - cd ../../
  post_build:
    commands:
      - echo Pushing the Docker image...

      - cd uber/$_PROJECT_NAME1_1
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME1:$CODEBUILD_RESOLVED_SOURCE_VERSION

      - cd ../$_PROJECT_NAME2_1
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME2:$CODEBUILD_RESOLVED_SOURCE_VERSION

      - cd ../$_PROJECT_NAME3_1
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME3:$CODEBUILD_RESOLVED_SOURCE_VERSION

      - cd ../$_PROJECT_NAME4_1
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME4:$CODEBUILD_RESOLVED_SOURCE_VERSION

      
      - cd ../../
      
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default

```
	
- codebuild 실행
- codebuild 프로젝트 및 빌드 이력
![image](https://user-images.githubusercontent.com/24773549/162342776-90c3ee58-993a-4193-9ee3-9dee863f7d7f.png)

- codebuild 빌드 내역 
![image](https://user-images.githubusercontent.com/24773549/162343187-888f5400-ab2c-492c-9976-61ef8bda4b98.png)

	
# Circuit Breaker

+ DestinationRule 생성

```
kubectl apply -f - << EOF
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: store
  spec:
    host: store
    trafficPolicy:
      outlierDetection:
        consecutive5xxErrors: 1
        interval: 1s
        baseEjectionTime: 3m
        maxEjectionPercent: 100
EOF
```

+ Circuit Breaker 테스트 환경 설정

```
kubectl scale deploy store --replicas=3
```
+ 새 터미널에서 Http Client 컨테이너를 설치하고, 접속한다.
```
kubectl create deploy siege --image=ghcr.io/acmexii/siege-nginx:latest
kubectl get pod -o wide
kubectl exec -it pod/siege-75d5587bf6-pqn2v -- /bin/bash
```
	
+ Circuit Breaker 동작 확인

```diff
+root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
- HTTP/1.1 200 OK
content-length: 39
content-type: text/plain;charset=UTF-8
Date: Fri, 08 Apr 2022 01:47:14 GMT
server: envoy
x-envoy-upstream-service-time: 215

store-67ff6476bb-ls9dw/192.168.33.74

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
Date: Fri, 08 Apr 2022 01:47:30 GMT
server: envoy
x-envoy-upstream-service-time: 16

store-67ff6476bb-6rzwc/192.168.82.163

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
Date: Fri, 08 Apr 2022 01:47:31 GMT
server: envoy
x-envoy-upstream-service-time: 25

store-67ff6476bb-sq452/192.168.12.147

```
+ 새로운 터미널에서 마지막에 출력된 store 컨테이너로 접속하여 명시적으로 5xx 오류를 발생 시킨다.
	
```diff
# 새로운 터미널 Open
# 3개 중 하나의 컨테이너에 접속
kubectl exec -it pod/store-67ff6476bb-ls9dw -c store -- /bin/sh
#
# httpie 설치 및 서비스 명시적 다운
apk update
apk add httpie
- http PUT http://localhost:8080/actuator/down
```
+ Siege로 접속한 이전 터미널에서 store 서비스로 접속해 3회 이상 호출해 본다.
```
http GET http://store:8080/actuator/health
```
+ 아래 URL을 통해 3개 중 `2개`의 컨테이너만 서비스 됨을 확인한다.
```diff
root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
date: Tue, 29 Mar 2022 05:28:15 GMT
server: envoy
x-envoy-upstream-service-time: 13

+ store-67ff6476bb-6rzwc/192.168.82.163

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
date: Tue, 29 Mar 2022 05:28:16 GMT
server: envoy
x-envoy-upstream-service-time: 7

- store-67ff6476bb-sq452/192.168.12.147

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
date: Tue, 29 Mar 2022 05:28:17 GMT
server: envoy
x-envoy-upstream-service-time: 12

+ store-67ff6476bb-6rzwc/192.168.82.163

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
date: Tue, 29 Mar 2022 05:28:19 GMT
server: envoy
x-envoy-upstream-service-time: 11

- store-67ff6476bb-sq452/192.168.12.147
```
+ Pool Ejection 타임(3’) 경과후엔 컨테이너 3개가 모두 동작됨이 확인된다.
```
root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 39
content-type: text/plain;charset=UTF-8
Date: Fri, 08 Apr 2022 01:47:14 GMT
server: envoy
x-envoy-upstream-service-time: 215

+ store-67ff6476bb-ls9dw/192.168.33.74

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
Date: Fri, 08 Apr 2022 01:47:28 GMT
server: envoy
x-envoy-upstream-service-time: 345

- store-67ff6476bb-6rzwc/192.168.82.163

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 40
content-type: text/plain;charset=UTF-8
Date: Fri, 08 Apr 2022 01:47:31 GMT
server: envoy
x-envoy-upstream-service-time: 311

! store-67ff6476bb-sq452/192.168.12.147

root@siege-75d5587bf6-pqn2v:/# http http://store:8080/actuator/echo
HTTP/1.1 200 OK
content-length: 39
content-type: text/plain;charset=UTF-8
Date: Fri, 08 Apr 2022 01:47:33 GMT
server: envoy
x-envoy-upstream-service-time: 10

+ store-67ff6476bb-ls9dw/192.168.33.74
```	

# Config Map / Persustemce Volume
	
+ PVC 생성

```yaml
kubectl apply -f - << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fs
  labels:
    app: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi
EOF
```
+ Secret 객체 생성
	
```yaml
kubectl apply -f - << EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  password: YWRtaW4=  
EOF
```

+ 해당 Secret을 mac-delivery (store) Deployment에 설정

![image](https://user-images.githubusercontent.com/24773549/162348649-e93461e5-09d7-42e9-8eda-9bd69650d035.png)
	

+ MySQL 설치
```yaml
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    name: lbl-k8s-mysql
spec:
  containers:
  - name: mysql
    image: mysql:latest
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-pass
          key: password
    ports:
    - name: mysql
      containerPort: 3306
      protocol: TCP
    volumeMounts:
    - name: k8s-mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: k8s-mysql-storage
    persistentVolumeClaim:
      claimName: "fs"
EOF

kubectl expose pod mysql --port=3306
```

+ Pod 에 접속하여 store orderdb 데이터베이스 공간을 만들어주고 데이터베이스가 잘 동작하는지 확인

```sql
# kubectl exec mysql -it -- bash
root@mysql:/# echo $MYSQL_ROOT_PASSWORD
admin
root@mysql:/# mysql --user=root --password=$MYSQL_ROOT_PASSWORD
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.28 MySQL Community Server - GPL

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database orderdb;
Query OK, 1 row affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| orderdb            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)
```

+ Pod 삭제 후 재생성하고 다시 db에 접속하여 `영속성` 확인
```yaml
kubectl delete pod/mysql

kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    name: lbl-k8s-mysql
spec:
  containers:
  - name: mysql
    image: mysql:latest
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-pass
          key: password
    ports:
    - name: mysql
      containerPort: 3306
      protocol: TCP
    volumeMounts:
    - name: k8s-mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: k8s-mysql-storage
    persistentVolumeClaim:
      claimName: "fs"
```
	
```sql
# kubectl exec mysql -it -- bash
root@mysql:/# echo $MYSQL_ROOT_PASSWORD
admin
root@mysql:/# mysql --user=root --password=$MYSQL_ROOT_PASSWORD
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.28 MySQL Community Server - GPL

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| orderdb            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)
```
	
# Polyglot
+ payment MicroService의 경우 Database 'HSQL' 적용

```maven
	<dependency>
	    <groupId>org.hsqldb</groupId>
	    <artifactId>hsqldb</artifactId>
	    <scope>runtime</scope>
	</dependency>
```

```
[INFO] >>> spring-boot-maven-plugin:2.1.9.RELEASE:run (default-cli) > test-compile @ payment >>>
Downloading from central: https://repo.maven.apache.org/maven2/org/hsqldb/hsqldb/2.4.1/hsqldb-2.4.1.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/hsqldb/hsqldb/2.4.1/hsqldb-2.4.1.pom (1.3 kB at 1.7 kB/s)
Downloading from central: https://repo.maven.apache.org/maven2/org/hsqldb/hsqldb/2.4.1/hsqldb-2.4.1.jar

	...
	
    drop table payment_table if exists Hibernate: 
    drop sequence hibernate_sequence if exists Hibernate: create sequence hibernate_sequence start with 1 increment by 1
Hibernate:     
    create table payment_table (
        pay_id bigint not null,
        order_id bigint,
        status varchar(255),
        primary key (pay_id)
    )
```

# Autoscaling
+ 기존 mac delivery store deploy에 Autoscale 설정 및 horizontalpodautoscaler, hpa 확인
	
```
kubectl autoscale deployment mac-delivery-store --cpu-percent=50 --min=1 --max=10
kubectl get horizontalpodautoscaler

NAME                 REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
mac-delivery-store   Deployment/mac-delivery-store   <unknown>/50%   1         10        0          1s
```

	
+ 테스트용 로드 제너레이터 확인 및 접속

```
kubectl get pod siege
kubectl exec -it pod/siege-75d5587bf6-pqn2v -- /bin/bash
```

+ 테스트용 로드 제너레이팅 전 상태

```
watch -d -n 1 kubectl get pod
```

![image](https://user-images.githubusercontent.com/24773549/162364365-e3ca821f-d8ec-48f2-a22a-95d767421194.png)


+ 테스트용 로드 제너레이팅 후 상태


```
siege -c30 -t30S -v http://mac-delivery-store	
watch -d -n 1 kubectl get pod
```

- 부하 완료

![image](https://user-images.githubusercontent.com/24773549/162364637-da890482-2227-4156-9060-40754963c770.png)

- Autoscaling 확인
- 각 pod당 CPU는 최대 0.5core 이며 50%(0.25core)의 임계치를 넘은 것을 확인 가능 + `siege` 부하 발생 후 Autoscaling 되어 pod 의 개수가 2개로 늘어나고 있는 것이 확인

![image](https://user-images.githubusercontent.com/24773549/162364603-80f4494e-7f00-4fa4-b4b9-cf5c7cd18501.png)


# Self Healing

+ `livenessProbe` 설정을 추가한 이미지 yaml 파일 작성

```yaml
#deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store
  labels:
    app: store
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store
  template:
    metadata:
      labels:
        app: store
    spec:
      containers:
        - name: store
          image: 979050235289.dkr.ecr.ca-central-1.amazonaws.com/user16-store:v1
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 15
            timeoutSeconds: 2
            successThreshold: 1
            periodSeconds: 5
            failureThreshold: 3
```

```
kubectl apply -f deployment.yaml
kubectl get svc
```

```
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes           ClusterIP   10.100.0.1       <none>        443/TCP    28h
store                ClusterIP   10.100.173.180   <none>        8080/TCP   1m
```

+ 해당 서비스의 health 확인

```
http 10.100.173.180:8080/actuator/health
```

	
```diff
HTTP/1.1 200
Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8
Date: Fri, 08 Apr 2022 04:58:49 GMT
Transfer-Encoding: chunked

{
+    "status": "UP"
}
```

+ 서비스 down

```
http put 10.100.173.180:8080/actuator/down
```

```diff
HTTP/1.1 200
Content-Type: application/json;charset=UTF-8
Date: Fri, 08 Apr 2022 04:59:30 GMT
Transfer-Encoding: chunked

{
-    "status": "DOWN"
}
```

+ 서비스 down 이후에도 여전히 pod가 Running 중임을 확인 및 세부 로그 확인(describe)

```
kubectl get pod
kubectl describe pod/mac-delivery-store-76d45569b4-6bvsz
```

```diff

+  Normal   Killing    107s (x2 over 4m12s)   kubelet, ip-192-168-67-172.ca-central-1.compute.internal  Container mac-delivery-store failed liveness probe, will be restarted
  Normal   Pulling    107s (x3 over 6m36s)   kubelet, ip-192-168-67-172.ca-central-1.compute.internal  Pulling image "979050235289.dkr.ecr.ca-central-1.amazonaws.com/user16-store:v1"
  Normal   Created    106s (x3 over 6m36s)   kubelet, ip-192-168-67-172.ca-central-1.compute.internal  Created container mac-delivery-store
  Normal   Started    106s (x3 over 6m36s)   kubelet, ip-192-168-67-172.ca-central-1.compute.internal  Started container mac-delivery-store
  Normal   Pulled     106s                   kubelet, ip-192-168-67-172.ca-central-1.compute.internal  Successfully pulled image "979050235289.dkr.ecr.ca-central-1.amazonaws.com/user16-store:v1" in 274.024538ms

```

+ livenessProbe가 5초에 한번씩 health check 수행하다가 unhealthy 발견 하여 `Warning` 발생
+ unhealthy 발견 즉시 `self-killing` & `healing` 수행한 것을 확인 가능

# Zero-downtime deploy(Readiness Probe)
+ 시간부족으로 못했으나 개념설명

- Liveness probe는 컨테이너의 상태가 비정상이라고 판단하면 해당 Pod를 재시작한다
- Readiness probe는 컨테이너가 비정상일 경우에는 해당 Pod를 사용 못 하게 하여 서비스등에서 제외한다. 가용성을 높이기 위함

```yaml
#예시 
readinessProbe:
  httpGet:
   path: /readiness
   port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

- HTTP Probe를 통해 HTTP GET으로 /readiness URL로 5초마다 호출을 해서 HTTP 응답을 200(정상)으로 인식하지만
- 해당 컨테이너를 정상으로 판단하고 200~300 범위를 벗어난 응답 코드를 받으면 `비정상으로 판단하여, 서비스 불가능한 상태로 인식해서 쿠버네티스 서비스에서 제외한다.`

