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
    3개의 도메인으로 관리되고 있으며 `배차요청(Grab)`, `결제(Payment)`, `배차할당(Allocation)`으로 구성된다.
 
```diff
    
    @Document
    @Table(name="Grab_table")
    public class Grab  {

        @Id
        @GeneratedValue(strategy=GenerationType.AUTO)
        private Long id;
        private Integer grabStatus;
        private String phoneNumber;
        private String startingPoint;
        private String destination;
        private Integer estimatedFee;

+       @PostPersist
        public void onPostPersist(){

            //배차요청
            GrabRequestConfirmed grabRequestConfirmed = new GrabRequestConfirmed();
+           BeanUtils.copyProperties(this, grabRequestConfirmed);
            grabRequestConfirmed.publishAfterCommit();
+           htaxi.external.Payment payment = new htaxi.external.Payment();
            payment.setId(getid());

            GrabApplication.applicationContext.getBean(htaxi.external.PaymentService.class).pay(payment);
            grabCancelled.publishAfterCommit();
        }
```
   
+ 서비스 호출흐름(Sync)<p>
`배차요청(Grab)` -> `결제(Pay)`간 호출은 동기식으로 일관성을 유지하는 트랜젝션으로 처리
* 고객이 목적지를 설정하고 택시 배차를 요청한다.
* 결제서비스를 호출하기위해 FeinClient를 이용하여 인터페이스(Proxy)를 구현한다.
* 배차요청을 받은 직후(`@PostPersist`) 결제를 요청하도록 처리한다.
```
// PaymentService.java

package htaxi.external;

import ...

@FeignClient(name="Payment", url="http://localhost:8080")
public interface PaymentService {
    @RequestMapping(method= RequestMethod.GET, path="/payments")
    public void pay(@RequestBody Payment payment);

}   
```
   
+ 서비스 호출흐름(Async)<p>
* 결제가 완료되면 배차할당시 배차요청내용(승차장소, 목적지, 고객정보등) 택시기사에게 전달하는 행위는 비동기식으로 처리되, `배차할당 상태의 변경이 블로킹 되지 않도록 처리`
* 이를 위해 결제과정에서 기록을 남기고 승인정보를 `Kafka`로 전달한다.
   
```diff
package htaxi;

@Entity
@Table(name="Payment_table")
public class Payment {

...
+   @PrePersist
    public void onPrePersist(){
     	PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
+       paymentApproved.publishAfterCommit();
    }

}

```

* 배차할당관리(Allocation)에서는 결제 승인 Event를 수신해 PolicyHandler에서 후행 작업을 처리한다.
* 택시기사는 수신된 배차정보를 수락하고 승차장소로 이동한다.

```java
package htaxi;

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_ConfirmAllocation(@Payload PaymentApproved paymentApproved){

        if(!paymentApproved.validate()) return;

        System.out.println("\n\n##### 배차할당 받음 : " + paymentApproved.toJson() + "\n\n");
  
  }   
```

 
# CQRS
+ grab 서비스(8081)와 allocate 서비스(8082)를 각각 실행

```
cd grab
mvn spring-boot:run
```

```
cd allocate
mvn spring-boot:run
```

+ taxi에 대한 grab 요청

```sql
http localhost:8081/grabs taxiId=1 taxiNum="서울32저4703"
```

```sql
HTTP/1.1 201
Content-Type: application/json;charset=UTF-8
Date: Tue, 29 Mar 2022 04:12:23 GMT
Location: http://localhost:8081/grabs/1
Transfer-Encoding: chunked

{
    "_links": {
        "grab": {
            "href": "http://localhost:8081/grabs/1"
        },
        "self": {
            "href": "http://localhost:8081/grabs/1"
        }
    },
    "taxiId": 1,
    "taxiNum": "서울32저4703",
}
```

+ 카프카 consumer 이벤트 모니터링

```
/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic shopmall --from-beginning
```

```sql
{"eventType":"Grabbed","timestamp":"20220329041223","id":1,"taxiId":1,"taxiNum":"서울32저4703","me":true}
{"eventType":"Allocated","timestamp":"20220329041223","id":1,"grabId":1,"taxiId":1,"taxiNum":"서울32저4703","me":true}
```

+ grabView 서비스를 실행

```
cd grabView
mvn spring-boot:run

```

+ grabView의 Query Model을 통해 Grab상태와 Allocate상태를 `통합조회`

- Query Model 은 발생한 모든 이벤트를 수신하여 자신만의 `View`로 데이터를 통합 조회 가능하게 함

```
http localhost:8090/grabStatuses
```

```sql
HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 29 Mar 2022 04:13:00 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "grabStatuses": [
            {
                "_links": {
                    "grabStatus": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    },
                    "self": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    }
                },
                "allocateId": 1,
                "allocateStatus": "Allocated",
                "grabStatus": "Grabbed",
                "taxiId": 1,
                "taxiNum": "서울32저4703",
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8090/profile/grabStatuses"
        },
        "search": {
            "href": "http://localhost:8090/grabStatuses/search"
        },
        "self": {
            "href": "http://localhost:8090/grabStatuses{?page,size,sort}",
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

+ grabView 에서 grab, allocate, taxi 상태를 통합 조회 가능함
+ Compensation Transaction 테스트(cancel grab)
+ Taxi Grab 취소

```
http DELETE localhost:8081/grabs/1
```

```sql
HTTP/1.1 204
Date: Tue, 29 Mar 2022 04:13:27 GMT
```

+ grab상태와 allocate상태 값을 확인

```
http localhost:8090/grabStatuses
```

```diff
HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 29 Mar 2022 04:13:36 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "grabStatuses": [
            {
                "_links": {
                    "grabStatus": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    },
                    "self": {
                        "href": "http://localhost:8090/grabStatuses/1"
                    }
                },
                "allocateId": 1,
+                "allocateStatus": "AllocateCancelled",
+                "grabStatus": "GrabCancelled",
                "taxiId": 1,
                "taxiNum": "서울32저4703",
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8090/profile/grabStatuses"
        },
        "search": {
            "href": "http://localhost:8090/grabStatuses/search"
        },
        "self": {
            "href": "http://localhost:8090/grabStatuses{?page,size,sort}",
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

+ grab cancel 정보가 grabView에 전달되어 `grabStatus`, `allocateStatus` 모두 cancelled 로 상태 변경 된 것을 통합 조회 가능함
 
 
# Correlation / Compensation
## Correlation Id

+ Correlation Id를 생성하는 로직은 common-module로 구성하였다. 해당 로직은, 모든 컴포넌트에 동일하게 적용하고 컴포넌트 간의 통신은 Json 기반의 Http request를 받았을 때, Filter 에서 생성
```diff
@Slf4j
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        CorrelationHttpHeaderHelper.prepareCorrelationParams(request);
+       CorrelationLoggerUtil.updateCorrelation();
        filterChain.doFilter(request, response);
        CorrelationLoggerUtil.clear();
    }
 }
```

+ Filter에서는, 요청받은 request 를 확인하여, Correlation-Id가 존재할 경우, 해당 데이터를 식별자로 사용하고, 존재하지 않을 경우에는, 신규 Correlation Id를 생성한다. 관련 로직은 다음과 같다.
```diff
@Slf4j
public class CorrelationHttpHeaderHelper {

    public static void prepareCorrelationParams(HttpServletRequest httpServletRequest) {
        String currentCorrelationId = prepareCorrelationId(httpServletRequest);
+       setCorrelations(httpServletRequest, currentCorrelationId);
        log.debug("Request Correlation Parameters : ");
        CorrelationHeaderField[] headerFields = CorrelationHeaderField.values();
        for (CorrelationHeaderField field : headerFields) {
            String value = CorrelationHeaderUtil.get(field);
            log.debug("{} : {}", field.getValue(), value);
        }
    }

    private static String prepareCorrelationId(HttpServletRequest httpServletRequest) {
+        String currentCorrelationId = httpServletRequest.getHeader(CorrelationHeaderField.CORRELATION_ID.getValue());
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
```diff
package com.example.kafkapub.publish;

import ...

@Component
+ public class GreetingProducer {
    @Autowired
    private KafkaTemplate<String, Greeting> greetingKafkaTemplate;

    @Value(value = "${greeting.topic.name}")
    private String greetingTopicName;

    public void sendMessage(Greeting greeting) {
        ListenableFuture<SendResult<String, Greeting>> future = greetingKafkaTemplate.send(greetingTopicName, greeting);

        future.addCallback(new ListenableFutureCallback<SendResult<String, Greeting>>() {
            @Override
            public void onSuccess(SendResult<String, Greeting> result) {
                Greeting g = result.getProducerRecord().value();
                System.out.println("Sent message=[" + g.toString() + "] with offset=[" + result.getRecordMetadata().offset() + "]");
            }

            @Override
            public void onFailure(Throwable ex) {
                // needed to do compensation transaction.
                System.out.println( "Unable to send message=[" + greeting.toString() + "] due to : " + ex.getMessage());
            }
        });
    }
}
```

```diff
package com.example.kafkasub.consume;

import ...

@Component
+ public class GreetingConsumer {

    @KafkaListener(topics = "${greeting.topic.name}", containerFactory = "greetingKafkaListenerContainerFactory")
+    public void greetingListener(Greeting greeting, Acknowledgment ack) {
        try {
            System.out.println("----Received Message----");
            System.out.println("id: " + greeting.getName());
            System.out.println("act: " + greeting.getMsg());

            ack.acknowledge();
        } catch (Exception e) {
            // 에러 처리
        }
    }
}

```

```diff
// Producer Log
2022-03-29 03:46:21.665  INFO 15252 --- [nio-8081-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-03-29 03:46:21.665  INFO 15252 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-03-29 03:46:21.668  INFO 15252 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 3 ms
2022-03-29 03:47:07.604  INFO 15252 --- [nio-8081-exec-4] o.a.k.clients.producer.ProducerConfig    : ProducerConfig values: 
	...
    
2022-03-29 03:47:07.625  INFO 15252 --- [nio-8081-exec-4] o.a.kafka.common.utils.AppInfoParser     : Kafka version: 2.3.1
2022-03-29 03:47:07.625  INFO 15252 --- [nio-8081-exec-4] o.a.kafka.common.utils.AppInfoParser     : Kafka commitId: 18a913733fb71c01
2022-03-29 03:47:07.625  INFO 15252 --- [nio-8081-exec-4] o.a.kafka.common.utils.AppInfoParser     : Kafka startTimeMs: 1648493227624
2022-03-29 03:47:07.689  INFO 15252 --- [ad | producer-1] org.apache.kafka.clients.Metadata        : [Producer clientId=producer-1] Cluster ID: PrON0srhTnuKFQX6k4LXNA
+ Sent message=[refund, 202203290347-189237!] with offset=[10]

```

```diff
// Consumer Log
----Received Message----
+ id: 202203290347-189237
+ act: refund
```

<img src = '/images/Screen Shot 2022-03-29 at 4.00.37.png'>


# Req / Resp (feign client)

* `Interface 선언`을 통해 자동으로 Http Client 생성
* 선언적 Http Client란, Annotation만으로 Http Client를 만들수 있고, 이를 통해서 원격의 Http API호출이 가능
 
+ Dependency 추가
```diff
dependencies {
    ...
    
    /** feign client*/
+    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
+    implementation group: 'io.github.openfeign', name: 'feign-gson', version: '11.0'

    /** spring web*/
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'junit:junit:4.13.1'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
    annotationProcessor 'org.projectlombok:lombok'
    
    ...
}
```

+ Controller
```diff
package com.example.feigntest.controller;

import ...

@Slf4j
@RestController
@RequiredArgsConstructor
public class HTaxiFeignController {

    private final HTaxiFeignService HTaxiFeignService;

+   @GetMapping(value = "/v1/github/{owner}/{repo}")
    public List<Contributor> getHTaxiContributors(@PathVariable String owner , @PathVariable String repo){
        return HTaxiFeignService.getContributor(owner,repo);
    }
}

```

+ Service
```diff
package com.example.feigntest.service;

import ...

@Slf4j
@Service
public class HTaxiFeignService {

  @Autowired
  private HTaxiFeignClient hTaxiFeignClient;

  public List<Contributor> getContributor(String owner, String repo) {
    List<Contributor> contributors = hTaxiFeignClient.getContributor(owner, repo);
    return contributors;
  }
}

```

+ FeignClient Interface
```diff
package com.example.feigntest.client;

import ...

- @FeignClient(name="feign", url="https://api.github.com/repos",configuration = Config.class)
public interface HTaxiFeignClient {
    @RequestMapping(method = RequestMethod.GET , value = "/{owner}/{repo}/contributors")
    List<Contributor> getContributor(@PathVariable("owner") String owner, @PathVariable("repo") String repo);
}


```

+ DTO
```java
package com.example.feigntest.dto;

import lombok.Data;

@Data
public class Contributor {
    String login;
    String id;
    String type;
    String site_admin;
}	
```
	
	
+ `@EnableFeignClients` Set
```diff
package com.example;

import ...
- @EnableFeignClients
@SpringBootApplication
public class ApiTestApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiTestApplication.class, args);
    }

}

```

+ Run 
<img src = '/images/Screen Shot 2022-03-29 at 0.54.37.png' width="900px">


	
	

	
	
	
	
	
	
	
## Autoscaling

+ Autoscaling 테스트를 위한 k8s pod 생성

```yaml
# h-taxi-grab.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grab
spec:
  selector:
    matchLabels:
      run: h-taxi-grab
  replicas: 1
  template:
    metadata:
      labels:
        run: h-taxi-grab
    spec:
      containers:
        - name: h-taxi-grab
          image: devgony/h-taxi-grab:stable
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: h-taxi-grab
  labels:
    run: h-taxi-grab
spec:
  ports:
    - port: 80
  selector:
    run: h-taxi-grab
```

```
kubectl apply -f h-taxi-grab.yaml
kubectl get all
```

```
NAME                               READY   STATUS    RESTARTS   AGE
pod/h-taxi-grab-75f877867b-xz5cd   1/1     Running   0          8s

NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/h-taxi-grab   ClusterIP   10.40.4.25   <none>        80/TCP    8s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/h-taxi-grab   1/1     1            1           8s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/h-taxi-grab-75f877867b   1         1         1       8s
```

+ Autoscale 설정 및 horizontalpodautoscaler, hpa 확인

```
kubectl autoscale deployment h-taxi-grab --cpu-percent=50 --min=1 --max=10
kubectl get horizontalpodautoscaler
```

```
NAME          REFERENCE                TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
h-taxi-grab   Deployment/h-taxi-grab   <unknown>/50%   1         10        0          14s
```

```
kubectl get hpa
```

```
NAME          REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
h-taxi-grab   Deployment/h-taxi-grab   0%/50%    1         10        1          68s
```

+ 로드 제너레이터 설치

```
kubectl apply -f siege.yaml
kubectl get pod siege
```

```
NAME    READY   STATUS    RESTARTS   AGE
siege   1/1     Running   0          2m16s
```

+ `siege` pod 내에서 부하 발생

```
kubectl exec -it siege -- /bin/bash
siege> siege -c30 -t30S -v http://h-taxi-grab
```

5. `h-taxi-grab` pod 에 대한 모니터링 수행

- `siege` 부하 발생 전 pod 상태

```
watch -d -n 1 kubectl get pod
```

```
Every 1.0s: kubectl get pod                                    labs-1676586095: Mon Mar 28 04:32:58 2022

NAME                           READY   STATUS    RESTARTS   AGE
h-taxi-grab-75f877867b-xz5cd   1/1     Running   0          10m
siege                          1/1     Running   0          6m20s
```

+ `siege` 부하 발생 후 grapana CPU usage 모니터링
  ![image](https://user-images.githubusercontent.com/51254761/160510381-c00a076e-e5f7-4476-ab45-d28d57a58a0b.png)
  - 각 pod당 CPU는 최대 0.5core 이며 50%(0.25core)의 임계치를 넘은 것을 확인 가능
+ `siege` 부하 발생 후 pod 상태 확인

```
Every 1.0s: kubectl get pod                                    labs-1676586095: Mon Mar 28 04:35:02 2022

NAME                           READY   STATUS              RESTARTS   AGE
h-taxi-grab-75f877867b-gw5sn   1/1     Running             0          8s
h-taxi-grab-75f877867b-lj5lv   0/1     ContainerCreating   0          8s
h-taxi-grab-75f877867b-xz5cd   1/1     Running             0          12m
h-taxi-grab-75f877867b-zpv46   0/1     ContainerCreating   0          8s
siege                          1/1     Running             0          8m24s
```

+ Autoscaling 되어 pod 의 개수가 4개로 늘어나고 있는 것이 확인 됌

	
## Zero-downtime deploy

+ `h-taxi-grab` deploy

```yaml
# deploy-h-taxi-grab.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grab
  labels:
    app: h-taxi-grab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grab
  template:
    metadata:
      labels:
        app: h-taxi-grab
    spec:
      containers:
        - name: h-taxi-grab
          image: devgony/h-taxi-grab:stable
          ports:
            - containerPort: 8080
```

```
kubectl apply -f deploy-h-taxi-grab.yaml
```

+ `h-taxi-grab` service

```yaml
# service-h-taxi-grab.yaml
apiVersion: "v1"
kind: "Service"
metadata:
  name: "h-taxi-grab"
  labels:
    app: "h-taxi-grab"
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: "h-taxi-grab"
  type: "ClusterIP"
```

```
kubectl apply -f service-h-taxi-grab.yaml
```

+ 부하테스트 `siege` pod 설치

```yaml
# siege.yaml
apiVersion: v1
kind: Pod
metadata:
  name: siege
spec:
  containers:
    - name: siege
      image: apexacme/siege-nginx
```

+ `siege` 내부에서 부하 수행

```
kubectl exec -it siege -- /bin/bash
siege> siege -c1 -t60S -v http://h-taxi-grab:8080/grab --delay=1S
```

+ `stable` -> `canary` 버전으로 수정 후 재 배포

```diff
# deploy-h-taxi-grab.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grab
  labels:
    app: h-taxi-grab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grab
  template:
    metadata:
      labels:
        app: h-taxi-grab
    spec:
      containers:
        - name: h-taxi-grab
-         image: devgony/h-taxi-grab:stable
+         image: devgony/h-taxi-grab:canary
...
```

```
kubectl apply -f deploy-h-taxi-grab.yaml
```

+ `siege` 결과 일부만 성공(80.35%)하고 나머지는 배포시 중단 된 것을 확인

```diff
siege>
Lifting the server siege...
Transactions:                   2368 hits
-Availability:                  80.35 %
Elapsed time:                  59.49 secs
Data transferred:               0.80 MB
Response time:                  0.02 secs
Transaction rate:              39.81 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                    0.79
Successful transactions:        2368
Failed transactions:             579
Longest transaction:            0.75
Shortest transaction:           0.00
```

+ readinessProbe 추가, `canary` -> `stable` 버전 변경

```diff
# deploy-h-taxi-grab.yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grab
  template:
    metadata:
      labels:
        app: h-taxi-grab
    spec:
      containers:
        - name: h-taxi-grab
-         image: devgony/h-taxi-grab:canary
+         image: devgony/h-taxi-grab:stable
          ports:
            - containerPort: 8080
+         readinessProbe:
+           httpGet:
+             path: "/grab"
+             port: 8080
+           initialDelaySeconds: 10
+           timeoutSeconds: 2
+           periodSeconds: 5
+           failureThreshold: 10
```

+ 부하 재 발생 및 무중단 배포 테스트

```
siege> siege -c1 -t60S -v http://h-taxi-grab:8080/grab --delay=1S
```

```
kubectl apply -f deploy-h-taxi-grab.yaml
```

```diff
[error] socket: unable to connect sock.c:249: Connection reset by peer
...
siege>
Lifting the server siege...
Transactions:                   2928 hits
+Availability:                  99.83 %
Elapsed time:                  59.05 secs
Data transferred:               0.99 MB
Response time:                  0.00 secs
Transaction rate:              49.59 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                    0.23
Successful transactions:        2928
Failed transactions:               5
Longest transaction:            0.04
Shortest transaction:           0.00
```

+ readinessProbe 설정을 통해 99.83%에 달하는 Availability를 보여주는 것을 확인 가능
+ 100%에 달하는 무중단 배포 이지만 부하 테스트의 delay가 짧아 socket에러 일부 발생하여 0.17% 낮아진 수치

	
## Self Healing

+ `livenessProbe` 설정을 추가한 이미지 yaml 파일 작성

```yaml
# h-taxi-grab-liveness.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grab
  labels:
    app: h-taxi-grab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grab
  template:
    metadata:
      labels:
        app: h-taxi-grab
    spec:
      containers:
        - name: h-taxi-grab
          image: devgony/h-taxi-grab-liveness:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: "/actuator/health"
              port: 8080
            initialDelaySeconds: 15
            timeoutSeconds: 2
            successThreshold: 1
            periodSeconds: 5
            failureThreshold: 3
```

+ yaml 파일 적용 후 LoadBalancer 타입으로 배포

```
kubectl apply -f h-taxi-grab-liveness.yaml
kubectl expose deploy h-taxi-grab --type=LoadBalancer --port=8080
kubectl get svc
```

```
NAME          TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
h-taxi-grab   LoadBalancer   10.40.2.145   <pending>     8080:31282/TCP   6ss
```

+ 해당 서비스의 health 확인

```
http 10.40.2.145:8080/actuator/health
```

```diff
HTTP/1.1 200
Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8
Date: Mon, 28 Mar 2022 06:53:49 GMT
Transfer-Encoding: chunked

{
+    "status": "UP"
}
```

+ 서비스 down

```
http put 10.40.2.145:8080/actuator/down
```

```diff
HTTP/1.1 200
Content-Type: application/json;charset=UTF-8
Date: Mon, 28 Mar 2022 06:54:30 GMT
Transfer-Encoding: chunked

{
-    "status": "DOWN"
}
```

+ 서비스 down 이후에도 여전히 pod가 Running 중임을 확인

```
kubectl get pod
```

```
NAME                          READY   STATUS    RESTARTS   AGE
h-taxi-grab-95cb5c959-cq4th   1/1     Running   1          2m51s
```

+ describe 통해 해당 pod 세부 로그 확인

```
kubectl describe pod/h-taxi-grab-95cb5c959-cq4th
```

```diff
...
Name:         h-taxi-grab-95cb5c959-cq4th
Namespace:    labs-1676586095
Priority:     0
Node:         gke-cluster-2-default-pool-a1811fce-bfcl/10.146.15.213
Start Time:   Mon, 28 Mar 2022 06:52:10 +0000
Labels:       app=h-taxi-grab
              pod-template-hash=95cb5c959
Annotations:  <none>
Status:       Running
IP:           10.36.5.6
IPs:
IP:           10.36.5.6
Controlled By:  ReplicaSet/h-taxi-grab-95cb5c959
Containers:
  h-taxi-grab:
    Container ID:   docker://0f59d6c23a68a81817b9e7d81fee3b0656c7c64c32b08db2429c37d18848dc1a
    Image:          devgony/h-taxi-grab-liveness:latest
    Image ID:       docker-pullable://devgony/h-taxi-grab-liveness@sha256:76c7e59956ad1411c08de4fb50df44ab72c01053eaf96646491eda9797dd8734
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 28 Mar 2022 06:54:48 +0000
    Last State:     Terminated
      Reason:       Error
      Exit Code:    143
      Started:      Mon, 28 Mar 2022 06:52:19 +0000
      Finished:     Mon, 28 Mar 2022 06:54:45 +0000
    Ready:          True
    Restart Count:  1
    Liveness:       http-get http://:8080/actuator/health delay=15s timeout=2s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8nhg5 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-8nhg5:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute for 300s
                             node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                  From                                               Message
  ----     ------     ----                 ----                                               -------
  Normal   Scheduled  <unknown>                                                               Successfully assigned labs-1676586095/h-taxi-grab-95cb5c959-cq4th to gke-cluster-2-default-pool-a1811fce-bfcl
  Normal   Pulled     3m27s                kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Successfully pulled image "devgony/h-taxi-grab-liveness:latest" in 5.954773255s
  Normal   Pulling    59s (x2 over 3m33s)  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Pulling image "devgony/h-taxi-grab-liveness:latest"
-  Warning  Unhealthy  59s (x3 over 69s)    kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Liveness probe failed: HTTP probe failed with statuscode: 503
-  Normal   Killing    59s                  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Container h-taxi-grab failed liveness probe, will be restarted
+  Normal   Created    56s (x2 over 3m25s)  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Created container h-taxi-grab
+  Normal   Started    56s (x2 over 3m25s)  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Started container h-taxi-grab
+  Normal   Pulled     56s                  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Successfully pulled image "devgony/h-taxi-grab-liveness:latest" in 2.704500252s
```

+ livenessProbe가 5초에 한번씩 health check 수행하다가 unhealthy 발견 하여 `Warning` 발생
+ unhealthy 발견 즉시 `self-killing` & `healing` 수행한 것을 확인 가능
