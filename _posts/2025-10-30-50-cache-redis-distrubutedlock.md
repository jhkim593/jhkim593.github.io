---
layout: post
title: "Redis 분산락을 활용한 Cache Stampede 방지"
author: "jhkim593"
tags: Redis
---

## Cache Stampede란?

Cache Stampede(캐시 스탬피드)는 캐시된 데이터가 만료되는 순간 다수의 요청이 동시에 데이터베이스에 접근하여 동일한 데이터를 조회하고 캐시를 갱신하려는 현상을 말합니다.

일반적으로 캐시는 데이터베이스의 부하를 줄이고 응답 속도를 개선하기 위해 사용됩니다. 자주 조회되는 데이터를 메모리에 저장해두고, 동일한 요청이 들어오면 **데이터베이스를 거치지 않고 캐시에서 바로 응답하는 방식**입니다. 하지만 영구적으로 유지되지 않으며 일정 시간이 지나거나 추가 처리를 통해 만료됩니다.

문제는 데이터의 캐시가 만료되는 순간에 발생합니다. 캐시가 만료됐을 때 요청이 몰린다면 모든 요청이 캐시 미스를 발생시키고 데이터베이스에 쿼리를 실행하게 됩니다. 이는 순간적으로 **데이터베이스에 큰 부하를 발생**시킬 수 있습니다.

<br>

### 발생 유형

Cache Stampede는 발생 원인에 따라 크게 두 가지 유형으로 구분할 수 있습니다.

- **같은 키에 대한 동시 요청**  
특정 인기 데이터의 캐시가 만료될 때, 동일한 키에 대해 여러 요청이 동시에 발생하는 경우입니다.  
**해결 방법**: 분산 락을 사용하여 동일한 키에 대해서는 첫 번째 요청만 DB를 조회하도록 제어하고, 나머지 요청들은 첫 번째 요청이 갱신한 캐시를 사용하도록 합니다.
 
- **서로 다른 키들의 동시 만료**  
여러 캐시 키들이 동일한 시간에 만료되어 동시에 DB 부하가 발생하는 경우입니다.  
**해결 방법**: TTL에 랜덤값을 추가하여 만료 시점을 분산시킬 수 있습니다.

이번 게시글에서는 분산 락을 사용해 **같은 키에 대한 동시 요청 문제**를 다뤄보도록 하겠습니다.

<br>

## Redis 분산락

Redis의 분산락은 `SET` 명령어의 `NX`와 `PX`옵션을 활용하여 구현됩니다.

```
SET lock:key value NX PX 3000
```

- **NX**: 키가 존재하지 않을 때만 값을 설정합니다. 이미 키가 존재하면 명령이 실패하여, 동시에 여러 요청이 들어와도 첫 번째 요청만 락을 획득할 수 있습니다.
- **PX**: 락의 자동 만료 시간을 밀리초 단위로 설정합니다. 락을 획득한 스레드가 비정상 종료되어도 일정 시간 후 자동으로 락이 해제되어 데드락을 방지합니다.
- **value**: 락 소유자를 식별하기 위한 고유 값입니다. 락 해제 시 자신이 획득한 락인지 확인하는 데 사용됩니다.

Redis는 단일 스레드로 동작하므로 `SET NX` 명령은 **원자적**으로 처리됩니다. 여러 클라이언트가 동시에 명령을 실행하더라도 순차적으로 처리되어 **정확히 하나의 클라이언트만 락을 획득**할 수 있습니다.

<br>

### Redisson 라이브러리

Spring Boot의 기본 Redis 클라이언트인 Lettuce는 분산락을 제공하지 않기 때문에 직접 구현해서 사용해야합니다. 또한 스핀락 방식으로 동작하기 때문에 락을 획득할 때까지 반복 요청을 보내며, 대기 스레드가 많을수록 Redis 부하가 증가하는 단점이 있습니다.

반면 Redisson은 분산락 구현체를 제공햐 쉽게 사용할 수 있으며 락 해제 시 Redis Pub/Sub으로 대기 중인 클라이언트에게 알림을 보내므로 스핀락 방식보다 Redis 부하를 크게 줄일 수 있습니다.

<br>

## 코드 구현

Redis 분산락을 활용하여 Cache Stampede를 방지하는 코드를 구현해보겠습니다. 락 획득시 다른 스레드가 이미 캐시를 갱신했을 수 있으므로, 재확인로직을 추가했습니다.

```java
public Object process(String type, long ttlSeconds, Object[] args, Class<?> returnType,
                          OriginDataSupplier<?> originDataSupplier) throws Throwable {
    String key = generateKey(type, args); // 1

    String cachedData = redisTemplate.opsForValue().get(key); // 2

    if (cachedData == null) {
        RLock lock = redissonClient.getLock(key); // 3
        try {
            boolean lockable = lock.tryLock(5, 2, TimeUnit.SECONDS); // 4
            if (!lockable) { // 5
                return redisTemplate.opsForValue().get(key);
            }

            cachedData = redisTemplate.opsForValue().get(key); // 6
            if (cachedData == null) {
                return refresh(originDataSupplier, key, ttlSeconds); // 7
            }

            OptimizedCache optimizedCache = DataSerializer.deserialize(cachedData, OptimizedCache.class); // 8
            return optimizedCache.parseData(returnType);

        } finally {
            if (lock != null && lock.isHeldByCurrentThread()) { // 9
                lock.unlock();
            }
        }
    } else { // 10
        OptimizedCache optimizedCache = DataSerializer.deserialize(cachedData, OptimizedCache.class);
        return optimizedCache.parseData(returnType);
    }
}
```
1. 캐시 키를 생성합니다.
2. Redis에서 캐시 데이터를 조회합니다. 캐시가 존재하면 **10번 단계에서 바로 데이터를 반환**합니다.
3. 캐시 미스가 발생하면 Redisson의 `RLock`을 사용하여 분산락 객체를 생성합니다. 락의 키는 캐시 키와 동일하게 설정하여 같은 데이터에 대한 동시 접근을 제한합니다.
4. `tryLock(5, 2, TimeUnit.SECONDS)`: 락 획득을 시도합니다.
   - **waitTime (5초)**: 락 획득을 위해 대기할 최대 시간입니다.
   - **leaseTime (2초)**: 락의 자동 만료 시간(TTL)입니다. 락을 획득한 스레드가 비정상 종료되어도 2초 후 자동으로 락이 해제되어 데드락을 방지합니다.
5. waitTime 동안 락 획득에 실패하면 다른 스레드가 이미 캐시를 갱신했을 가능성이 있으므로 캐시를 다시 조회해 반환합니다.
6. 락 대기 중 다른 스레드가 이미 캐시를 갱신했을 수 있으므로, 불필요한 DB 조회를 방지하기 위해 캐시를 **재확인** 합니다.
7. **재확인** 후 캐시가 없다면 DB에서 데이터를 조회하고 캐시에 저장합니다.
8. **재확인** 후 캐시가 존재하면 데이터를 역직렬화하여 반환합니다.
9. `finally` 블록에서 락을 안전하게 해제합니다. `isHeldByCurrentThread()` 확인을 통해 현재 스레드가 락을 보유한 경우만 해제하여, 락을 획득하지 못했거나 TTL이 만료된 경우 잘못된 락 해제를 방지합니다.
10. 캐시 히트인 경우(첫 번째 조회에서 데이터가 존재), 락 없이 바로 데이터를 역직렬화하여 반환합니다.

<br>


### Logical TTL과 Physical TTL 전략

캐시가 만료될 때마다 락 경합이 발생하면 응답 시간이 길어질 수 있습니다. 이를 해결하기 위해 **Logical TTL**과 **Physical TTL**을 같이 적용해 볼 수 있습니다.

- **Physical TTL**: Redis에 실제로 설정되는 캐시 만료 시간입니다. 이 시간이 지나면 캐시 데이터가 삭제됩니다.
- **Logical TTL**: 캐시 데이터 내부에 저장되는 논리적 만료 시간입니다. Physical TTL보다 짧게 설정하여, 실제 캐시가 삭제되기 전에 미리 갱신할 수 있도록 합니다.

예를 들어 Physical TTL을 15초, Logical TTL을 10초로 설정하면 10초가 지났을 때 캐시는 여전히 Redis에 존재하지만 **논리적으로 만료된 것으로 간주**됩니다. 
이 시점에 요청이 들어오면 락을 획득하여 캐시를 갱신하게 되는데 이렇게 하면 실제 캐시가 완전히 만료되기 전에 미리 갱신할 수 있습니다. 또한 **Logical TTL이 만료되어도 캐시가 남아있으므로, 락 획득 실패 시에도 대기없이 기존 데이터를 즉시 반환**할 수 있습니다.

하지만 이 전략은 기존 데이터를 반환될 수 있어서 **실시간성이 중요한 데이터에는 주의가 필요**하며 이러한 경우에는 락 대기를 통해 최신 데이터를 확실히 조회하는 방식이 더 적합할 수 있습니다.

<br>
Logical TTL과 Physical TTL을 적용해 수정한 코드는 다음과 같습니다.
```java
public Object process(String type, long ttlSeconds, Object[] args, Class<?> returnType,
                          OriginDataSupplier<?> originDataSupplier) throws Throwable {
        String key = generateKey(type, args);

        //physical ttl 만료
        String cachedData = redisTemplate.opsForValue().get(key);
        if(cachedData == null){
            RLock lock = redissonClient.getLock(key);
            try {
                boolean lockable = lock.tryLock(5, 2, TimeUnit.SECONDS);
                if (!lockable) return redisTemplate.opsForValue().get(key);

                //double check
                cachedData = redisTemplate.opsForValue().get(key);
                if (cachedData == null) {
                    return refresh(originDataSupplier, key, ttlSeconds);
                }
                OptimizedCache optimizedCache = DataSerializer.deserialize(cachedData, OptimizedCache.class);
                return optimizedCache.parseData(returnType);
            } finally {
                if(lock!= null && lock.isHeldByCurrentThread()) lock.unlock();
            }
        } else { // 1
            OptimizedCache optimizedCache = DataSerializer.deserialize(cachedData, OptimizedCache.class); // 2
            if(optimizedCache.isNotExpired()){ // 3
                return optimizedCache.parseData(returnType);
            }

            RLock lock = redissonClient.getLock(key); // 4
            try {
                boolean lockable = lock.tryLock(0, 2, TimeUnit.SECONDS); // 5
                if (!lockable) return optimizedCache.parseData(returnType); // 6
                return refresh(originDataSupplier, key, ttlSeconds); // 7
            } finally {
                if(lock!= null && lock.isHeldByCurrentThread()) lock.unlock();
            }
        }
    }
```

1. Physical TTL은 남아있어서 캐시 데이터가 존재하는 경우입니다. Logical TTL을 확인하여 갱신이 필요한지 판단합니다.
2. 캐시 데이터를 `OptimizedCache` 객체로 역직렬화합니다. 이 객체에는 실제 데이터와 함께 Logical TTL 정보가 포함되어 있습니다.
3. `isNotExpired()`: Logical TTL이 아직 유효한지 확인합니다. 유효하면 기존 캐시 데이터를 그대로 반환하여 빠른 응답을 제공합니다.
4. Logical TTL이 만료되었으므로 캐시 갱신을 위해 락을 획득합니다. 이 시점에서는 Physical TTL은 아직 남아있어 캐시 데이터는 유효합니다.
5. `tryLock(0, 2, TimeUnit.SECONDS)`: 락 획득을 시도합니다. waitTime을 0으로 설정하여 락을 획득하지 못하면 즉시 실패를 반환합니다.
6. 락 획득에 실패한 경우, 현재 캐시 데이터(Logical TTL은 만료됐지만 Physical TTL은 유효)를 반환합니다. 이를 통해 락 대기 없이 빠른 응답이 가능합니다.
7. 락 획득에 성공하면 DB에서 최신 데이터를 조회하고 캐시를 갱신합니다. 새로운 Logical TTL과 Physical TTL이 함께 설정됩니다.