# 13.2

## 스레드 안전 상태
### 1. 불변(Immutable)

자바 문법의 불변에 해당하는 대표적인 키워드는 `final` 그리고 객체에는 **Record**가 있다.<br />
이 둘이 붙은 객체의 필드 변화는 스레드까지 갈 필요도 없이 애초부터 절대 엄금하고 있다.

<img alt="스크린샷 2024-12-02 오전 12 05 38" src="https://github.com/user-attachments/assets/566bf26e-1d30-412b-9dd7-377b678b928a" style="width: 45%;" />
<img alt="스크린샷 2024-12-02 오전 12 05 38" src="https://github.com/user-attachments/assets/e8208654-f7f2-49e7-bf07-79ebae451408" style="width: 45%;" />

자바 클래스에서 대표적인 불변 타입은 String, Long, Double, BigInteger, BigDecimal 등이 있다.<br />
근데 **AtomicInteger**나 **AtomicLong**은 불변이 아니다. 잠시 들여다보면...

```java
public class AtomicLong extends Number implements java.io.Serializable {
    
    private static final Unsafe U = Unsafe.getUnsafe();

    // ...

    public final long getAndIncrement() {
        return U.getAndAddLong(this, VALUE, 1L);
    }

    public final long incrementAndGet() {
        return U.getAndAddLong(this, VALUE, 1L) + 1L;
    }

    // ...
```

`AtomicLong` 클래스의 필드에 보면 `Unsafe`라는 타입이 확인된다.<br />
`Unsafe` 클래스는 메모리 관리 및 저수준 시스템 작업에 접근할 수 있으며, **원자적 연산을 위한 CAS(Compare And Swap) 명령**을 담당한다.<br />

```java
public final class Unsafe {

    // ...

    @IntrinsicCandidate
    public final long getAndAddLong(Object o, long offset, long delta) {
        long v;
        do {
            v = getLongVolatile(o, offset);
        } while (!weakCompareAndSetLong(o, offset, v, v + delta));
        return v;
    }

    // ...

    @IntrinsicCandidate
    public final boolean weakCompareAndSetLong(Object o, long offset,
                                               long expected,
                                               long x) {
        return compareAndSetLong(o, offset, expected, x);
    }

    // ...

    @IntrinsicCandidate
    public final native boolean compareAndSetLong(Object o, long offset,
                                                  long expected,
                                                  long x);
```

`AtomicLong` 클래스의 `getAndIncrement()` 메소드를 타고 들어가면 매커니즘이 아래와 같다.

>1. 현재 메모리 위치에서 저장된 값을 읽는다.<br />
>*자바는 직접 메모리 접근이 불가능하기 때문에 상대적인 메모리 주소 위치인 오프셋(`offset`)을 활용한다.*
>3. 예상한 값(`expected`)과 비교하여 값이 변경되지 않았다면 새로운 값(`x`)으로 변경한다.
>4. 예상한 값이 현재 값과 다르다면 연산이 실패해서 `false`를 반환한다.
>5. 실패한 연산일 경우 다시 값을 읽어와서 시도한다. 참고로 해당 메소드는 원래 값을 반환한다.

### 2. 절대적 스레드 안전

여러 스레드가 동시에 동일 객체나 자원을 사용해도 동기화 문제 혹은 경합 상태가 발생하지 않는 것을 의미한다.<br />
즉, 개발자 입장에서 동기화를 강제하는 등의 조치를 고려할 필요가 없는 상태다.<br />
불변 개념과 유사하지만, 수정이 가능하다는 점에서 차이가 있다.

<img width="863" alt="스크린샷 2024-12-02 오전 1 28 00" src="https://github.com/user-attachments/assets/22294af8-8a7b-4ed5-bf3c-eba5701c1d4d">

대표적인 예시인 `AtomicInteger` 클래스 인스턴스를 생성할 때 별도의 동기화 조치(`synchronized` 등)가 없어도 예상 결과가 나온다.

### 3. 조건부 스레드 안전

일반적으로 스레드 안전하다 생각하기 때문에 보통은 별도 조치가 필요없지만, **특정 조건**에서는 보호 조치가 필요할 수 있다.<br />
예를 들어, 특정한 방식으로 자원에 접근하거나 특정 시점에만 자원에 접근할 때 스레드 안전한 케이스가 있을 수 있다.<br />
이런 경우를 조건부 스레드 안전이라고 하며, 맵 구조의 동기화 보장 버전인 `HashTable`이나 리스트 구조의 동기화 보장 버전인 `Vector`가 있다.

테스트를 위해 커스텀 조건부 스레드 안전 클래스를 생성한다.<br />
아래의 클래스는 `count` 필드가 10000 미만에서는 스레드 안전하게 가산되지만 그 이후에는 보장되지 않는다.

```java
public class ConditionalSafety {

    private int count = 0;

    public void increment() {
        if (count < 10_000) {  // 조건이 만족되는 경우에만 안전
            synchronized (this) {
                count++;
            }
        } else {
            count++;
        }
    }

    public int getCount() {
        return count;
    }
}
```

이것을 바탕으로 스레드풀 서비스를 2번에 나눠서 연산 테스트를 해보면 아래처럼 결과가 나온다.

<img width="1076" alt="스크린샷 2024-12-02 오전 1 54 19" src="https://github.com/user-attachments/assets/9075047e-e7fb-4b0a-86f8-ff594a1e5142">

### 4. 스레드 호환

일반적으로 객체 자체가 스레드로부터 안전하지 않지만 조치를 통해 스레드 안전하게 만들 수 있는 경우다.<br />
웬만한 자바 클래스들이 이에 해당하며, 다른 말로는 **스레드 안전하지 않다**고 할 수 있다.<br />

### 5. 스레드 적대적

동기화 조치를 취해도 멀티 스레드 환경에서 안전하게 사용할 수 없는 경우를 말한다.<br />
보통 스레드 적대적인 코드의 말로는 데드락 상태를 유발하므로 유의해야 한다.

직접 데드락을 발생시켜보자. `Lock` 객체를 활용해서 두 개의 잠금 시나리오를 상정한다.

```java
private static class SharedResource {
    private final Lock lock1 = new ReentrantLock();  // 락 객체 1
    private final Lock lock2 = new ReentrantLock();  // 락 객체 2

    public void method1() {
        lock1.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " lock1 획득, lock2 잠금 시도중...");
            Thread.sleep(100);
            lock2.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " lock2 획득");
            } finally {
                lock2.unlock();
            }
        } catch (InterruptedException e) {
            System.err.println(Thread.currentThread().getName() + " interrupted");
        } finally {
            lock1.unlock();
        }
    }

    public void method2() {
        lock2.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " lock2 획득, lock1 잠금 시도중...");
            Thread.sleep(100);
            lock1.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " lock1 획득");
            } finally {
                lock1.unlock();
            }
        } catch (InterruptedException e) {
            System.err.println(Thread.currentThread().getName() + " interrupted");
        } finally {
            lock2.unlock();
        }
    }
}
```

위의 코드는 두 개의 잠금 객체를 필드로 가지는 전역 내부 클래스, `SharedResource`다.<br />
해당 클래스에는 `lock1` 필드를 잠그고 `lock2` 필드의 잠금을 얻으려는 메소드(`method1()`)와 그 반대인 메소드(`method2()`)가 존재한다.<br />
여기서 두 개의 스레드를 생성해서 동일 `SharedResource` 인스턴스의 각 메소드 작업을 부여해서 호출하는 테스트를 진행한다.<br />
10초가 지나도 결과가 나오지 않으면 데드락으로 간주하고 타임아웃을 설정한다.

<img width="1246" alt="스크린샷 2024-12-02 오전 2 28 46" src="https://github.com/user-attachments/assets/ff10dfe1-8bfa-490d-8e82-7963a5609ba2">

데드락이 발생한 이유는 아래와 같다.
>1. 스레드 1이 `lock1`을 잠근다.
>2. 스레드 2가 `lock2`를 잠근다.
>3. 스레드 1은 `lock1`을 획득한 후, `lock2`를 잠그려고 시도한다.<br />
>하지만 `lock2`는 이미 스레드 2가 보유하고 있기 때문에, 스레드 1은 `lock2`를 무한히 기다린다.
>4. 이는 스레드 2에서도 마찬가지다. `lock2`를 획득한 상태에서 `lock1`을 잠그려고 시도하지만 <br />
>역시나 스레드 1이 선점했기 때문에 무한히 기다린다.


## 스레드 안전성 보장

### 1. synchronized vs java.util.concurrent.locks.Lock

| 특성                       | `synchronized`                                 | `Lock` 인터페이스                             |
|----------------------------|-----------------------------------------------|--------------------------------------------|
| **사용법**                  | 코드 블록이나 메서드에 키워드 사용             | `lock()`와 `unlock()` 메서드로 락 관리        |
| **락 제어**                 | 자동 락 관리 (암시적)                         | 수동 락 관리 (명시적)                         |
| **락 획득 실패 처리**        | 불가능                                       | `tryLock()`을 사용해 실패를 처리할 수 있음   |
| **교착 상태 처리**          | 교착 상태를 직접 관리하기 어려움              | 교착 상태를 관리할 수 있는 기능 제공        |
| **인터럽트 처리**           | 인터럽트 처리 불가능                         | `lockInterruptibly()`로 인터럽트 가능        |
| **성능**                    | JVM에서 제공하는 기본 기능이므로 성능이 우수  | 유연한 기능을 제공하나 약간의 성능 오버헤드가 있음 |

### 2. 나름의 성능 비교 테스트

문득 저 둘의 성능이 궁금해서 간단하게 JUnit으로 테스트를 해봤다.<br />
사실 예상은 `Lock` 인터페이스가 조금 더 괜찮지 않을까 싶었으나... 스레드풀 서비스 선택 전략에 따라 다르게 나타났다.

<img width="944" alt="스크린샷 2024-12-02 오전 3 10 23" src="https://github.com/user-attachments/assets/c81ca7d2-8618-4ded-a6cc-c0c15490233c">

위의 테스트 케이스는 `newFixedThreadPool(THREAD_COUNT)`를 기반으로 스레드들을 생성한 경우다.<br />
이것은 내 예상대로 `Lock` 인터페이스 기반이 근소하게나마 우위를 차지했다.

<img width="878" alt="스크린샷 2024-12-02 오전 3 06 30" src="https://github.com/user-attachments/assets/57d76de8-9d00-48ba-94e4-7274916c0255">

반면, 이것은 `newVirtualThreadPerTaskExecutor()`을 기반으로 스레드들을 생성한 경우다.<br />
이 경우에는 `synchronized` 기반이 우위를 차지했다.

내 생각에는 원칙적으로는 락 경합 상태에서 `synchronized`는 컨텍스트 스위칭 비용이 들지만 `Lock`에서는 해당 구현체인 `ReentrantLock`이 스핀 락과 대기 큐를 효율적으로 관리하므로 실행 시간이 더 짧게 나오지만, 가상 스레드가 OS 비용에서 더 효율적이기 때문에 JVM 내부에서 최적화가 잘 딘 `synchronized`가 간단한 반복 연산에 대해서 우위를 차지하는 것이 아닌가 싶다. 실행 시간을 비교하는 게 아닌 메모리 사용량 등을 비교하는 것이 더 의미가 있겠다...(사실 다른 테스트들도 그렇겠지만)

