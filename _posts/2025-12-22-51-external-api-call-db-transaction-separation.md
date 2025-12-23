---
layout: post
title: "외부 API 호출 시 DB 트랜잭션 분리"
author: "jhkim593"
tags: Architecture
---

프로젝트를 진행하면서 결제 API 연동을 구현했는데, 결제와 같은 외부 API 호출 시 DB 트랜잭션 관리가 생각보다 까다로웠습니다.
이번 게시글에서는 외부 API 호출 시 DB 트랜잭션을 처리하면서 발생한 문제와 이를 해결한 **트랜잭션 분리 방법**에 대해 다뤄보겠습니다.

<br>

## 요구사항

결제 API와 연동하면서 만족해야 할 2가지 조건이 있었습니다.

**1. 결제 API 결과와 DB 데이터 간 정합성이 중요**
- 결제 API 호출 결과와 DB에 저장된 결제 상태가 일치해야 합니다.

**2. 사용자는 결제 결과를 즉시 알아야 함**
- 사용자가 결제 성공/실패 여부를 즉시 응답받아야 합니다.

<br>

## 결제 API 롤백 문제

결제 API 호출보다 **결제 데이터를 먼저 저장한 이유는 결제 API는 롤백이 쉽지 않기 때문**입니다. 

만약 결제 API를 먼저 호출하고 이후에 DB에 저장하는 순서라면, DB 저장이 실패했을 때 결제는 성공했지만 DB에는 반영되지 않아 상태 불일치가 발생합니다.

이 경우 결제 취소 API를 호출하는 등의 보상 로직이 필요한데, 보상 로직 또한 실패할 수 있어 문제가 더 복잡해집니다.

이러한 이유로 먼저 DB에 모든 데이터를 저장한 뒤 결제 API를 호출하는 구조로 설계를 진행했습니다.

<br>

## 하나의 트랜잭션으로 처리

처음에는 DB에 데이터를 미리 저장하고, 결제 API를 호출하는 방식으로 구현했습니다.

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentRepository paymentRepository;
    private final PaymentApi paymentApi;

    @Transactional
    public void processPayment(PaymentRequest request) {
        // 1. 결제 데이터 저장 
        Payment payment = Payment.create(request);
        paymentRepository.save(payment);

        // 2. 외부 API 호출
        PaymentResponse response = paymentApi.requestPayment(request);
    }
}
```
`processPayment`에 `@Transactional`을 적용하여 결제 API 호출이 실패하더라도 DB에 저장된 결제 데이터도 함께 롤백되도록 해 정합성을 보장했습니다.

<br>


이 방식은 정합성 측면에서 확실한 **장점**이 있습니다.
- DB 저장이 실패하면 트랜잭션이 롤백되고 결제 API 호출을 하지 않습니다.
- 결제 API 호출이 실패하면 예외가 발생하여 DB에 저장된 데이터도 롤백됩니다.

<br>

하지만 **성능 문제가 발생**할 수 있습니다.

전체 과정이 하나의 트랜잭션으로 묶여있기 때문에, **결제 API 응답을 기다리는 동안 계속 DB 커넥션을 점유**하게 됩니다.

결제 API의 경우 응답 지연을 고려해 read timeout을 30초로 설정해야 했는데, DB 커넥션 풀의 크기는 제한되어 있기 때문에 동시에 많은 요청이 들어오면 커넥션을 받지 못한 요청들은 대기하거나 실패하게 됩니다.

<br>
이러한 문제를 확인하고 나서, 결제 API 호출과 DB 트랜잭션을 분리하는 새로운 구조를 시도했습니다.

<br>

## DB 트랜잭션과 결제 API 분리

결제 사전 데이터 생성 → 외부 API 호출 → 결제 데이터 업데이트 순서로 진행해 DB 트랜잭션과 결제 API를 분리했습니다.
```java

@Service
@RequiredArgsConstructor
public class PaymentService {
   private final PaymentTransactionManager transactionManager;
   private final PaymentApi paymentApi;

   public void processPayment(PaymentRequest request) {
      // 1. 결제 데이터 저장
      Payment payment = transactionManager.createPayment(request);

      // 2. 외부 API 호출 
      PaymentResponse response = paymentApi.requestPayment(request);

      // 3. 결제 데이터 업데이트
      transactionManager.updatePayment(payment, response.getStatus());
   }
}
```

```java
@Component
@RequiredArgsConstructor
public class PaymentTransactionManager {
   private final PaymentRepository paymentRepository;

   @Transactional
   public Payment createPayment(PaymentRequest request) {
      Payment payment = Payment.create(request);
      return paymentRepository.save(payment);
   }

   @Transactional
   public void updatePayment(Payment payment, String status) {
      payment.update(status);
      paymentRepository.save(payment);
   }
}
```
각 DB 작업은 독립적으로 실행되기 때문에 결제 API 응답을 기다리지 않고 이전 방식보다 빠르게 커밋됩니다.

<br>
하지만 트랜잭션을 분리하면서 정합성 문제가 다시 발생할 수 있는데 

결제 데이터 업데이트 단계에서 실패하면, 결제는 성공했지만 DB에는 업데이트되지 않아 **실제 결과와 DB 간 상태 불일치**가 발생하게 됩니다.

또한 결제 데이터 업데이트 과정에서 예외가 던져진다면 결제는 성공했음에도 사용자는 결제 실패 응답을 받게 되는 문제도 있습니다.

위와 같은 문제를 해결하기 위해 **비동기 처리와 주기적인 정합성 체크 로직을 추가**했습니다.

<br>

### 비동기 처리

결제 데이터 업데이트 과정은 예외가 발생하더라도 무시해야 하고 완료될 때까지 대기할 필요가 없기 때문에 비동기로 처리하도록 수정했습니다.
```java
@Component
@RequiredArgsConstructor
public class PaymentTransactionManager {
   private final PaymentRepository paymentRepository;
    
   ...
    
   @Async
   @Transactional
   public void updatePayment(Payment payment, String status) {
      payment.update(status);
      paymentRepository.save(payment);
   }
}
```
`updatePayment` 메소드에 `@Async`를 추가했습니다. 
이렇게 하면 결제 API 호출이 완료된 후 바로 사용자에게 응답을 반환하고, 결제 데이터 업데이트가 실패하더라도 사용자 응답에는 영향을 주지 않게 됩니다.

<br>

### 주기적인 정합성 체크
결제 데이터 업데이트가 실패하면 결제 API 결과와 DB 간 정합성이 깨질 수 있습니다.
이를 해결하기 위해 주기적으로 업데이트되지 않은 결제 건을 조회해서, 결제 조회 API로 실제 상태를 확인하고 DB에 반영하는 방식을 사용했습니다.

```java
@Component
@RequiredArgsConstructor
public class PaymentRetryer {
    private final PaymentTransactionManager transactionManager;
    private final PaymentRepository paymentRepository;
    private final PaymentApi paymentApi;

    @Scheduled(
            fixedDelay = 10, 
            timeUnit = TimeUnit.SECONDS
    )
    public void checkPendingPayments() {
        List<Payment> pendingPayments = paymentRepository.findPendingPayments();

        for (Payment payment : pendingPayments) {
            try {
                updatePayment(payment);
            } catch (Exception e) {
                ...
            }
        }
    }

    private void updatePayment(Payment payment) {
        PaymentResponse response = paymentApi.getPayment(payment.getId());
        transactionManager.updatePayment(payment, response.getStatus());
    }
}
```
먼저 `@Scheduled`를 통해 10초마다 업데이트되지 않은 결제 건들을 조회합니다.
이후 결제 조회 API를 호출해 실제 결제 상태를 확인하고, 그 결과를 DB에 반영해 정합성을 유지할 수 있도록 했습니다.

<br>

## 최종 동작 흐름

최종적으로 적용된 결제 처리 흐름을 정리하면 다음과 같습니다.

**1. 결제 요청**
- 결제 사전 데이터를 DB에 저장합니다. (트랜잭션 1)
- 결제 API를 호출합니다.
- 결제 API 성공 시 즉시 사용자에게 성공 응답을 반환합니다.

**2. 비동기 DB 업데이트**
- 결제 데이터 업데이트가 비동기로 실행됩니다 (트랜잭션 2)
- 업데이트가 실패하더라도 사용자 응답에는 영향을 주지 않습니다.

**3. 주기적인 정합성 체크**
- 10초마다 업데이트되지 않은 결제를 조회합니다.
- 결제 조회 API로 실제 상태를 확인하여 DB에 반영합니다.

<br>

## 주의사항

이 방식은 업데이트가 비동기로 이루어지기 때문에 **결제 API 결과와 DB 간 일시적인 불일치가 발생합니다.**

예를 들어 결제 API는 성공했지만, 업데이트가 아직 완료되지 않았거나 실패한 경우 일시적으로 업데이트 전 상태로 남아있게 됩니다. 이러한 불일치는 `PaymentRetryer`가 주기적으로 체크해 **최종적으로는 일치하도록 보장**하지만 일정 시간 동안 불일치가 발생합니다.



