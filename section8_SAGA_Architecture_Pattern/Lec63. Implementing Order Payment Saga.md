### Lec63. Implementing Order Payment Saga

> 최대한 강의 원문 스크립트 해석을 직역하되 필요한 부분에 대해 이해를 빠르게 하기 위한 부분은 나름의 의역 및 요약 작업을 수행함.

<br>



안녕하세요 여러분.

이번 강의에서는 주문 서비스(Order Service)를 업데이트하고 payment, approval 이라는 Saga Step 들을 추가하겠습니다.<br>

먼저 infrastructure 모듈로 이동하여 saga라는 새 모듈을 만듭니다. 이 infrastructure 모듈에는 두 개의 SAGA Step 인터페이스를 넣을 것입니다. pom.xml에서는  `common-domain` 종속성을 허용하겠습니다.<br>
왜냐하면 `common-domain`의 도메인 이벤트 인터페이스를 사용해야 하기 때문입니다.<br>

<br>



**SagaStep 인터페이스 정의**<br>

그런 다음 com.food.ordering.system 패키지를 saga 모듈 내의 /src/main/java 에 만듭니다. 그리고 `SagaStep` 라는 이름으로 `interface` 를 추가합니다.

```java
package com.food.ordering.system.saga;

import com.food.ordering.system.domain.event.DomainEvent;

public interface SagaStep<T, S extends DomainEvent, U extends DomainEvent> {
    S process(T data);
    U rollback(T data);
}
```

여기서는 세 가지 제너릭 변수 T, `S extends DomainEvent` `U extends DomainEvent` 를 사용하도록 해주었습니다.

그리고 두 가지 메서드를 만듭니다.

- process (T data) : S
  - T 타입의 파라미터 data 를 받아서 DomainEvent 타입 S 를 리턴합니다.
- rollback(T data): U
  - 롤백 메서드 입니다.
  - T 타입의 파라미터 data 를 받아서 DomainEvent 타입 U를 리턴합니다.

<br>



 이 인터페이스는 각 Saga 단계로 구현되며 `process(T data)` 메서드는 트랜잭션으로 표준 처리를 처리하고 `rollback(T data)` 메서드는 다음 Saga 단계에서 오류가 발생할 경우 보상 트랜잭션을 처리합니다.

핵심 아이디어는 `다음 단계(Stage)가 실패한다면, 이전 단계가 변경사항을 롤백할 수 있어야 한다` 입니다. 



**Void 타입의 반환**<br>

SagaStep 의 메서드 들인 porcess(T data), rollback(T data) 를 자세히 보면 각 SAGA 단계는 T 타입을 처리하고 DomainEvent 를 반환한다. 그런데 일부 SAGA Step 에서는 종료 작업일 경우 이벤트를 실행할 필요가 없는 경우가 있다. 



이런 케이스를 처리하기 위해 common-domain 모듈의 event 패키지에 `EmptyEvent`를 생성하겠습니다.

`:common:common-domain/src/main/java/com.food.ordering.system.event.EmptyEvent.java`

```java
package com.food.ordering.system.domain.event;

public final class EmptyEvent implements DomainEvent<Void> {

    public static final EmptyEvent INSTANCE = new EmptyEvent();

    private EmptyEvent() {
    }

    @Override
    public void fire() {

    }
}
```



여기서는 `<Void>` 제너릭 타입으로 DomainEvent 를 구현하겠습니다. 그리고  IDE에 의해 오버라이딩 된 fire 메서드는 빈 블록으로 둡니다.

`public static final EmptyEvent INSTANCE = new EmptyEvent();`

여기에서는 이 클래스에 대한 싱글턴 인스턴스를 전역 인스턴스 상수로 생성하겠습니다. 그리고 `EmptyEvent` 클래스를 `final` 로 선언합니다.

또한 여기서는 private 생성자를 허용하겠습니다. 왜냐하면 이 EmptyEvent 클래스는 단지 마커 클래스일 뿐이고 다른 클래스 간에 동일한 인스턴스를 공유해도 괜찮기 때문입니다.

<br>



**pom.xml**

이제 프로젝트의 최상위에 있는 pom.xml 파일에 saga 종속성을 추가해준다.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.food.ordering.system</groupId>
      <artifactId>saga</artifactId>
      <version>${project.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

<br>



이번에는 `:order-service:order-domain:order-application-service/pom.xml` 을 열고 버전을 정의없이 saga 모듈 종속성을 추가한다.

```xml 
<dependencies>
  <dependency>
    <groupId>com.food.ordering.system</groupId>
    <artifactId>saga</artifactId>
  </dependency>
</dependencies>
```

<br>



**SAGA Step 을 왜 `:order-service:order-domain:order-application-service` 모듈 내에 정의할까?**

좋습니다. 이제 `:order-service:order-domain:order-application-service` 모듈 내에 첫번째 SAGA Step 을 만들어보겠습니다. 

왜 `:order-service:order-domain:order-application-service` 에서 첫번째 SAGA Step 을 생성할까요?

왜냐하면 Order Service 를 SAGA 흐름의 coordinator(코디네이터로) 사용하고 있기도 하고, 모든 SAGA Step 을 Order Service 내에 유지해두고 싶기 때문입니다. (Order Service가 SAGA 의 지휘자처럼 사용되는 역할이기에 가급적 SagaStep 관련 구현들을 Order Service 내에 모아두고 싶다는 이야기.)

<br>



**OrderPaymentSaga 구현**

그래서 `:order-service:order-domain:order-application-service` 모듈 내에  OrderPaymentSaga 라는 새 클래스를 만들겠습니다. @Slf4j, @Component 어노테이션을 추가해줬습니다. @Component 어노테이션을 사용함으로써 `OrderPaymentSaga` 를 스프링 매니지드 빈으로 선언했습니다.

결제 레스토랑 메시지 리스너 입력 클래스는 입력 부분 결제 응답 메시지 리스너를 구현한다는 점을 기억하세요. 그리고 주문 메시징 모듈에서는 결제 응답 Kafka 리스너 클래스의 결제 응답 메시지 리스너 인터페이스를 사용합니다.

이 클래스에는 Kafka 리스너 주석이 포함된 수신 메서드가 있으므로 실제로 Kafka 소비자를 생성 및 사용하고 결제 응답 Kafka 주제에서 데이터를 가져온 다음 이 결제 응답 메시지 리스너 인터페이스를 사용하여 모든 복제 서비스로 보냅니다.

그런 다음 결제 응답 메시지 리스너 입력 클래스에서 주문 결제 사가를 사용하여 작업을 처리하거나 롤백합니다.
먼저 주문 결제 사가 구현을 마치고 여기로 돌아와 결제 응답 메시지 리스너 입력 클래스를 수행하고 주문 결제 사가 개체를 사용하여 사가 작업을 처리하거나 롤백하겠습니다.

이제 주문 결제 Saga 클래스에서 매개변수 T를 대체하는 데이터 유형에 대한 Saga 단계 인터페이스를 구현하겠습니다.

이 단계는 결제 서비스에서 응답을 받은 후 호출되므로 결제 응답을 사용하겠습니다.

결제 응답 모델을 데이터 유형으로 설정한 후, 결제가 완료되면 주문 서비스에서 결제 완료 이벤트를 받고 주문 결제 사가는 다음 단계인 레스토랑을 진행해야 하기 때문에 처리 방법의 반환 유형을 주문 결제로 설정하겠습니다. 승인.

레스토랑 승인 요청은 주문 결제 이벤트와 함께 실행되어야 한다는 점을 기억하세요. 그래서 여기서는 처리 방식의 반환 유형을 주문 결제 이벤트로 설정했습니다.

롤백 방법의 경우 결제가 실패하면 서비스 로컬 데이터베이스 작업을 주문하기 위해 롤백하면 되기 때문에 빈 이벤트만 설정하고 이전 단계가 없기 때문에 법의 이야기가 중지됩니다. 주문 결제 단계.

좋습니다. 모두 이러한 일반 유형을 사용하여 프로세스 및 롤백 메서드를 작성해 보겠습니다. 또한 데이터베이스 트랜잭션을 생성하고 이러한 메서드의 변경 사항을 커밋하려고 하기 때문에 두 메서드 모두에 트랜잭션 주석을 넣었습니다.

그리고 앞서 언급한 대로 결제 응답 메시지 리스너 입력이 될 도메인 이벤트를 호출자에게 반환하면 간단히 이벤트가 시작됩니다.

이 경우 로컬 데이터베이스 트랜잭션 변경 사항이 이미 커밋됩니다.

### 강의 스크립트

Hi everyone.

In this lecture I will update the order service and add payment and approval saga steps.

First, I will go to infrastructure module and create a new module called saga.

I will put two saga step interface in this infrastructure module.
In the palm exam I'll file off this module, I will let common domain dependency.
Because I will need to use domain event interface from common domain.


Then I will create calm food ordering system saga package, and then create saga step interface.

Here I will let three generic variables T, S extends domain event, and U extends domain event.

And I will create two methods.
First method is process methods which parameter T as the data and process method returns S, which is a domain event.

And the second method is rollback methods.

Again, with T as a parameter and this time I will return U, which is again a domain event. This interface will be implemented with each saga step and process methods will handle the standard processing with a transaction, and roll-back methods will handle the compensating transaction in case a failure occurs in the next saga step.

The idea is that, if the next saga step fails previous one should be able to roll-back its changes.
As you see, each saga step will process a type T and it returns a domain event.

In some saga steps, I will not need to fire an event if it is an ending operation. To handle these cases, I will go and create an empty event in event package in the common domain module.

Here I will simply implement domain events with void generic type. And override the fire methods with an empty quote block.

Here I will create a singleton instance of this class with a global instance constant, and make this class final.

Also I will let a private constructor here, because this empty event is just a marker class and sharing the same instance among different classes is okay.

Now to use saga module as a dependency let's add the saga dependency in the base palm exam file with project version.

Then I will open order application service palm exam file and add saga dependency here without version.

Great, now let's create the first saga step in order application service module.

Why do I create this saga step here in the order application service?
Because I use the order service as the coordinator of Saga ~~law~~  flow and I want to keep all saga steps here in the order service.

So I will create a new class here called Order Payment saga. I will let slf4j annotation to use in logging and @component annotation, so this class will be a sprint managed pin.

Remember that, payment restaurant message listener input class implements the input part payment response message listener. And in order messaging module, I use the payment response message listener interface in the payment response Kafka listener class.

This class has a received methods with Kafka listener annotation, so it actually creates and uses a Kafka consumer and gets the data from payment response Kafka topic, and then send it to all the replication service using this payment response message listener interface.

Then in the payment response message listener input class I will just use the order payment saga to process or roll back to operation.
So let's first finish the order payment saga implementation and I will come back here to do payment response message listener input class and use the order payment saga object to process or roll-back saga operation.

Now in the order payment saga class I will implement saga step interface for the data type which is the replacement for parameter T.

I will use payment response because this step will be called after getting a response from payment service.

After setting payment response model as a data type I will set the return type of process methods as order payment, because when a payment is completed order service will get a payment completed event and the order payment saga should proceed to do next step which is restaurant approval.

Remember that restaurant approval request should be triggered with a order paid event. So that's why I set the return type of the process method as order paid event here.

For the rollback method, I will just set the empty events because if the payment is not successful, I just need to roll back to order service local database operations and the saga of law will stop since I don't have any previous step before the order payment step.

Great, let's all write the process and rollback methods with these generic types. I also put transactional annotation in both methods because I want to create a database transaction and commit the changes in these methods.

And when I return the domain events to the caller which will be the payment response message listener input as I mentioned, I will simply fire the events.

As the local database transaction changes will be already committed in that case.



Now I will create three final fields here:

order domain service, order repository, and order paid restaurant request message publisher.

Let's inject these fields using constructor injection.

Then in the process methods first, I will log an info message.
I will rename the parameter as payment response here, and I will get to order from database using order ID.

For this, I will create and call a private method here find order, with string order ID parameter. I will return an order object.

Here I will need to find by ID methods of order repository and currently I only have find by tracking ID methods as you see, so I will let find by ID methods which order ID parameter to the order repository.

And in the implementation class I will override and implement it.
I will simply call the JPA repository find by ID methods.
And map the response JPA and the object to order domain object using the Mapper class.

Now I will return to the order payment saga and call the find by ID methods.

As the parameter, I will create order ID and use UUID from string methods to create the order ID UUID parameter.

This will return an optional order. If this optional object is empty I will log an error message and throw an order not found exception.

Otherwise, I will simply get the order from the optional object and return. After getting to order object in process methods

I will call order domain service pay order methods and pass the order and order paid message publishers as parameters. This will return an order paid advance and I will update to order with the changes into database using all the repository save methods.

Final, I will log info message and return the order paid domain event.
In the roll-back methods, I will log info message and find to order using the same find order methods that I just created.

Then I will call order domain service, cancel order methods with order and failure messages parameters. As you see this cancel order methods has void return type so I don't get a domain event as a response.

That is fine because with this roll-back methods I don't need an event as there is no previous saga step to be triggered. And because of that, here I have an empty event.

Then I will save the order with changes into database, and log an info message, and then return to empty event here.

Now I will return to payment response message listener input class and inject and use the order payment cycle object.

Let's do that in the next lecture.