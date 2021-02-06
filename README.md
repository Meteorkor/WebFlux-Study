# WebFlux-Kotlin-Study

Why was Spring WebFlux created?
* 적은 스레드 적은 리소스를 통한 병렬처리
  * Servlet3.1부터 non-blockingI/O를 제공했지만, 여전히 sync동작(Filter, Servelet)과 blocking(getParameter, getPart) 방식을 사용하고 있기 때문
  * 새로운 API가 잘구축된 non-blocking runtime에 필요로 했고, non-blocking, async를 구현한 서버(Netty)가 중요해졌음
* 함수형 프로그래밍
  * 람다가 Java8에 추가되며 자바에 함수형 프로그래밍의 기회가 만들어졌다.
  * non-blocking 어플리케이션과 연속적인 API 스타일들
  * 개발 모델 단계에서 Java8에서 Spring WebFlux는 함수형 웹처리와 어노테이션이 붙은 Controller를 제공한다.

Define "Reactive"
* 변화에 반응하는 개발모델, non-blocking은 완료를 통지 받거나 데이터가 가용됬음을 받는것들에 중요하다.
  * network 컴포넌트들은 I/O 이벤트를 감지하고
  * UI Controller들은 마우스 이벤트를 감지하고
* Spirng Webflux에서 중요하게 여기는것중 하나로, non-blocking back pressure
  * 동기식 코드에서는, blocking을 통해 caller가 기다리도록 하지만, ]
  * non-blocking 코드에서는 이벤트의 비율에 따라 제어하여 빠른 이벤트 생산자가 목적지(소비자 혹은 이벤트를 저장하는 저장소)를 압도하지 않도록 합니다.
* Reactive Streams은 스펙들을 제공하며, 그것들을 통해 구독자가 이벤트 생산자의 생성 속도를 제어 할 수 있도록 합니다.
  * 스펙(https://github.com/reactive-streams/reactive-streams-jvm/blob/master/README.md#specification)
    * 구성 요소
      * Publisher(발행자, 생산자)
      * Subscriber(구독자)
      * Subscription(신청)
      * Processor(프로세서)
    * 흐름
      * Publisher는 잠재적으로 무한의 데이터를 제공합니다, 데이터의 발행은 구독자에 의해 받은 요구에 따라 이루어진다.
      * Publisher.subscribe(Subscriber) 호출에 대한 응답은 순차적으로 메소드 호출 프로토콜에 의해 이루어 진다.
        * 프로토콜 : onSubscribe onNext* (onError | onComplete)?
          * onSubscribe는 언제나 신호로인해 불리는 부분이며, 무한의 데이터만큼 onNext가 신호로 인해 불리며, 에러가 있다면 onError, 더이상 데이터가 없다면 onComplete
    * 용어
      * Signal : onSubscribe, onNext, onComplete, onError, request(n) or cancel 메소드들
      * Demand : 구독자의 요청으로 인해 집계된 데이터 수, 더많은 데이터들을 request
      * Syncchronous(ly) : 호출중인 스레드에서 수행되는것
      * Return nomally : 호출자가 선언한 타임으로만 값을 반환합니다, 실패 신호를 알리는 정상적인 방법은 Subscriber의 onError를 통해야합니다.
      * Responsivity : 다른 컴포넌트간에는 응답들에 대해서는 영향을 주면 안됩니다.
      * Non obstructing : 호출 스레드에서 가능한 빠르게 실행할 수 있도록, 과도한 계산을 하지 않도록 한다.
      * Terminal state : 종결상태, Publisher : onComplete or onError 신호를 보냈을때, Subscriber : onComplete or onError가 도달 했을때
      * NOP : 호출한 스레드에 영향을 감지 할수 없는 실행, 여러번 실행하더라도 안전하다
      * Serial(ly) : Signal 관점에서, 덮어쓰여지지 않으며, JVM 관점에서는 호출간에 관계가 있는 경으에는 순차적으로 호출되며, 비동기적으로 수행될때에는 락 혹은 atomic 연산을 통해 순차적임을 제공
      * Thread-safe : 정확성을 보장하기 위해 외부 동기화 없이 안전하게 동기 혹은 비동기 방식 제공
    * 구성 상세
      * https://github.com/reactive-streams/reactive-streams-jvm/blob/master/README.md#specification
      * Publisher
        ```java
        public interface Publisher<T> {
          public void subscribe(Subscriber<? super T> s);
        }

        ```
        1. Publishers는 Subscriber가 request 한 데이터 보다 많은 신호를 발생시킬수 없다.
        2. Publisher는 요청보다 더 적은 onNext를 발생시킬수 있고, Subscription의 onComplete나 onError에 종료될 수 있습니다.
        3. onSubscribe, onNext, onError 와 onComplete 신호는 순차적으로 이뤄져야한다.
           * 멀티 쓰레드 환경에서도 지켜져야한다.
        4. Publisher의 실패는 onError를 통해 시그널 보내야 한다.
        5. Publisher의 정상적인 종료는 반드시 onComplete 를 사용한다.
        6. Publisher가 onError 혹은 onComplete를 시그널 하면, Subscriber는 Subscription에 cancelled를 고려해야한다.
        7. 한번 종료 상태(onError, onComplete)가 되면, 더이상 시그널을 발생시키지 않는다.
        8. Subscription이 cancelled되면, Subscriber는 반드시 결국에는 시그널을 멈춰야한다.
        9. Publisher.subscribe는 반드시 Subscriber의 onSubscribe를 호출해야하며, 정상적으로 리턴해야 한다.
           * Subscriber가 null일 경우에는 NullPointerException을 던진다.
           * onSubscribe는 최대 한번만 호출되어야합니다.
        10. Publisher.subscribe는 원하면 여러번 호출될수 있지만, 반드시 다른 Subscriber를 통해 호출해야합니다.
        11. Publisher는 여러개의 Subsciber를 지원할 수 있고, Subscription이 unicast인지 multicast인지에 따라 결정됩니다.
        
      * Subscriber
        ```java
        public interface Subscriber<T> {
            public void onSubscribe(Subscription s);
            public void onNext(T t);
            public void onError(Throwable t);
            public void onComplete();
        }

        ```
      * Subscription
        ```java
        public interface Subscription {
            public void request(long n);
            public void cancel();
        }

        ```
      
      
    


