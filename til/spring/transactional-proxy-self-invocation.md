# Spring `@Transactional` — Self-invocation에서 설정이 적용되지 않는 이유

> 학습일: 2026-11-11 · 분야: Spring / Transaction  
> 기록 유형: 이해 중심 TIL · 확인: **프록시 우회 원리를 내 말로 설명함** · Spring/DB 실행 **미검증**

## 1. 처음 헷갈린 부분

`@Transactional`이 붙어 있으면 메서드가 호출될 때 언제나 트랜잭션 설정이 적용된다고 생각했다. 그런데 같은 서비스 객체 안에서 메서드를 호출하면 왜 적용되지 않는지 이해하기 어려웠다. `REQUIRES_NEW`뿐 아니라 기본값인 `REQUIRED`에도 같은 제약이 있는지 궁금했다.

## 2. 이해한 내용

Spring의 **기본 프록시(proxy) 방식**에서는 외부에서 Bean의 메서드를 호출할 때 프록시가 트랜잭션 처리를 가로챈다. 프록시는 메서드의 `@Transactional` 설정에 따라 트랜잭션을 시작하거나 기존 트랜잭션에 참여하도록 처리한다.

반면 실제 서비스 객체 내부에서 `this.methodB()`처럼 다른 메서드를 직접 호출하면 **프록시를 다시 거치지 않는다.** 그래서 **호출 대상 `methodB()`의 `@Transactional` 설정이 이 호출에 대해서는 적용되지 않는다.** 이를 *Self-invocation(자기 호출)*이라고 한다.

중요한 예외 조건: **프록시 우회가 곧 '트랜잭션 없음'은 아니다.** 바깥 `methodA()`가 이미 트랜잭션을 시작했다면, 내부 DB 작업은 그 트랜잭션에 포함될 수 있다. 단지 내부 메서드의 별도 전파 설정이 추가로 적용되지 않는 것이다. 이 설명은 *Spring의 기본 프록시 방식*을 전제하며, AspectJ 방식은 다르다.

## 3. 내 말로 설명할 수 있는 부분

```java
@Service
public class ExampleService {
    public void methodA() {
        this.methodB();
    }

    @Transactional
    public void methodB() {
        // DB 작업이 있다고 가정
    }
}
```

**조건:** Controller가 Spring Bean의 `methodA()`를 외부에서 호출한다. 트랜잭션 처리는 기본 프록시 방식이며, 이미 존재하는 상위 트랜잭션은 없다고 가정한다. 코드는 설명용으로 **미실행**이다.

```text
Controller → Spring Bean 프록시 → 실제 methodA()
                                      └→ this.methodB() (프록시 우회)
```

**내 예상:** `methodB()`에 붙은 `@Transactional`은 이 내부 호출에 적용되지 않는다. 따라서 **그 어노테이션 때문에** 트랜잭션이 시작되지는 않는다. 실제 Repository 자체의 트랜잭션 동작은 별개이므로, 모든 DB 작업이 반드시 트랜잭션 없이 실행된다고 단정하지는 않는다.

학습 대화에서 나는 "실제 ExampleService 객체 안에서 호출하기 때문에 프록시를 우회한다"고 설명했다. 이 호출 흐름을 **말로 이해한 것**과 실제 DB에서 실행해 **검증한 것**은 구분한다.

## 4. 미검증 및 다음 확인

- **확인한 범위:** 프록시 경유 호출과 자기 호출을 구분하고, 내부 메서드의 어노테이션이 적용되지 않는 이유를 설명함.
- **미검증:** Spring 실행 환경에서 프록시 동작, 트랜잭션 활성 상태, 실패 후 DB 최종 상태.
- **선택적 실험:** 같은 클래스의 `this.methodB()`와 별도 Spring Bean의 `methodB()` 호출을 비교한다. 트랜잭션 상태 및 최종 DB 결과를 검사한다.

## 5. 참고 자료

- [Spring 공식 문서 — `@Transactional` 사용 및 Self-invocation 제약](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring 공식 문서 — 선언적 트랜잭션 구현](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html)
- [연결 TIL — REQUIRED와 REQUIRES_NEW](transaction-required-vs-requires-new.md)

**한 문장 정리:** `@Transactional`이 붙어 있는지만 볼 것이 아니라, **실제 호출 경로가 프록시를 거치는지** 확인해야 한다.

**스스로 확인할 질문:** 바깥 `methodA()`에 `@Transactional`이 이미 적용되었다면, `this.methodB()`가 프록시를 우회해도 내부 DB 작업은 어떤 트랜잭션에서 실행될까?
