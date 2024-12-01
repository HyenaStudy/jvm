# 13.2

## 불변(Immutable)

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
