# Spring Self-invocation — 내부 호출은 왜 `@Transactional`을 다시 적용하지 않을까?

> 작성일: 2026-11-11 · 분야: Spring
> 기록 유형: 이해 중심 · 상태: 대화에서 원리 설명함 / 실제 Spring·DB 실행 미검증
> **초안:** 본인이 내용과 공개 범위를 검토한 뒤 게시 결정

## 1. 처음 헷갈린 부분

`@Transactional`이 메서드에 붙어 있으면 호출할 때마다 자동으로 트랜잭션이 시작된다고 생각했다. 그런데 같은 서비스 객체 안에서 메서드를 부르면 프록시를 우회한다는 설명이 이해되지 않았다. `REQUIRES_NEW`뿐 아니라 기본 `REQUIRED`에도 이 문제가 있는지 궁금했다.

## 2. 이해한 내용

Spring의 기본 프록시 기반 트랜잭션에서는 프록시가 호출을 가로채 트랜잭션 경계를 처리한다. 그러나 실제 객체 내부의 `this.methodB()` 호출은 프록시를 다시 거치지 않아 **`methodB()`에 선언된 트랜잭션 설정이 새로 적용되지 않는다**.

**반례가 아닌 중요한 조건:** 이미 바깥 `methodA()`가 트랜잭션을 시작했다면 `methodB()`의 DB 작업은 그 기존 트랜잭션 안에서 실행될 수 있다. 따라서 자기 호출을 `항상 트랜잭션 없음`으로 외우면 안 된다.

## 3. 내 말로 설명할 수 있는 부분

```java
@Service
public class ExampleService {
    public void methodA() {
        methodB(); // this.methodB(): 자기 호출
    }

    @Transactional
    public void methodB() {
        // DB 작업 (설명용 코드, 미실행)
    }
}
```

Spring 프록시를 통해 `methodA()`를 호출하더라도, `methodA()`의 실행 도중 실제 `ExampleService` 객체 내부에서 `methodB()`를 호출하기 때문에 프록시를 우회한다. 따라서 바깥에 다른 트랜잭션이 없다는 조건에서는 `methodB()`의 어노테이션으로 새 트랜잭션이 시작되지 않는다. 이는 2026-11-11 학습 대화에서 직접 설명한 범위다.

## 4. 미검증 및 다음 확인

- **확인한 것:** 자기 호출이 프록시를 우회하는 이유를 말로 설명함.
- **미검증:** `REQUIRES_NEW`의 실제 트랜잭션 분리, DB 최종 상태 및 Rollback 결과.
- **다음 검증 아이디어:** 같은 객체 내부 호출과 별도 Spring Bean 호출을 테스트로 비교하고, 실패 시 최종 DB 상태를 확인한다. 실제 테스트는 아직 하지 않았다.

## 5. 참고 자료

- [Spring 공식 문서 — Annotation-driven Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring 공식 문서 — Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)

**한 문장 정리:** `@Transactional`의 선언만 보지 말고, 호출이 프록시를 거치는지와 이미 트랜잭션이 존재하는지를 함께 확인해야 한다.

**스스로 확인할 질문:** `methodA()`가 먼저 트랜잭션을 시작하고 `this.methodB()`를 호출하는 경우, `methodB()`의 `REQUIRES_NEW`는 적용될까? 아니면 기존 트랜잭션 안에서 실행될까?
