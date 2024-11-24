# 12.3

## 자바에서의 메모리 구조

`멀티 프로세서 <-> 캐시 <-> 메인 메모리` 구조처럼 자바에서의 메모리 구조도 `멀티 스레드 <-> 작업 메모리 <-> 메인 메모리`로 구성된다.<br />
물론 작업 메모리가 캐시와 완벽히 대응되는 구조는 아니다. 정확히는 **스레드의 로컬 메모리**에 해당한다.<br />
임시 저장소의 역할을 하며 동기화가 필요하다는 점은 같지만 캐시와의 차이점이 많은 편이다.

| **특징**            | **작업 메모리**                          | **캐시**                                   |
|---------------------|----------------------------------------|------------------------------------------|
| **개념**             | JVM의 논리적 모델 (소프트웨어 레벨)         | CPU 하드웨어의 물리적 메모리 계층             |
| **역할**             | 스레드가 메인 메모리와 독립적으로 작업하도록 지원 | CPU가 메인 메모리 접근 시간을 줄이기 위해 사용 |
| **위치**             | JVM 내부 (스택, 레지스터, 등)             | 프로세서 내부 또는 외부 (L1, L2, L3 캐시)       |
| **데이터 동기화**     | JMM 규칙에 따라 메모리 동기화 수행           | 캐시 일관성(Coherence) 프로토콜로 동기화 관리   |

### 1. 멀티 스레드에서 메모리 간 상호작용

>1. 메인 메모리에서 작업 메모리로 변수 복사
>2. 작업 메모리 내용 다시 메인 메모리로 동기화

위의 단계는 `잠금 -> 잠금 해제 -> 읽기 -> 적재 -> 사용 -> 할당 -> 저장 -> 쓰기` 순으로 이뤄진다.<br />
다만 이 순서는 단일 메모리의 관점에서의 작업 순서이고 복수의 메모리 간에서는 뒤죽박죽 섞인 형태다.<br />
JMM에서는 **스레드의 작업 메모리와 메인 메모리 간 동기화가 명확하게 보장되지 않는다.**

> 1. A 스레드가 a 변수를 읽음
> 2. B 스레드가 b 변수를 읽고 연산하고 씀
> 3. A 스레드는 아직 연산하고 쓰는 작업 x

즉, JMM의 비결정적 실행 순서 및 동기화 부족으로 이런 일이 발생 -> **동시성 이슈**

### 2. 왜 이따구로 설계됐나?

#### (1) 작업 메모리의 존재

사실 이런 동시성 이슈를 해결하는 간단한 방법은, **직접 메인 메모리에 접근**하는 것.<br />
다만 이러면 결국 작업 메모리의 캐시로써의 의의가 사라지게 된다.<br />
또한, 스레드 간 순서 상호작용 강제 동기화는 상상을 초월하는 리소스를 소모한다. -> 엄청난 성능저하 유발

#### (2) 굳이 동기화를 보장할 필요가 없음

개발자가 필요한 부분에만 동기화를 강제하면 된다.<br />
동기화의 강제로 인한 병렬 처리의 장점을 잃을 바에는 필요한 곳에서만 동기화를 구축하면 된다.<br />
즉, **성능**과 **확장성**을 위해서이다.

## volatile 키워드

### 1. 개념 정리

#### (1) 정의 및 용도

`volatile` 키워드는 **자바 코드의 변수를 메인 메모리에 저장할 것을 명시**한다.<br />
메인 메모리와 작업 메모리 간의 데이터 일관성을 보장하고, 변수의 쓰기 작업이 즉각 메인 메모리에 반영케 한다.<br />
두 개의 스레드가 `counter`라는 변수를 읽고 있다고 생각해보자.

![image](https://github.com/user-attachments/assets/69176b86-55d0-4b2a-b3ea-5ba799aa53fe)

스레드 1이 `counter` 변수를 0에서 7로 바꿔도, 스레드 2는 이를 모른다. 이를 **가시성의 문제**라고 한다.<br />
여기서 `counter` 변수에게 `volatile` 키워드를 부여하면 메인 메모리와 작업 메모리의 변수 값이 동일해질 수 있다.<br />
즉, 단일 스레드 상황에서는 메인 메모리와 작업 메모리 간의 변수 일관성을 유지할 수 있게 된다.<br />

또한, **명령어 재정렬을 방지**한다. JVM과 CPU는 최적화를 위해 명령어 실행 순서를 조정할 수 있다.<br />
`volatile` 키워드는 메모리 배리어를 삽입하여 읽기 -> 쓰기 순서를 보장하게 된다.

#### (2) 동시성 이슈의 해결책?

다만 이것을 멀티 스레드 상황으로 넓히면 상황이 복잡해진다.<br />
아래의 상황은, 멀티 스레드에서 메인 메모리의 `a` 변수에 접근하면서 연산(처리)하는 상황이다.

![image](https://github.com/user-attachments/assets/514a8383-a131-4600-8764-e811798307be)

이것을 순전히 `volatile` 키워드만 쓰면, 단일 스레드 관점에서는 변수의 최신화는 보장된다.<br />
다만 그것이 복수의 스레드가 동시 접근하는 **경합 상황**에서는 전혀 소용이 없다.<br />

>1. 스레드1이 `a` 변수에 접근하여 값을 읽는다.
>2. 스레드2도 `a` 변수에 접근하여 값을 읽는다.
>3. 스레드1이 `a` 변수에 +5 연산을 수행한다.
>4. 스레드1이 `a` 변수를 메인 메모리에 쓴다.
>5. 스레드2가 `a` 변수에 +15 연산을 수행한다.
>6. 스레드2가 `a` 변수를 메인 메모리에 쓴다.
>7. 이제 `a` 변수는 20이 아니라 15로 업데이트된다.

#### (3) 예제 코드

직접 코드로 구현해서 `volatile` 키워드가 어떤 의미를 가지는지 확인해보자.<br />

```java
public class VolatileTest {
    private static boolean running = true;

    public static void main(String[] args) {

        // 스레드 1: running 값에 따라 동작
        Thread thread1 = new Thread(() -> {
            while (running) {
                /**
                 * 여기에 sout 같은 로직을 넣으면 thread1이 sout를 호출하는 순간 CPU 양보
                 * 그 틈에 main 스레드가 실행될 수 있는 기회를 가지고, running -> false 할당
                 * 그래서 루프 종료
                 */
            }

            System.out.println("Thread 1: running 값 변경 감지, 루프 종료");
        });

        thread1.start();

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println("Main Thread: 2초 후, running 값 false 변경");
        running = false;

        // 스레드 종료 대기
        try {
            thread1.join();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println("Main Thread: 모든 작업 완료");
    }
}
```

>#### 예상 시나리오
>1. main 스레드와 thread1 스레드가 있다.
>2. 2초 후, `running` 변수가 `false`로 바뀐다.
>3. 그로 인해 thread1의 로직이 종료되고 main도 종료된다.

하지만 실행 결과는 아래처럼 `running` 변수가 `false`로 변해도 무한히 앱이 동작한다. 즉 스레드가 종료되지 않는다.

<img width="540" alt="스크린샷 2024-11-24 오후 4 19 50" src="https://github.com/user-attachments/assets/04ac1f65-ccf2-4715-a7a4-fefaac47c11d">

이유는, 하나의 스레드(`main`)에서 값을 변경한 후 다른 스레드(`thread1`)가 그 값을 즉시 읽지 못하기 때문이다.<br />
**캐시 일관성 문제**로, thread1에서 `running` 값이 `false`로 변경돼도, thread1이 `running` 값을 캐시로 읽어와서 계속 `true`로 인식하는 것이다.

이제, 여기서 `ruuning` 변수에 `volatile` 키워드를 부여해보고 다시 실행해보자.
```java
public class VolatileTest {
    private static volatile boolean running = true;

    // ...
```
<img width="545" alt="스크린샷 2024-11-24 오후 4 25 59" src="https://github.com/user-attachments/assets/4b8fa860-81a7-451a-8b6b-77276717244d">

`Process finished with exit code 0`, 즉 오류나 외부 개입 없이 앱이 정상적으로 종료됐다.<br />
`volatile` 키워드로 메인 메모리에서 읽기 및 쓰기 연산을 하도록 보장돼서 예상 시나리오대로 동작했다.

이제 이것을 복수의 스레드, 즉 멀티 스레드 환경으로 시나리오를 확장해보자.

```java
//  595p 코드 12-1 수정 ver

public class VolatileTest {
    public static volatile int race = 0;

    public static void increase() {
        race++;
    }

    private static final int THREADS_COUNT = 20;

    public static void main(String[] args) {
        Thread[] threads = new Thread[THREADS_COUNT];
        System.out.println("스레드 카운트 : " + THREADS_COUNT);

        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    increase();
                }
            });
            threads[i].start();
        }

        for (int i = 0; i < THREADS_COUNT; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                System.err.println(e.getMessage());
            }
        }

        // 예상 : 200000
        System.out.println("결과 : " + race);
    }
}
```

`Thread.activeCount()`는 현재 JVM 내의 모든 스레드 개수를 반환하게 된다.<br />
만약 별개의 클래스에서 구현하면 메인 스레드는 종료되지 않으므로 무한루프에 걸리게 된다.<br />
즉, 모든 스레드가 종료됐는지 여부만 판별하고 특정 스레드의 세부적인 종료 여부는 고려하지 않기 때문에 적절한 메소드가 아니라고 생각했다.<br />
또한, `Thread.yield()`는 다른 스레드의 종료 대기를 보장하지 않음. 반대로 `join()`은 다른 스레드의 종료 대기를 보장한다.

아무튼 예상 시나리오를 확인하고 결과를 확인해보자.

>1. 20개의 스레드가 할당된다.
>2. 각 스레드는 `increase()` 메소드를 10000번 호출한다.
>3. 그로 인해 `race` 변수가 200000으로 업데이트된다.

<img width="732" alt="스크린샷 2024-11-23 오후 6 26 27" src="https://github.com/user-attachments/assets/6c233412-df07-4e29-9256-baa9a540326c">

`volatile` 키워드가 붙여졌음에도, 멀티 스레드 환경에서는 동시성 문제가 해결되지 않는 것을 확인할 수 있다.<br />
즉, 메모리 가시성 문제에는 `volatile` 키워드로 충분히 커버되지만 동시성 문제를 해결하기에는 미흡하다.

위의 이유는 `increase()` 메소드의 작업 구조 때문이다.<br />
`race` 변수를 가져오는 **읽기 작업**과 , 즉 읽어온 `race` 변수를 +1하는 연산인 **쓰기** 작업이 분리됐다.<br />
`volatile` 키워드는 **읽기**에 있어서는 메인 메모리에서 가져오도록 보장시켜준다. 그렇지만 **쓰기**에는 아무런 관여가 없다.<br />
이 두 작업이 **원자적이지 않기 때문**에 동시성 문제는 여전히 남아있는 것이다.

## synchronized 키워드

`volatile`은 메인 메모리의 값 읽기와 쓰기를 즉시 반영하도록 보장하지만, 멀티 스레드에서 동시에 객체에 접근할 경우 데이터의 일관성을 유지하지 못한다.<br />
여기서 확인할 수 있는 근본적인 문제의 키워드는 **멀티 스레드가 객체에 동시에 접근하게 되는 것**이다.
이를 해결하려고 **하나의 스레드가 객체를 사용하는 동안 다른 스레드의 접근을 차단하고, 데이터의 일관성과 동기화를 보장**하는 것이 `synchronized` 키워드의 역할이다.

![image](https://github.com/user-attachments/assets/b3d307c6-1fd4-4fd0-ae9d-da10cbc3b4bd)

자바의 모든 객체는 **내재적 잠금**을 포함하며, 이것이 객체에 대한 동기화 접근을 제어한다.<br />
여기서 `synchronized` 블록 혹은 메소드가 호출될 때, 스레드가 해당 객체의 잠금을 해제해야 블록 내 코드를 실행할 수 있다.<br />
이런 `synchronized` 키워드는 **모니터 락** 혹은 **내재적 락**이라는 메커니즘을 통해 동작한다.<br />

*참고로, 모든 객체가 내재적 잠금을 갖고 있어 `synchronized` 키워드로 잠금이 구현되는 자바와 달리 C#은 `Monitor` 클래스를 기반으로 직접 구현한다.*

메소드 혹은 동기화 강제가 필요한 코드에 `synchronized` 키워드를 붙여서 구현할 수 있다.

```java
# 동기화 메소드
public static synchronized void increase() {
    race++;
}

# 동기화 블록
public static void increase() {
    synchronized (SynchronizedTest.class) {race++;}
}
```
<img width="542" alt="스크린샷 2024-11-24 오후 7 53 27" src="https://github.com/user-attachments/assets/b9a6a194-009e-4e55-9a1f-78baecf5b5d8">

위처럼 동기화가 강제되면서 우리가 원하는 값을 도출할 수 있다.<br />

참고로, 동기화 블록이 동기화 메소드보다 성능상 우위를 차지할 수 있다.<br />
동기화 강제 작업이 필요한 연산에만 선별적으로 적용할 수 있기 때문이다.

```java
public class SynchronizedTest {
    public static int race = 0;
    private static final int THREAD_COUNT = 20;
    private static final int ITERATIONS = 1_000_000;

    public synchronized static void methodSync() {
        for (int i = 0; i < 100; i++) {}

        race++;
    }

    public static void blockSync() {
        for (int i = 0; i < 100; i++) {}

        synchronized (SynchronizedTest.class) {
            race++;
        }
    }

    public static void main(String[] args) {
        long methodSyncTime = measureExecutionTime(() -> {
            for (int i = 0; i < THREAD_COUNT; i++) {
                new Thread(() -> {
                    for (int j = 0; j < ITERATIONS; j++) {
                        methodSync();
                    }
                }).start();
            }
        });

        long blockSyncTime = measureExecutionTime(() -> {
            for (int i = 0; i < THREAD_COUNT; i++) {
                new Thread(() -> {
                    for (int j = 0; j < ITERATIONS; j++) {
                        blockSync();
                    }
                }).start();
            }
        });

        System.out.println("동기화 메소드 실행 시간: " + methodSyncTime);
        System.out.println("동기화 블록 실행 시간: " + blockSyncTime);
    }

    public static long measureExecutionTime(Runnable runnable) {
        long start = System.nanoTime();
        runnable.run();
        long end = System.nanoTime();
        return end - start;
    }
}
```
<img width="540" alt="스크린샷 2024-11-24 오후 8 16 22" src="https://github.com/user-attachments/assets/3bb754dd-2342-42d5-abbc-0dd977e6f3cc">





## 선 발생 원칙

---

*출처*
*https://qkrqkrrlrl.tistory.com/174*
*https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html*
