# 의류대여

# Table of contents

- [의류대여](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

의류 대여 커버하기 - https://1sung.tistory.com/106

기능적 요구사항
1. 고객이 의류를 선택하여 주문한다
1. 주문이 되면 주문 내역이 의류 배송팀에게 전달된다
1. 의류 배송팀이 확인하여 의류를 배송한다
1. 고객이 의류를 반납한다
1. 반납 접수되면 반납 내역이 의류 배송팀에게 전달된다
1. 배송 전에만 고객이 주문을 취소할 수 있다
1. 주문이 취소되면 배송이 취소된다
1. 고객이 주문상태를 중간중간 조회한다
1. 주문상태가 바뀔 때 마다 메일로 알림을 보낸다

비기능적 요구사항
1. 트랜잭션
    1. 고객의 주문 취소는 반드시 배송팀 배송취소가 전제되어야 한다  Sync 호출 
1. 장애격리
    1. 배송관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 배송이 과중되면 배송을 잠시동안 받지 않고 잠시후에 배송 처리 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 마이페이지에서 주문 및 배송 상태를 확인할 수 있어야 한다  CQRS
    1. 주문상태가 바뀔때마다 메일 등으로 알림을 줄 수 있어야 한다  Event driven



# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계



## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/byIxok3KxwPmNS0LTBaRBxIaWot2/mine/da2c98d4d6fc151ee88572a6ad82c909


### 완성된 모형

![image](https://user-images.githubusercontent.com/66341540/100956395-b763d000-355b-11eb-946d-1157cf5ba00e.png)

 
### 비기능 요구사항에 대한 검증

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 고객 취소시 배송처리:  배송이 취소되지 않은 주문은 취소되지 않는다는 경영자의 오랜 신념(?) 에 따라, ACID 트랜잭션 적용. 주문취소 전 배송취소 처리에 대해서는 Request-Response 방식 처리
        - 나머지 모든 inter-microservice 트랜잭션: 주문, 회수 등 모든 이벤트와 같이 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.



# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd order
mvn spring-boot:run

cd delivery
mvn spring-boot:run 

cd customercenter
mvn spring-boot:run  

cd gateway
mvn spring-boot:run 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 order 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

```
package clothrental;

import javax.persistence.*;

import com.esotericsoftware.kryo.util.IntArray;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Objects;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String productId;
    private Integer qty;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Order order = new Order();
        order.setStatus(order.getStatus());
        System.out.println("##### Status : " + order.getStatus());

        if (Objects.equals(status, "Order")) {

            Ordered ordered = new Ordered();
            BeanUtils.copyProperties(this, ordered);
            ordered.publishAfterCommit();
        }
        if (Objects.equals(status, "Return")){

            Returned returned = new Returned();
            BeanUtils.copyProperties(this, returned);
            returned.publishAfterCommit();
        }

    }

    @PostUpdate
    public void onPostUpdate(){
        System.out.println("################# Order Status Updated and Update Event raised..!!");
        OrdereCancelled ordereCancelled = new OrdereCancelled();
        BeanUtils.copyProperties(this, ordereCancelled);
        ordereCancelled.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        clothrental.external.Cancellation cancellation = new clothrental.external.Cancellation();
        // mappings goes here
        // 아래 this는 Order 어그리게이트
        cancellation.setOrderId(this.getId());
        cancellation.setStatus("Delivery Cancelled");
        OrderApplication.applicationContext.getBean(clothrental.external.CancellationService.class)
                .cancelship(cancellation);

    }

    @PreRemove
    public void onPreRemove(){
        OrdereCancelled ordereCancelled = new OrdereCancelled();
        BeanUtils.copyProperties(this, ordereCancelled);
        ordereCancelled.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        clothrental.external.Cancellation cancellation = new clothrental.external.Cancellation();
        // mappings goes here
        // 아래 this는 Order 어그리게이트
        cancellation.setOrderId(this.getId());
        cancellation.setStatus("Delivery Cancelled");
        OrderApplication.applicationContext.getBean(clothrental.external.CancellationService.class)
            .cancelship(cancellation);


    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getProductId() {
        return productId;
    }

    public void setProductId(String productId) {
        this.productId = productId;
    }
    public Integer getQty() {
        return qty;
    }

    public void setQty(Integer qty) {
        this.qty = qty;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터소스 유형에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package clothrental;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{
}
```
- 적용 후 REST API 의 테스트
```
# order 서비스의 주문처리
http http://order:8080/orders productId=1001 qty=5 status=Order

# delivery 서비스의 배송취소처리
http http://delivery:8080/cancellations orderId=1 status="Delivery Cancelled"

# 주문 상태 확인
http http://20.194.37.221:8080/mypages
http http://customercenter:8080/mypages

```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(order)->배송취소(cancellation) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 배송서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (order) CancellationService.java

package clothrental.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="delivery", url="${api.delivery.url}")
public interface CancellationService {

    @RequestMapping(method= RequestMethod.POST, path="/cancellations")
    public void cancelship(@RequestBody Cancellation cancellation);

}
```

- 배송 취소가 되면(@PostUpdate) 주문 취소가 가능하도록 처리
```
# Order.java (Entity)

    @PostUpdate
    public void onPostUpdate(){
        System.out.println("################# Order Status Updated and Update Event raised..!!");
        OrdereCancelled ordereCancelled = new OrdereCancelled();
        BeanUtils.copyProperties(this, ordereCancelled);
        ordereCancelled.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        clothrental.external.Cancellation cancellation = new clothrental.external.Cancellation();
        // mappings goes here
        // 아래 this는 Order 어그리게이트
        cancellation.setOrderId(this.getId());
        cancellation.setStatus("Delivery Cancelled");
        OrderApplication.applicationContext.getBean(clothrental.external.CancellationService.class)
                .cancelship(cancellation);

    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 배송 (delivery) 서비스를 잠시 내려놓음 (ctrl+c)

#주문취소처리
http PATCH http://order:8080/orders/2 status="Delivery Cancelled"   #Fail

#배송서비스 재기동
cd delivery
mvn spring-boot:run

#주문취소처리
http PATCH http://order:8080/orders/2 status="Delivery Cancelled"   #Success

```

![배송서비스stop한상태](https://user-images.githubusercontent.com/66341540/105003606-35c9f200-5a76-11eb-8e06-5595b19d4c38.JPG)


![배송서비스active한상태](https://user-images.githubusercontent.com/66341540/105001947-dff44a80-5a73-11eb-9140-fc8aa9725aaa.JPG)


- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


주문이 이루어진 후에 배송시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 배송 시스템의 처리를 위하여 주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 주문이력에 기록을 남긴 후에 곧바로 배송이 시작되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package clothrental;

import javax.persistence.*;

import com.esotericsoftware.kryo.util.IntArray;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Objects;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String productId;
    private Integer qty;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Order order = new Order();
        order.setStatus(order.getStatus());
        System.out.println("##### Status : " + order.getStatus());

        if (Objects.equals(status, "Order")) {

            Ordered ordered = new Ordered();
            BeanUtils.copyProperties(this, ordered);
            ordered.publishAfterCommit();
        }
        if (Objects.equals(status, "Return")){

            Returned returned = new Returned();
            BeanUtils.copyProperties(this, returned);
            returned.publishAfterCommit();
        }

    }
```

- 배송 서비스에서는 배송 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package clothrental;

...

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    DeliveryRepository deliveryRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrdered_Ship(@Payload Ordered ordered){

        if(ordered.isMe()){
            // To-Do : SMS발송, CJ Logistics 연계, ...
            Delivery delivery = new Delivery();
            delivery.setOrderId(ordered.getId());
            delivery.setStatus("Delivery Started");

            deliveryRepository.save(delivery);

            System.out.println("##### listener Ship : " + ordered.toJson());
        }
    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrdered_Ship(@Payload Returned returned){

        if(returned.isMe()){
            // To-Do : SMS발송, CJ Logistics 연계, ...
            Delivery delivery = new Delivery();
            delivery.setOrderId(returned.getId());
            delivery.setStatus("Return Started");

            deliveryRepository.save(delivery);

            System.out.println("##### listener Ship : " + returned.toJson());
        }
    }

```
배송은 주문과 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 배송 시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
  
```
package clothrental;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Delivery_table")
public class Delivery {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Shipped shipped = new Shipped();
        BeanUtils.copyProperties(this, shipped);
        shipped.publishAfterCommit();


    }

```
```
# 배송 서비스 (Delivery) 를 잠시 내려놓음 (ctrl+c)

#주문처리
http http://order:8080/orders productId=1001 qty=5 status=Order   #Success

#배송목록 확인
http http://delivery:8080/deliveries    # 배송 목록이 조회안됨

#배송 서비스 기동
cd delivery
mvn spring-boot:run

#배송목록 확인
http http://delivery:8080/deliveries     # 모든 주문의 목록이 조회됨
```

```
고객센터는 주문/배송과 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 고객센터 시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
  
```
package clothrental;

import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface MypageRepository extends CrudRepository<Mypage, Long> {

    List<Mypage> findByOrderId(Long orderId);

}

```
# 고객센터 서비스 (customercenter) 를 잠시 내려놓음 (ctrl+c)

#주문처리
http http://order:8080/orders productId=1001 qty=5 status=Order   #Success

#주문목록 확인
http http://customercenter:8080/mypages    # 모든 주문의 목록이 조회안됨

#고객센터 서비스 기동
cd customercenter
mvn spring-boot:run

#주문목록 확인
http http://customercenter:8080/mypages     # 모든 주문의 목록이 조회됨
```


# 운영

## CI/CD 설정



## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 주문(order)-->배송취소(delivery) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

feign:
  hystrix:
    enabled: true

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(배송:cancellation) 의 임의 부하 처리 - 500 밀리에서 증감 220 밀리 정도 왔다갔다 하게, Thread.currentThread().sleep((long) (500 + Math.random() * 220));
```
# (delivery) cancellation.java (Entity)

    @PrePersist
    public void onPrePersist(){
        System.out.println("################# cancellation start");

        try {
            Thread.currentThread().sleep((long) (500 + Math.random() * 220));
            // Thread.currentThread().sleep((long) (800 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 2명
- 5초 동안 실시

```
$ siege -c2 -t5S -v --content-type "application/json" 'http://order:8080/orders/1 PATCH {"status": "Delivery Cancelled"}'
:
:

```
![서킷브레이크결과](https://user-images.githubusercontent.com/66341540/105006247-bb9b6c80-5a79-11eb-9c29-f2b4827269d7.JPG)

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌.


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy order --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 1분 동안 걸어준다.
```
siege -c150 -t60S -v --content-type "application/json" 'http://order:8080/orders/1 PATCH {"status": "Delivery Cancelled"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다: kubectl get hpa 에 대한 캡처가 필요
```
kubectl get deploy order -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![오토스케일결과](https://user-images.githubusercontent.com/66341540/105006359-e5549380-5a79-11eb-8de7-1e2a85d1e2fe.JPG)

## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

delivery 에 deployment.yaml readiness probe 없는 상태

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c2 -t60S -v --content-type "application/json" 'http://delivery:8080/cancellations POST {"productId": "1001", "qty":5}'

```

- 새버전으로의 배포 시작
```
kubectl set image deployment/delivery delivery=clothrentalt.azurecr.io/delivery:latest --record
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

![무정지배포_리드니스없는상태](https://user-images.githubusercontent.com/66341540/105007085-e4703180-5a7a-11eb-83c9-1369166503fb.JPG)


배포기간중 Availability 가 평소 100%에서 2.75% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:

![무정지배포_리드니스있는상태](https://user-images.githubusercontent.com/66341540/105007195-049ff080-5a7b-11eb-9795-33fe618dab86.JPG)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.






# 신규 개발 조직의 추가

  ![image](https://user-images.githubusercontent.com/487999/79684133-1d6c4300-826a-11ea-94a2-602e61814ebf.png)


## 마케팅팀의 추가
    - KPI: 신규 고객의 유입률 증대와 기존 고객의 충성도 향상
    - 구현계획 마이크로 서비스: 기존 customer 마이크로 서비스를 인수하며, 고객에 음식 및 맛집 추천 서비스 등을 제공할 예정

## 이벤트 스토밍 
    ![image](https://user-images.githubusercontent.com/487999/79685356-2b729180-8273-11ea-9361-a434065f2249.png)


## 헥사고날 아키텍처 변화 

![image](https://user-images.githubusercontent.com/487999/79685243-1d704100-8272-11ea-8ef6-f4869c509996.png)

## 구현  

기존의 마이크로 서비스에 수정을 발생시키지 않도록 Inbund 요청을 REST 가 아닌 Event 를 Subscribe 하는 방식으로 구현. 기존 마이크로 서비스에 대하여 아키텍처나 기존 마이크로 서비스들의 데이터베이스 구조와 관계없이 추가됨. 

## 운영과 Retirement

Request/Response 방식으로 구현하지 않았기 때문에 서비스가 더이상 불필요해져도 Deployment 에서 제거되면 기존 마이크로 서비스에 어떤 영향도 주지 않음.

* [비교] 결제 (pay) 마이크로서비스의 경우 API 변화나 Retire 시에 app(주문) 마이크로 서비스의 변경을 초래함:

예) API 변화시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

                --> 

        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제2(pay);

    }
```

예) Retire 시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        /**
        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

        **/
    }
```





![image](https://user-images.githubusercontent.com/66341540/100953402-e37c5280-3555-11eb-9c05-d7009e60936f.png)
