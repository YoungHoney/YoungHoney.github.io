---
layout: post
title: 스프링 리액티브 프로그래밍을 배우며며
date: 2025-08-04
categories: [OOP,Spring, Reactive]
tags: [Java,Spring, Reactive]
---

# 1 들어가며

7월달부터 SSAFY 생활을 하면서 적응을 하느라 개인공부를 할 시간을 내지 못했다. 그러다가 잠깐 시간이 남아서 Java Spring WebFlux에 관한 공부를 했고, 간단히 기록하고자 한다. 

원래 목표는 Line의 [Armeria](https://armeria.dev/)라는 오픈소스에 기여하는 것이었는데, 이 프레임워크를 알아가기위해 이와 유사한 Spring WebFlux를 학습하기로 했다. 

예전 프로젝트를 진행하면서 구매한 
["스프링으로 시작하는 리액티브 프로그래밍"](https://product.kyobobook.co.kr/detail/S000201399476) 을 보고있다. 리액티브 프로그래밍이 뭔지, 리액티브 스트림즈의 인터페이스들을 보면서 생긴 궁금한 점들을 파헤쳐보았다.

## 1. 리액티브란?
리액티브라는 개념은 결국 "반응"을 얼마나 잘 해주는지에 중점을 둔 프로그래밍 개념이다. [리액티브 선언문](https://www.reactivemanifesto.org/ko)을 참고하자면,

- MEANS(방법) : 비동기 메시지 I/O 를 사용.
- FORMS(형태) : 시스템이 유연성있고 탄력성있게 동작하는 형태
- VALUE(값, 가치) : 시스템이 사용자의 요구를 문제없이 수용하고 응답하는 것

이라고 이해 할 수 있다. 

그리고 이러한 리액티브 시스템을 구축하기 위해 필요한 프로그래밍 모델이 바로 **리액티브 프로그래밍** 이다. 리액티브 프로그래밍의 특징은 다음과 같다.
1. 선언형 프로그래밍 방식이다. 기존의 for문을 사용하거나, if문을 사용하는 명령형 프로그래밍 방식과 달리, Java의 람다식을 활용한 Stream과 같이 메서드 체인으로 구성되어있다.

2. Data Stream, propagation of change, 데이터가 지속적으로 발생하며, 데이터를 계속 전달하는 것을 의미한다.

## 2. 리액티브 스트림즈와 역압력

개발자가 리액티브한 코드를 작성하기위해 정의해 놓은 **표준 사양** 이 바로 **리액티브 스트림즈** 이다. 리액티브 스트림즈의 구성요소는 다음과 같다.

- Publisher : 데이터를 생성하고, 통지하는 역할 (방송국, 유튜브 채널로 이해)
- Subscriber : 구독한 Publisher로부터 통지된 데이터를 전달받고 처리. (방송 수신자, 구독자로 이해)
- Subscription : publisher에 요청할 데이터의 개수를 지정하고, 구독을 취소한다.
- Processor : Publisher와 Subscriber의 기능을 모두 가지고 있다. Subscriber로서 다른 Publisher를 구독할 수 있고, Publisher로서 다른 Subscriber가 구독할 수 있다.

내가 궁금했던 부분은 이 부분인데, Publisher의 인터페이스가 다소 신기하게 생겼기 때문이다.

```java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```

보통 pub/sub관계를 이해할 때 **구독자.구독(X튜브 채널)** 의 형태로 정의되는게 평범하다고 이해할 수 있으나. 반대로 **X튜브 채널.구독(구독자)** 의 형태로 되어있다.  

이 어색해 보이는 설계의 핵심에는 리액티브 스트림즈의 가장 중요한 가치인 **'역압력(Backpressure)'** 이라는 개념이 있다.

### 역압력(Backpressure)이 뭐길래?

간단한 비유를 들어보자. 초밥 뷔페의 회전 레일(Publisher) 위에 요리사가(Publisher가 생성하는 데이터) 쉴 새 없이 초밥 접시를 올려놓고, 손님(Subscriber)은 레일 위의 접시를 가져다 먹는다. 만약 요리사가 손님이 먹는 속도와 상관없이 무한정 빠르게 초밥을 올린다면 어떻게 될까? 레일 위에는 접시가 가득 쌓이다 못해 바닥으로 떨어지고, 손님은 결국 제대로 식사를 할 수 없을 것이다.

이때 손님이 요리사에게 "잠깐만! 지금 3개만 더 먹을 수 있으니, 3개만 보내줘!"라고 소리쳐서 생산 속도를 제어할 수 있다면 어떨까? 이것이 바로 **역압력**이다.

> **역압력(Backpressure)** 이란, 데이터 소비자(`Subscriber`)가 데이터 생산자(`Publisher`)에게 자신이 처리할 수 있는 데이터의 양을 요청하여 데이터 흐름을 제어하는 메커니즘이다. 이를 통해 `Publisher`가 너무 빠른 속도로 데이터를 발행하여 `Subscriber`가 과부하에 걸리거나 시스템 장애(`OutOfMemoryError` 등)가 발생하는 것을 막을 수 있다.

### 역압력과 `Publisher.subscribe()` 설계의 관계

이제 다시 처음의 질문으로 돌아와 보자. 왜 **`X튜브 채널.구독(구독자)`** 의 형태일까? 바로 이 역압력 메커니즘을 가장 효율적으로 구현하기 위한 설계이기 때문이다. 둘의 '구독' 과정, 즉 최초의 '악수(Handshake)' 과정을 단계별로 살펴보면 그 이유가 명확해진다.

```
  Subscriber         |      Subscription        |      Publisher
       |             |                          |          |
       |             |                          |          |
       | <--- 1. subscribe(this) 호출됨 -------- |          |  (외부에서 publisher.subscribe(subscriber) 실행)
       |             |                          |          |
       |             |                          | -- 2. Subscription 생성 --> |
       |             |                          |          |
       | -- 3. onSubscribe(subscription) 호출 -->|          |
       |             |                          |          |
       |         (Subscriber가 '리모컨'을 전달받음)           |
       |             |                          |          |
       |             | <--- 4. request(n) ----- |          |  (데이터 n개 요청)                                 |          |
       |             |                          |          |
       |             |                          |          |  (요청받은 개수만큼 데이터 생성)                               
       |             |                          |          |
       | <--- 5. onNext(data) ------------------|          |  (데이터 전달)                                     |
       |             |                          |          |
       | <--- 5. onNext(data) ------------------|          |  (... n번 반복 ...)
       |             |                          |          |
       | <--- 6. onComplete() / onError() ------|          |  (스트림 종료)
       |             |                          |          |

```

1.  **`publisher.subscribe(subscriber)` 호출**
    * `Subscriber`가 `Publisher`에게 "이제부터 당신의 데이터를 받겠다"라고 자신을 등록하는 과정이다.

2.  **`subscriber.onSubscribe(subscription)` 호출**
    * 이게 바로 핵심이다! `Publisher`는 `subscribe` 요청을 받자마자 데이터를 바로 보내는 것이 아니라, **`Subscriber`의 `onSubscribe` 메서드를 호출하면서 `Subscription` 객체를 전달**한다.
    * 이 `Subscription` 객체는 `Subscriber`가 `Publisher`를 제어할 수 있는 '리모컨'과도 같다.

3.  **`subscription.request(n)` 호출**
    * '리모컨'을 전달받은 `Subscriber`는 이제 준비가 되었을 때, `subscription.request(n)`을 통해 "저는 n개의 데이터를 받을 준비가 됐으니 보내주세요." 라고 능동적으로 요청한다.

4.  **`publisher.onNext(data)` 호출**
    * `Publisher`는 `request(n)` 요청을 받은 만큼만 데이터를 생성하여 `Subscriber`의 `onNext()` 메서드를 통해 전달한다. n개의 데이터를 모두 보내거나, 더 이상 보낼 데이터가 없으면 `onComplete()`를, 에러가 발생하면 `onError()`를 호출하여 스트림을 종료한다.

이 과정을 보면, 구독의 시작은 `Publisher`에 요청했지만 **데이터 흐름의 제어권(리모컨)은 `Subscriber`에게 위임**되는 '제어의 역전(Inversion of Control)'이 일어나는 것을 볼 수 있다. 이 구조 덕분에 `Subscriber`는 자신의 처리 능력에 맞춰 데이터 흐름을 완벽하게 조절할 수 있게 된다.

### 다른 대안은 없었을까?

물론 다른 설계도 고려해볼 수 있다. 하지만 왜 현재의 방식이 채택되었는지 그 한계점을 통해 알아보자.

* **대안 1: `Subscriber.subscribe(Publisher)`**
    * 직관적이지만, `Subscriber`가 `Publisher`에 등록한 후 제어권을 가진 '리모컨(`Subscription`)'을 돌려받는 과정이 매끄럽지 않다. `Publisher`의 등록 메서드가 `Subscription`을 반환하고, `Subscriber`가 그걸 다시 자신의 상태로 저장하는 등 불필요하게 복잡한 코드가 만들어진다. `onSubscribe` 콜백 방식이 이 과정을 훨씬 명확하고 우아하게 만든다.

* **대안 2: Kafka 같은 인메모리 메시지 브로커(이벤트 버스) 모델**
    * `Publisher`와 `Subscriber`가 서로를 전혀 모르는 '느슨한 결합' 구조이다. 이는 분산 시스템 간 통신에는 유리하지만, 리액티브 스트림즈의 목표인 **'1:1 스트림의 정교한 흐름 제어'** 에는 적합하지 않다. 특정 `Subscriber`가 처리하지 못한다고 해서 `Publisher`의 발행 속도를 늦추는 등의 역압력 메커니즘을 구현할 수 없기 때문이다.

결론적으로, 리액티브 스트림즈의 `Publisher.subscribe(Subscriber)` 설계는 언어적 직관성을 약간 포기하는 대신, 비동기 환경에서 시스템을 안정적으로 지켜주는 **역압력(Backpressure)을 가장 확실하게 구현하기 위한 최적의 선택**이었던 셈이다. 이 설계를 이해하는 것이 리액티브 프로그래밍을 제대로 시작하는 첫걸음이라 할 수 있다.
