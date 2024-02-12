# Project Reactor

[https://github.com/schananas/practical-reactor?tab=readme-ov-file](https://github.com/schananas/practical-reactor?tab=readme-ov-file)

## Introduction to Reactive Programming

### Reactive Programming

- Publisher-Subscriber
- Publisher가 Subscriber에게 통지, `push`

### R.P 필요성

- Blocking can be wasteful
    - 프로그램의 성능 향상 방법:
        - 많은 스레드, 하드웨어 자원 병렬화
        - 현재 리소스에서 효율성 추구
    - Worse still, blocking wastes resources. If you look closely, as soon as a program involves some latency (notably I/O, such as a database request or a network call), resources are wasted because threads (possibly many threads) now sit idle, waiting for data.

- Async to Rescue ?
    - . By writing asynchronous, non-blocking code, **you let the execution switch to another active task that uses the same underlying resources and later comes back to the current process when the asynchronous processing has finished.**
        - Callback:  콜백 지옥 ..
        - Futures: get() 이 blocking methods, multi value & error handling 지원 부족, lazy computation 기능 X
        
- So `Project Reactor` 은 다음과 같은 장점이 있다..
    - **Composability** and **readability**
        - 콜백 지옥 같이 nested 되서 알아보기 힘든 구조가 아니라, multi async task 를 orchestrate 하기 쉽도록 & nested 최소화
    - Data as a **flow** manipulated with a rich vocabulary of **operators**
    - Nothing happens until you **subscribe**
        - `Subscribing` 하면 Publisher-Subscirber 연결
    - **Backpressure** or *the ability for the consumer to signal the producer that the rate of emission is too high*
        - we described in the assembly line analogy as a feedback signal sent up the line when a workstation processes more slowly than an upstream workstation.
    - **High level** but **high value** abstraction that is *concurrency-agnostic*
    
    → `Assembly Line` 처럼 .. raw materials pours from a source (⇒ `Pulbisher`) , a finished product ready to be pushed to the customer (⇒ `Subscribier`)
    
    raw material들은 여러 transformation 거치고.. assembly line의 파트가 되기도 하고 ..
    
    만약 문제가 생기면(⇒product를 포장하는 과정에서 필요 이상의긴 시간이 걸린다면..), workstation은 raw material의 flow를 제한한다 .. (⇒ 아마 `backpressure` 의 비유인듯)
    
    → **Each operator adds behavior to a `Publisher` and wraps the previous step’s `Publisher` into a new instance**. 
    
    The whole chain is thus linked, such that data originates from the first **`Publisher`** and moves down the chain, transformed by each link. Eventually, a **`Subscriber`** finishes the process. Remember that nothing happens until a **`Subscriber`** subscribes to a **`Publishe`**
    
    ## Reactor Core Features
    
    - Flux: reactive sequence of 0..N items
    - Mono: single-value-or-empty(0..1) result

→ 연산자가 type을 바꿀 수 있음 (`count` 오퍼레이터는 Flux에 존재하지만, `Mono<Long>` 을 리턴)

![Untitled](Project%20Reactor%200417f0701b3a443f8766251ba493d9f8/Untitled.png)

A **`Flux<T>`** is a standard **`Publisher<T>`** that **represents an asynchronous sequence of 0 to N emitted items, optionally terminated by either a completion signal or an erro**r. As in the Reactive Streams spec, these three types of signal translate to calls to a downstream Subscriber’s **`onNext`**, **`onComplete`**, and **`onError`** methods.

![Untitled](Project%20Reactor%200417f0701b3a443f8766251ba493d9f8/Untitled%201.png)

A **`Mono<T>`** is a specialized **`Publisher<T>`** that emits at most one item *via* the **`onNext`** signal then terminates with an **`onComplete`** signal (successful **`Mono`**, with or without value), or only emits a single **`onError`** signal (failed **`Mono`**).

### Subscribe

```jsx
subscribe(); 
//subscibe & trigger the sequence

subscribe(Consumer<? super T> consumer); 
//do smth with each produced value

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer); 
//deal w/ values but also reacts to an error

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer); 
//deal w values but run some codes when the sequence successfully completes

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer,
          Consumer<? super Subscription> subscriptionConsumer);
//deal w/values, errors, successful completion but also do smth 
//with the Subscription produced by 
//this subscibe calls..
```

```java
Flux<Integer> ints = Flux.range(1, 4); 
ints.subscribe(i -> System.out.println(i),
    error -> System.err.println("Error " + error),
    () -> System.out.println("Done"));

/*
1
2
3
4
Done
*/
```

- Cancelling a `subscribe()` with Its `Disposable`
    
    subscribe() 에는 Disposable 리턴 타입이 있고, `dispose()` 를 호출하여 subscription cancel 
    
    `Disposables` class 를 이용 가능 
    
    - `Disposables.swap()` 을 이용하여 Disposable Wrapper 생성, 원자적 취소 가능
    - `Disposables.composite()` 를 이용하여 여러가지(서비스 호출과 관련된 여러 진행 중인 요청)을 수집하고 한꺼번에 취소 가능

- An alternative to Lambdas: `BaseSubscriber`
    
    lambda 식을 다음과 같은 클래스로 대체 가능
    
    ```java
    SampleSubscriber<Integer> ss = new SampleSubscriber<Integer>();
    Flux<Integer> ints = Flux.range(1, 4);
    ints.subscribe(ss);
    
    ///
    
    package io.projectreactor.samples;
    
    import org.reactivestreams.Subscription;
    
    import reactor.core.publisher.BaseSubscriber;
    
    public class SampleSubscriber<T> extends BaseSubscriber<T> {
    
    	@Override
    	public void hookOnSubscribe(Subscription subscription) {
    		System.out.println("Subscribed");
    		request(1);
    	}
    
    	@Override
    	public void hookOnNext(T value) {
    		System.out.println(value);
    		request(1);
    	}
    }
    
    /////RESULT
    Subscribed
    1
    2
    3
    4
    ```
    

`BaseSubsciber` 상속, user-defined subscriber 만들 수 있음

subsciber behavior를 오버라이드 가능 by hook 

`BaseSubscirber` —> `requestUnbounded` (==`request(Long.MAX_VALUE)`) , `cancel` 제공 ..
