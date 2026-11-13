# Spring 트랜잭션 전파 — `REQUIRED`와 `REQUIRES_NEW`의 차이

> 학습일: 2026-11-11 · 이해 확인: 2026-11-13 · 분야: Spring / Transaction  
> 기록 유형: 이해 중심 TIL · 확인: **자기 호출 상황의 결과를 내 말로 예측함** · 별도 Bean 비교/DB 실행 **미검증**

## 1. 처음 헷갈린 부분

`@Transactional(propagation = Propagation.REQUIRES_NEW)`가 정확히 무엇을 뜻하는지 몰랐다. 기존 트랜잭션에 참여하는 `REQUIRED`와 어떻게 다른지, 바깥 메서드에서 예외가 나면 안쪽 저장 결과가 남는지 궁금했다.

## 2. 이해한 내용

**전파(Propagation)**는 트랜잭션 설정이 적용되는 메서드를 호출할 때 기존 트랜잭션을 어떻게 다룰지 정하는 옵션이다.

| 설정 | 이미 트랜잭션 A가 있다면 | 없다면 |
| --- | --- | --- |
| `REQUIRED` (기본값) | 기존 A에 참여 | 새 트랜잭션 시작 |
| `REQUIRES_NEW` | A를 잠시 중단하고 독립적인 B 시작 | 새 트랜잭션 시작 |

`REQUIRED`로 기존 A에 참여한 저장 작업은 A와 **같은 물리 트랜잭션**에 속하므로 A가 롤백되면 함께 롤백된다. 반면 `REQUIRES_NEW`로 시작한 B가 정상적으로 **Commit까지 완료되었다면**, 이후 A가 롤백되어도 B의 변경은 남을 수 있다.

단, 위 표는 **Spring의 트랜잭션 설정이 실제로 적용되는 호출**에 대한 설명이다. 기본 프록시 방식에서 `this.inner()`로 자기 호출하면 내부 어노테이션이 적용되지 않는다. 자세한 호출 구조는 [Self-invocation TIL](transactional-proxy-self-invocation.md)에 정리했다.

`REQUIRES_NEW`는 독립적인 연결 등 DB 자원을 추가로 필요로 할 수 있으므로, 항상 사용하기보다 **정말 별도 Commit이 필요한지**와 Connection Pool 부담을 고려해야 한다.

## 3. 내 말로 설명할 수 있는 부분

### 직접 답변으로 확인한 예측: 자기 호출

아래 코드는 실제 실행한 코드가 아닌 **설명용 의사 코드**다. Controller가 `accept()`를 프록시를 통해 호출하고, 두 저장 작업이 같은 트랜잭션 매니저와 DB 자원에 참여한다고 가정한다. 기본 롤백 규칙이며 예외를 잡지 않는다.

```java
// InvitationService (동일한 Spring Bean 안에 두 메서드가 있음)
@Transactional
public void accept() {
    participationRepository.save(participation);
    this.saveAttempt();
    throw new RuntimeException("처리 실패");
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAttempt() {
    attemptRepository.save(attempt);
}
```

**내가 예측한 흐름:**

1. `accept()`에 대한 외부 호출을 프록시가 가로채 트랜잭션 A를 시작한다.
2. `this.saveAttempt()`는 프록시를 우회하므로 내부의 `REQUIRES_NEW`가 적용되지 않는다.
3. 두 저장 작업은 **기존 A의 변경**이며, 새로운 B는 시작되지 않는다.
4. `RuntimeException`이 `accept()` 바깥까지 전파되면 기본 규칙에 따라 A 전체가 롤백된다.

**예상 DB 최종 상태:** `participation`과 `attempt` 모두 남지 않는다. `saveAttempt()`가 *별도로* 롤백되는 것이 아니라, **같은 트랜잭션 A의 변경이 함께 취소되는 것**이다.

이는 내가 학습 대화에서 직접 설명한 판단이며, 실제 DB 실행 결과는 아니다.

### 아직 실행으로 비교하지 않은 조건: 별도 Bean

```java
// InvitationService에서 호출: attemptService는 별도 Spring Bean
attemptService.saveAttempt(attempt);

// AttemptService의 public 메서드 (프록시 경유 호출)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAttempt(Attempt attempt) {
    attemptRepository.save(attempt);
}
```

**예상:** 별도 Bean의 프록시에서 `REQUIRES_NEW`가 적용되고 B가 성공적으로 Commit했다면, 이후 A의 롤백에도 `attempt`는 남는다. `REQUIRED`라면 A에 참여하므로 함께 롤백된다. **이 비교 상황은 직접 실행·독립 확인하지 않았다.**

## 4. 미검증 및 다음 확인

- **확인한 범위:** 자기 호출에서 `REQUIRES_NEW`가 적용되지 않고, A의 롤백에 두 저장이 함께 포함되는 이유를 말로 설명함.
- **미검증:** 별도 Bean 호출과 `REQUIRED`/`REQUIRES_NEW` 비교의 실제 Commit/Rollback 결과 및 자원 비용.
- **선택적 실험:** 두 호출 경로를 각각 실행한 뒤 **테스트 트랜잭션 바깥의 별도 DB 조회**로 최종 상태를 비교한다.

**실행 상태: 미실행.** 예상과 실제 관찰을 혼동하지 않는다.

## 5. 참고 자료

- [Spring 공식 문서 — 트랜잭션 전파](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
- [Spring 공식 문서 — `@Transactional` 및 프록시 모드](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring 공식 문서 — 기본 롤백 규칙](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)
- [연결 TIL — 프록시와 Self-invocation](transactional-proxy-self-invocation.md)

**한 문장 정리:** `REQUIRED`는 기존 트랜잭션에 참여하고 `REQUIRES_NEW`는 독립 트랜잭션을 시작하지만, **Spring의 기본 프록시 방식에서는 자기 호출로 내부 전파 설정을 적용할 수 없다.**

**스스로 확인할 질문:** `saveAttempt()`를 별도의 Spring Bean으로 옮겨 B가 성공적으로 Commit한 후 A에서 예외가 발생한다면, 두 테이블의 최종 상태는 어떻게 다를까?
