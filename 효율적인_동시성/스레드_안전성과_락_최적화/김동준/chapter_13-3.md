# 13.3

## 1. 락 최적화

### (1) 스핀 락
스레드가 락을 획득할 때까지 **반복적으로 시도**하는 락<br />
잠깐의 대기 시간 동안 락을 획득하는 경우에 효율이 좋은 편이다.<br />
다만 결국 락 획득을 시도한다는 것이 CPU를 계속 점유하므로 과소비의 우려가 있다.

### (2) 락 제거
락 자체가 결국 동기화를 위한 수단이므로 리소스를 소모한다.<br />
그렇기 때문에 단일 스레드 혹은 락이 불필요한 상황에서 굳이 락을 유지할 필요가 없다.<br />

직접 코드로 확인해보고 싶었는데, 정확히는 JVM이나 JIT나 실행 시점에 락을 제거한다고 한다.<br />
그래서 JVM 로그를 확인해보는 방법을 먼저 익혀야 한다...(어우 어려워)

### (3) 락 범위 확장

여러 동기화 작업을 하나의 락으로 통합하여 처리하는 방법이다.<br />

```java
private int count = 0;

synchronized void add1() {
  count++;
}

synchronized void add2() {
 count++;
}
```

스레드에서 `count` 변수의 가산을 위해 위의 스레드가 필요로 한다고 가정하자.<br />
위 코드에 따르면 각각의 메소드 호출에 대하여 스레드 당 2번씩 동기화가 발생한다.<br />
이것을 하나로 묶어본다.

```java
private int count = 0;

synchronized void addAll() {
  add1();
  add2();
}

void add1() {
  count++;
}

void add2() {
 count++;
}
```

위의 코드는 두 개의 `synchronized` 메소드를 하나로 묶은 예시가 된다.<br />
이런 경우여도 똑같이 안전한 가산이 가능하며 스레드 당 1번씩만 동기화가 발생하기 때문에 효율성을 챙길 수 있다.<br />
이런 케이스를 **락 범위 확장**이라고 한다. 다만, 과도하게 사용하면 병목 현상이 유발될 수 있다.

실제로 위 두 코드를 기반으로 실행시간 테스트를 해보면 락 범위 확장 코드가 근소하게 우위를 차지한다.<br />
애플리케이션 레벨에서도 차이가 발생하는데 저수준 고성능 상황에서는 더욱 차이가 심할 것으로 추측.

<img width="931" alt="락 범위 확장 커스텀 테스트" src="https://github.com/user-attachments/assets/629be30e-dbef-4dd7-b578-15f39272efe7">


### (4) 경량 락

락 경쟁은 필연적으로 리소스 소모를 유발하는데, 이를 줄이기 위해 고안된 매커니즘 중 하나가 경량 락이다.<br />
기존의 `synchronized`나 `ReentrantLock` 등은 락 획득을 위한 자원 소모 경쟁을 유발하게 된다.<br />
경량 락은 이런 통상적으로 락이 붕 뜨는, 대기 시간에 효율적으로 리소스를 활용하기 위한 동적인 락 전환이 기본 골자인데, 보통 락 경쟁이 심화될 수록 경량 락에서 중량 락으로 전환되며 이런 동적인 락 전환에는 `Atomic` 관련 클래스의 원자적 연산에서 봤던 CAS(Compare And Swap)를 기반으로 이루어진다.

사실 우리가 신경 쓸 필요는 없는 편이긴 하다. Java 9 이후부터는 JVM이 자동으로 최적화해주기 때문...

### (5) 편향 락

얘는 더하다. 아예 15버전에서 `deprecated` 선언이 됐기 때문...<br />
그래도 간단히 개념만 알고 넘어가자면 편향 락은 단일 스레드가 특정 객체에 대해 지속적으로 락을 획득하는 경우 성능을 최적화하는 방식 정도로만...<br />
참고로 왜 삭제됐냐면, 얘는 경쟁이 작을 수록 효율적인데 경쟁이 작을 수록 락을 쓸 의미가 없지 않나..? 라는 생각이 든다.<br /> 
>*참고 : https://openjdk.org/jeps/374*

## 2. 락 커스터마이징

사실 JVM에서의 개념을 아는 게 스터디의 취지지만,<br />
동시성에 많은 관심이 있기도 했고 락 기법에 대해 개념을 잘 숙지해두면 좋을 것 같아서 나름의 커스터마이징을 해봤다.

### (1) 스핀 락

스핀 락은 위에서 말했듯, 자원을 얻기 위해 지속적으로 획득 시도(이게 흡사 빙빙 돈다는 이미지라서 스핀 락)를 하는 락이다.<br />
다음 락을 얻기 위한 시도 때문에 스핀 락에는 스레드 전환 시간이란 게 존재하지 않는다. 그래서 구현이 상당히 심플하다.

```java
public class SpinLock {

    private final AtomicBoolean isLocked = new AtomicBoolean(false);

    public void lock() {
        while (isLocked.compareAndSet(false, true)) {
            System.out.println("아직 다른 스레드가 락을 점유 중...");
        }

        System.out.println("락 획득");
    }

    public void unlock() {
        isLocked.set(false); 
        System.out.println("락 해제");
    }
}
```

>1. `isLocked` 변수는 `SpinLock`의 잠금 상태를 나타낸다.
>2. `lock()` 메소드에서는 락을 획득하려고 반복적으로 시도한다. 만약 `isLocked`가 `false`일 경우, <br />
>락이 해제된 것이니 `true`로 바꾸고 락을 획득. 만약 이미 `true`라면, 락이 점유 중이라는 의미이므로 반복해서 시도
>4. `unlock()` 메소드는 락 해제를 설정하고 있다.

<img width="823" alt="커스텀 스핀 락 테스트" src="https://github.com/user-attachments/assets/83c1c923-fc7e-4cc8-ac35-93a5c84a0396">

참고로 자바의 재진입 락(`ReentrantLock`)과 동작 매커니즘이 유사하다.<br />
스핀 락이 반복적으로 획득을 시도하는 것처럼 재진입 락 역시 락을 획득한 스레드가 다시 락 획득 시도가 가능하다.<br />
차이점은, 재진입 락에서는 락 획득에 실패한 경우 스레드가 **대기** 상태로 돌아가는 것과 그로 인한 **컨텍스트 스위치** 비용이 발생한다는 점이다.

### (2) 읽기 / 쓰기 락

읽기 작업에 대해서는 다수의 접근을 허용하면서 쓰기 작업에만 독점적인 접근만을 허가하는 락 방식이다.<br /> 
설명에서 나와있듯이, 많은 읽기 작업에 비해 적은 쓰기 작업에서 성능이 좋으며 읽기 작업과 쓰기 작업은 배타적이어야 한다.<br /> 
무슨 말이냐면 쓰기 작업이 완료된 후에야 읽기 작업들을 수행할 수 있고, 읽기 작업들이 전부 완료돼야 쓰기 작업이 진행될 수 있다.<br />
읽기 락은 여러 스레드가 동시에 읽을 수 있도록 허용해야 하지만, 쓰기 락은 단일 스레드만 접근할 수 있도록 해야 한다.

jdk 21 기준으로 `ReadWriteLock` 인터페이스를 구현하는 `ReentrantReadWriteLock`에 정적 내부 클래스로 `WriteLock`과 `ReadLock`이 있다.<br />
얘네들은 상호 배타적으로 동작하면서 충돌이 상정되기 때문에 동시에 하나의 리소스에 대하여 접근을 허용하지 않는다. 즉, 동시에 읽기와 쓰기가 불가능하다.<br />
간단하게 커스텀 코드로 `ReentrantReadWriteLock` 인스턴스를 받아와 얘네를 기반으로 읽기 / 쓰기 락을 커스터마이징 해본다.

```java
public class CustomReadWriteLock {
    private int data;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();  // 읽기 락
    private final Lock writeLock = lock.writeLock();  // 쓰기 락

    public CustomReadWriteLock(int initialValue) {
        this.data = initialValue;
    }

    public int read() {
        readLock.lock();  // 읽기 락 획득
        try {
            return data;
        } finally {
            readLock.unlock();  // 읽기 락 해제
        }
    }

    public void write() {
        writeLock.lock();  // 쓰기 락 획득
        try {
            data++;
        } finally {
            writeLock.unlock();  // 쓰기 락 해제
        }
    }

    public int getData() {
        return data;
    }
}
```
<img width="872" alt="스크린샷 2024-12-03 오전 12 37 01" src="https://github.com/user-attachments/assets/07623077-3537-45eb-86a8-62a641cfbf1b">

로그가 뒤죽박죽이고 읽기 스레드의 값이 다르게 나오는 이유는 애시당초 `ExecutorService` 기반으로 동작시키며 병렬 실행되기 때문이다.<br />
즉, 스레드들은 섞여 실행되지만 내부는 `ReentrantWriteReadLock` 때문에 안전하게 데이터 증가 연산이 원자적으로 이뤄질 수 있는 것이다.

### (3) 네임드 락
공유 리소스 자원에게 이름을 붙여, 스레드가 이름을 참조했을 경우에 락을 힉득하게 된다.<br />
이름 기반으로 동시성이 관리되기 때문에 다수의 충돌에도 특정 리소스의 보호 정도가 강하다.<br />
단점은 결국 이름 기반으로 리소스를 관리하기 때문에 데드락 가능성이 커진다.<br />
아래는 네임드 락의 커스텀 코드 및 순서도다.

```java
public class NamedLock {
    private final Map<String, AtomicBoolean> lockMap = new HashMap<>();
    private int count;

    public NamedLock(int count) {
        this.count = count;
    }

    public boolean referenceName(String lockName) {
        while (lockMap.containsKey(lockName)) {
            System.err.println(Thread.currentThread().getName() + " 이름 이미 참조 중");
            Thread.yield();
        }
        lockMap.putIfAbsent(lockName, new AtomicBoolean(false));
        return true;
    }

    public boolean lock(String lockName) {
        AtomicBoolean lock = lockMap.get(lockName);
        return lock != null && lock.compareAndSet(false, true);
    }

    public void unlock(String lockName) {
        AtomicBoolean lock = lockMap.get(lockName);
        if (lock != null) {
            lock.set(false);
        }
    }

    public void dereferenceName(String lockName) {
        lockMap.remove(lockName);
    }

    public void execute(String lockName) {
        if (referenceName(lockName)) {
            try {
                while (!lock(lockName)) {
                    // 락을 획득하지 못했을 때, 다른 스레드에게 CPU 양보
                    System.err.println(Thread.currentThread().getName() + " 락 획득 실패");
                    Thread.yield();
                }
                try {
                    System.out.println(Thread.currentThread().getName() + " 락 획득");
                    count++;
                } finally {
                    unlock(lockName);
                    System.out.println(Thread.currentThread().getName() + " 락 해제");
                }
            } finally {
                dereferenceName(lockName);
                System.out.println("이름 참조 해제");
            }
        } else {
            System.err.println("이름 참조 실패.");
        }
    }

    public int getCount() {
        return count;
    }
}
```

>1. 한 스레드가 이름을 참조해서 작업을 수행한다
>2. 한 스레드가 작업을 수행하려하지만 아직 이름을 참조하지 못해서 작업을 수행 못한다
>3. 한 스레드가 이름을 참조했으나 다른 스레드가 작업 중이라 작업을 수행 못한다

해당 코드의 테스트 결과는 아래와 같다.

<img width="988" alt="스크린샷 2024-12-04 오후 9 04 52" src="https://github.com/user-attachments/assets/7dfea5c8-232d-4ec3-913e-6576a6f781da">
