---
layout: post
title: "JPA- Optimistic Lock , Pessimistic Lock"
author: "jhkim593"
tags: JPA

---

## JPA Optimistic Lock
JPA가 제공하는 낙관적 락은 버전(@Version)을 사용합니다. 따라서 낙관적 락을 사용하려면 버전이 있어야합니다. 낙관적 락은 트랜잭션을 커밋하는 시점에 충돌을 알 수있다는 특징이있습니다.
락 옵션없이 @Version만 있어도 낙관적 락이 적용됩니다. 락 옵션을 사용하면 락을 더 세밀하게 제어할 수있습니다.

- 락 옵션 적용 X :락 옵션을 적용하지 않아도 entity에 @Version이 적용된 필드만 있으면 낙관적 락이 적용됩니다. 조회한 entity를 수정할 때 다른 트랜잭션에 의해 변경 되지 않아야 합니다. 엔티티를 수정할 때 버전을 체크하면서 버전을 증가합니다. 이때 데이터베이스의 버전 값이 현재 버전이 아니면 예외가 발생합니다.

- OPTIMISTIC : @Version만 적용했을 때와 다르게 entity를 조회만 해도 버전을 체크합니다. 이 것은 한번 조회한 entity는 트랜잭션을 종료할 때까지 다른 트랜잭션에서 변경하지 않음을 보장합니다. 락 옵션을 걸지 않고 @Version만 사용하면 entity를 수정해야 버전 정보를 확인하지만 Optimitstic옵션을 사용하면 entity를 수정하지 않고 단순히 조회만 해도 버전을 확인합니다.

- OPTIMISTIC_FORCE_INCREMENT : OPTIMISTIC과 달리 버전을 강제로 증가시키는 잠금입니다. 커밋 직전에 아래처럼 버전만 증가시키는 쿼리가 항상 발행됩니다. 따라서 해당 엔터티에 변경이 있었을 경우에는 변경사항에 대한 업데이트문과 버전을 증가시키는 업데이트문에 의해서 두번 버전이 증가합니다. OPTIMISTIC와 동일하게 엔터티 자체에 변경사항이 있을 경우에는 불필요하게 업데이트 문이 발행되므로 주의할 필요가 있습니다. 그리고 암시적인 행 배타잠금(Row Exclusive Lock)이 발생되어 정합성을 보증할 수는 있으므로 자식 엔터티를 수정할때 자식엔터티 전체에 대한 잠금용도로 사용할 수 있습니다.



- @OptimisticLocking
  - @Version
  - ALL<br>
      위의 @Version 필드에 의한 잠김과는 다르게 조건절에 전체 컬럼이 걸려있습니다. 이와 같이 컬럼 전체에 대한 업데이트 여부를 확인 함으로써 버전없는 낙관적 잠금이 가능합니다. 주의할 점은 ALL을 사용할 경우에는 @DynamicUpdate 와 같이 사용해야 하는데 이유는 필드 단위로 Dirty 여부를 확인 하기 위함입니다.

  - DIRTY<br>
      DIRTY로 지정했을 경우에는 위와 같이 갱신될 컬럼의 갱신전 값으로 조건절에 바인딩 됩니다. 특정한 컬럼만 충돌확인이 되므로 ALL이나 @Version을 사용 했을 때에 비해 충돌 가능성을 낮출 수 있습니다. 결과적으로 특정 엔티티의 서로 다른 부분을 업데이트 하는 프로그램이 있을 경우 충돌하지 않고 처리가 가능합니다.

<br>

- @Version을 적용할 수있는 타입
  - int
  - Integer
  - short
  - Short
  - long
  - Long
  - java.sql.Timestamp

  <br>


## JPA Pessimistic Lock
race condition이 발생한다는 것을 전제로 두고 아예 잠궈버리는 방식입니다. Optimistic locking보다 데이터 정합성에서 확실하게 보장 받을 수있습니다. 버전 정보는 사용하지 않으며 select for update구문을 사용하며 , PESSIMISTIC_WRITE모드를 주로 사용합니다.  다만 성능적인 측면에서 손실을 감수해야합니다.

- PESSIMISTIC_WRITE
   - 데이터 베이스에 쓰기 락을 걸 때 사용합니다
   - select for update를 사용해서 락을 겁니다.
   - NON-REPEATABLE READ방지 락이 걸린 로우는 다른 트랜잭션이 수정할 수없습니다.


~~~java
 public interface PayRepository extends JpaRepository<Pay,Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Pay> findByPayId(String payId);
}

~~~

~~~java
 private ExecutorService ex = Executors.newFixedThreadPool(100);
private CountDownLatch latch=new CountDownLatch(100);

 @Test
public void lockTest()throws Exception{

    PayRequestDto payRequestDto=new PayRequestDto();
    payRequestDto.setCardNum("1234567890123456");
    payRequestDto.setExpirationDate("1125");
    payRequestDto.setCvc("777");
    payRequestDto.setPrice(20000L);
    payRequestDto.setVat(5000L);
    PayResponseDto payResponseDto = payService.createPay(payRequestDto);
    List<CancelPayResponseDto>list=new ArrayList<>();

    for(int i=0; i<100;i++){
        ex.execute(()->{
            try {
                CancelPayRequestDto cancelPayRequestDto=new CancelPayRequestDto();
                cancelPayRequestDto.setPayId(payResponseDto.getPayId());
                cancelPayRequestDto.setCancelPrice(100L);
                cancelPayRequestDto.setVat(10L);
                CancelPayResponseDto cancelPay = payService.createCancelPay(cancelPayRequestDto);
                list.add(cancelPay);
            } catch (Exception e) {

            }
            latch.countDown();
        });
    }
    latch.await();

    CancelPayRequestDto cancelPayRequestDto=new CancelPayRequestDto();
    cancelPayRequestDto.setPayId(payResponseDto.getPayId());
    cancelPayRequestDto.setCancelPrice(100L);
    cancelPayRequestDto.setVat(10L);

    CancelPayResponseDto cancelPay = payService.createCancelPay(cancelPayRequestDto);

    assertThat(cancelPay.getOriPrice()).isEqualTo(9900L);
    assertThat(cancelPay.getOriVat()).isEqualTo(3990L);
    assertThat(list.get(list.size()-1).getOriPrice()).isEqualTo(10000L);
    assertThat(list.get(list.size()-1).getOriVat()).isEqualTo(4000L);

 }

}
~~~
메소드 내부에서 CountDownLatch로 100번 createCancelPay를 호출해서 100원을 100번 총 10000원 취소했습니다. 테스트가 성공하며 Lock설정을 통해 데이터의 정합성이 보장되는 것을 확인했습니다.



- PESSIMISTIC_REDAD
  - 데이터를 반복 읽기만 하고 수정하지않는 용도로 락을 걸 때 사용합니다. 일반적으로 잘 사용하지 않으며 대부분 데이터베이스 방언에 의해 PESSIMISTIC_WRITE로 동작합니다.


- PESSIMISTIC_FORCE_INCREMENT
   - 비관적 락중 유일하게 버전 정보를 사용하며 버전정보를 강제로 증가시킵니다.
