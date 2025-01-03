# VisualVM 기반 트래픽 테스트 시나리오 분석

메모리를 직접 모니터링하고 성능 향상을 할 수 있는 방법을 고민하기 위해 직접 ~~머리박치기를 통해~~ 테스트 시나리오를 작성하고 테스트를 하였다.

테스트 시나리오는 아래와 같으며, 총 4가지의 시나리오를 분석할 예정이다.

>1. 빈 초기화 지연 이슈
>2. 메모리 누수로 인한 객체 회수 누락
>3. 스레드 무분별 생성 및 대기
>4. GC 옵션에 따른 동작 분석

## Issue 1. 빈 초기화 지연 이슈

빈은 스프링 컨테이너에 의해 관리되는 재사용 가능한 소프트웨어 구성요소(컴포넌트)를 의미한다. 이 빈들은 애플리케이션이 부팅되면서 IoC 컨테이너가 자동으로 생성하고 의존성 주입을 처리하는 초기화 작업이 수행된다.

이런 빈의 초기화에는 다양한 전략이 있고, 또 그것들을 반영하는 `@PostConstruct` 어노테이션, `InitializingBean` 인터페이스 구현체 등의 다양한 내용이 있으며, 그중에는 **지연 초기화 전략**도 포함된다.

### 1. 빈 초기화 지연

스프링에서는 `@Lazy` 어노테이션을 제공하는데 이를 통해 스프링 빈의 초기화를 지연시키면서 불필요한 자원 사용을 어느 정도 막고 앱의 성능을 최적화할 수 있다.

다만 무조건 만능은 아닌 게, 결국 빈이 초기화가 늦어지는 것은 의존성 주입을 위한 로딩이 늦춰지게 되는데 이 과정에서 성능 저하가 발생할 수 있으며 메모리에 영향을 끼칠 수 있다.

요약하자면, 일반 빈은 애플리케이션이 부팅할 때 초기화가 되기 때문에 사용 시점에서는 그냥 사용만 하면 되지만, `@Lazy` 어노테이션 적용 빈은 사용 시점에서 **초기화 + 사용**이 같이 이뤄지기 때문에 사용이 빈번해질 수록 메모리에 부담이 간다.

다만 이것은 이론이고, 트래픽이 몰리는 상황에서의 실제 결과는 또 다를 수 있으니...

### 2. 테스트 세팅

테스트 코드는 아래와 같다. `@Lazy` 어노테이션이 부여 여부에 따라 동일 환경에서 테스트를 2번 실행한다.

```java
@Slf4j
@Component
@Lazy // 스프링은 즉시 초기화가 디폴트지만, 얘는 지연 초기화 어노테이션
public class LazyInitBean {

    public void performTask() {
        log.info("*** LazyInitBean 작업 수행 ***");
    }
}
```
```java
@RestController
@RequestMapping("/lazy")
@RequiredArgsConstructor
public class LazyInitController {

    private final LazyInitBean lazyInitBean;

    @GetMapping("/test")
    public String testLazyInit() {
        lazyInitBean.performTask();
        return "느릿느릿 빈 초기화 + 작업 수행";
    }
}
```

테스트 환경은 아래와 같다.

>- 트래픽 발생 툴 : JMeter
>- 가상 사용자 수 : 200
>- 램프업 타임 : 30s
>- 루프 카운트 : 30
>- 계측 도구 : IntelliJ Profiler

사실 메모리 계측을 VisualVM으로 수행하려 했으나 왜인지 프로파일러 결과값이 나오질 않았다(...) 대략 5시간 가까이 끙끙 앓았으나 결국 포기하고 IntelliJ Ultimate Edition에서 제공하는 프로파일러로 메소드별 메모리 할당량 계측으로 선회...

### 3. 테스트 진행 및 결과

#### (1) JMeter 설정

<img width="75%" alt="빈초기화테스트스레드그룹" src="https://github.com/user-attachments/assets/e641f084-8e89-432f-930d-5c531fa05c48" />

#### (2) Lazy 어노테이션 적용(지연 초기화)

<img width="75%" alt="지연초기화메모리할당랼" src="https://github.com/user-attachments/assets/f840c41b-4c4c-412e-8ca4-05f112e5bd06" />

#### (3) Lazy 어노테이션 미적용(즉시 초기화)

<img width="75%" alt="즉시초기화메모리할당량" src="https://github.com/user-attachments/assets/c9145aa8-d0ac-4c6f-9fb5-7dc1e38ff232" />

### 4. 테스트 분석

이론과 다르게 실제 메모리 할당량은 거의 차이가 없었다.

개인적인 고찰 결과, 트래픽 테스트로는 빈 초기화의 영향력을 확인하는 것이 어려울 것으로 생각됐다. 애시당초 트래픽 테스트는 애플리케이션의 동시 처리 능력에 더 집중하는 경향이 높은 것과 별개로 **빈은 한 번 초기화가 이뤄지면 그걸로 끝**이기 떄문에 총체적인 성능에 영향을 미치지는 않는 것이다.

그렇기 때문에 실제 프로젝트에서 `@Lazy` 어노테이션은 **초기화 비용이 비싼 빈**이나 **활용이 매우 드문 빈**에 적용하는 수준으로 고려하면 충분할 듯하다.

## Issue 2. 메모리 누수로 인한 객체 회수 누락

기본적으로 사용이 끝난(사망 판정을 받은) 객체는 해제되어야 한다. 왜냐면 사용 가치가 없는데 메모리에 해당 객체를 남겨두는 것은 곧 메모리 낭비가 되기 때문이다.

그렇기 때문에 사용 가치가 없는 객체를 적재적소에 정리하는 것은 매우 중요하며, 자바 진영에서는 **가비지 컬렉터**가 그 역할을 담당한다. 그럼에도 특정 상황에서는 GC가 객체를 회수하지 못하는 상황이 발생할 수도 있다.

### 1. 메모리 누수

전술한 메모리에서 해제 예정인 객체가 제때 해제되지 않고 메모리에 남아있는 현상을 **메모리 누수**라고 한다. 이 메모리 누수가 지속되면 JVM의 힙 영역이 과포화되면서 성능이 저하되고, 심각할 경우 **OutOfMemoryError**가 발생해 앱이 종료될 수도 있다.

통상 사용이 끝난 객체가 컬렉션(집합, 리스트, 맵 등등...)이나 정적 필드에 저장된 상태로 남아있거나 동적 클래스의 과도한 로딩 등이 메모리 누수의 주요 원인이다. 이 메모리 누수가 발생하는 근본적인 이유는 **GC가 도달 가능하되 참조가 되지 않는 객체를 판별하지 못 하기 때문**이다.

### 2. 테스트 세팅

우선, 테스트 코드를 아래와 같이 짠다. 핵심은 불필요 객체를 담을 **정적 컬렉션** 변수의 도입이다.

```java
@Slf4j
@Service
public class MemoryLeakService {

    private static final List<byte[]> memoryLeakList = new ArrayList<>();

    public void generateLeak() {
        // 1MB 크기의 데이터를 리스트에 추가 (의도적 누수)
        memoryLeakList.add(new byte[1024 * 1024]); // 1MB 크기
        log.info("현재 누적 객체 수: {}", memoryLeakList.size());
    }
}
```
```java
@RestController
@RequestMapping("/leak")
@RequiredArgsConstructor
public class MemoryLeakController {

    private final MemoryLeakService memoryLeakService;

    /**
     * VisualVM Heap Dump 분석 + JMeter 호출 처리
     * -> 가상 사용자수 조건이 과하면 OutOfMemoryException 발생 가능성
     */
    @GetMapping("/test")
    public String testMemoryLeak() {
        memoryLeakService.generateLeak();
        return "메모리 누수 발생!";
    }
}
```

그런 다음, 스프링부트 앱을 실행하기 전에 인자를 제공해서 OOE가 발생할 때 힙 덤프 파일을 생성할 수 있도록 설정을 추가한다.

<img width="75%" alt="스크린샷 2025-01-03 오전 1 49 34" src="https://github.com/user-attachments/assets/16267b6b-bcb1-4136-a47e-5511bcf89023" />

트래픽 테스트 실행 환경은 아래와 같다.

테스트 환경은 아래와 같다.

>- 트래픽 발생 툴 : JMeter
>- 가상 사용자 수 : 200
>- 램프업 타임 : 30s
>- 루프 카운트 : 30

### 3. 테스트 진행 및 결과

#### (1) JMeter 테스트 스레드 설정

<img width="75%" alt="메모리누수테스트세팅" src="https://github.com/user-attachments/assets/dde832ca-930b-4f31-b3fb-c3320d18f67f" />

#### (2) 실행 결과 OutOfMemoryError 발생

정적 컬렉션에 저장되는 객체 수 카운팅 로그를 남기다가 어느 시점에서 OOE가 발생하는 것이 포착됐다. ~~그와 동시에 자바 관련 툴들 전부 먹통~~

<img width="75%" alt="OOE발생" src="https://github.com/user-attachments/assets/b6885188-a9f2-4aa0-bfa4-667ade655539" />

실행 인자에 `-XX:+HeapDumpOnOutOfMemoryError`를 부여해서 OOE가 발생하는 시점에 자동으로 힙 덤프 파일이 생성된다.

#### (3) VisualVM 기반 힙 덤프 파일 체크

앱이 부팅되고 난 바로 직후의 힙 덤프 파일을 확인해보면 아래와 같다.

<img width="75%" alt="초기힙덤프" src="https://github.com/user-attachments/assets/3c4770b2-4870-4361-81bf-827a91b54308" />

여기서 유심히 봐야할 부분이 바로 테스트 코드의 정적 컬렉션 필드 타입인 `byte[]`인데, 현 시점에서는 메모리 차지하는 크기가 약 4MB 정도밖에 안 된다. 그리고 위에서 언급한 OOE가 발생할 때 내가 직접 캐치한 힙 덤프 파일은 아래와 같다.

<img width="75%" alt="OOE발생시점힙덤프" src="https://github.com/user-attachments/assets/e93ddcc8-b516-4b94-872e-e40b372ad9f8" />

아까 정적 컬렉션의 타입인 `byte[]`의 크기가 2044MB, 약 2GB인 것을 확인할 수 있다. OOE가 발생한 시점에서 대략 500배나 그 크기가 급증한 것을 확인할 수 있다. 물론 이것은 OOE 발생 시점에 정확히 찍혔다고는 보기 어려우므로 좀 더 자세하게 확인해본다.

#### (4) OOE 힙 덤프 파일 체크

아까 앱을 실행할 때, `-XX:+HeapDumpOnOutOfMemoryError` 인자를 부여했었다. 이로 인해 자동으로 힙 덤프 파일이 생성됐다.

히카리풀에 명시됐던 경로인 `/var/folders/tz/1_xqpm3x4pd6hdvswtb_fkl40000gn/T/visualvm_kimdongjun.dat/localhost_9244/java_pid9244.hprof`를 탐색하면 인텔리제이로 힙 덤프 파일을 확인할 수 있다.

<img width="75%" alt="Retained중심OOE힙덤프파일체킹" src="https://github.com/user-attachments/assets/1316c545-f7dc-4584-850c-d9c18a8bf1e1" />

웬만한 경향은 VisualVM에서 확인한 힙 덤프와 유사하나, 직접 체크할 때 봐야 할 부분은 **Retained** 컬럼을 위주로 확인해야 한다. **Shallow** 탭은 객체 자체의 크기만을 나타내나 Retained 탭은 해당 객체가 참조하는 모든 객체가 차지하는 메모리 크기를 명시하기 때문에 Retained 컬럼을 통해 메모리 누수를 확인할 수 있다.

### 4. 테스트 분석

앞서 이론으로 봤던 `static` 변수, 컬렉션 변수에 사용이 종료된 객체를 저장하고 따로 비우는 로직이 없으면 GC는 도달 가능한 객체로만 판별하고 사망 판정을 내리지 않기 때문에 GC가 회수할 수 없게 된다.

그 이유는, 정적 변수나 컬렉션 변수는 애플리케이션의 생명주기와 똑같이 가져가기 때문이다. 정적 변수는 클래스 로더가 딱 한 번 메모리에 로드하면서 참조를 유지하기 때문에 명시적인 `null` 할당이 요구된다. 컬렉션에 저장된 객체들은 컬렉션 자체가 참조를 유지하기 때문에 자연스레 컬렉션의 생명주기를 따라가면서 살아남게 되는 것이다.

즉, 쓸데없이 메모리 차지를 하는 객체가 정리되지 못함에 따라 메모리가 쉽게 비워지지 않으면서 결국 메모리 누수가 발생하고 OutOfMemoryError가 발생하는 것이다.

요약하자면 **GC는 참조 여부만을 판단하지, 쓸데없는 참조인지는 판단할 수 없기 때문에 메모리 누수가 발생**하고, 그 대표적인 원인은 정적 변수, 컬렉션 변수가 있다. 참고로 그냥 인스턴스 필드는 객체의 생명주기와 같이하기 때문에 객체가 참조되지 않는 시점에 바로 같이 GC에 의해 정리되므로 앞서 언급한 메모리 누수의 원인에서 자유롭다.

## Issue 3. 스레드 무분별 생성 및 대기

자바는 스레드와 별개로 볼 수 없는 관계다. 애초에 자바가 다른 프로그래밍 언어와 가지는 대표적인 특징이 멀티스레딩이기도 하니까... 이 멀티스레딩을 활용해 비동기적 작업 처리, 대기 시간 감소 등의 최적화를 꾀할 수 있지만 동시성 이슈, 병목 현상 유발 등의 문제점도 같이 갖고 있다.

마찬가지로 스레드가 JVM의 메모리에 미치는 영향 역시 무시할 수 없어서 테스트 시나리오로 삼게 됐다.

### 1. 스레드 생성과 대기

자바의 스레드는 `Thread` 클래스 혹은 `Runnable` 인터페이스를 활용해 단위 생성된다. 이 생성된 스레드는 아래와 같은 상태를 가지게 된다.

>- New : 스레드 객체가 생성되었으나 아직 시작되지 않은 상태
>- Runnable : 실행 가능한 상태 (CPU에 의해 실행될 준비가 됨)
>- Blocked : 자원에 접근을 위해 대기 중인 상태
>- Waiting : 다른 조건을 기다리는 상태
>- Timed Waiting : 일정 시간 동안만 대기하는 상태
>- Terminated : 스레드 실행이 완료되어 종료된 상태

실행 가능한 상태에서 적재적소에 스레드가 실행되는 것을 관리하면 그 자체로도 최적화를 이끌어낼 수 있지만, 이유 없이 생성되거나 생성된 상태로 그저 대기만 하는 스레드는 JVM의 메모리에 악영향을 끼칠 수 있다.

### 2. 테스트 세팅

스레드를 생성하는 테스트 코드를 짜면서 중간에 대기 상태를 위한 동기화 로직을 작성한다. 이 로직을 의도적으로 반복시켜 메모리 소모량을 가시적으로 늘인다.

```java
@Slf4j
@Service
public class ThreadWaitingService {

    public void processRequest(int threadCount) {
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    log.info("스레드 시작 - {}", Thread.currentThread().getName());
                    // 스레드를 대기 상태로 두기
                    synchronized (this) {
                        wait(); // 계속 대기 상태로 두기
                    }
                    log.info("스레드 종료 - {}", Thread.currentThread().getName());
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    log.error("스레드 중단됨", e);
                }
            }).start();
        }
    }
}
```
```java
@RestController
@RequestMapping("/thread")
@RequiredArgsConstructor
public class ThreadWaitingController {

    private final ThreadWaitingService threadWaitingService;

    /**
     * VisualVM Heap Dump 체킹 + 스레드풀은 RejectedExecutionException으로 효율 예외 처리?
     * -> 직접 스레드를 생성하고 대기 상태로 오래 유지시키면 생기는 악영향?
     * -> 스레드풀 크기 확장 + HTTP 요청 타임아웃 적용
     */
    @GetMapping("/test")
    public String testBlockingThreadPool() {
        for (int i = 0; i < 100; i++) {
            threadWaitingService.processRequest(200);
        }
        return "스레드풀 블로킹 처리!";
    }
}
```

테스트 환경은 아래와 같다.

>- 트래픽 발생 툴 : JMeter
>- 가상 사용자 수 : 200
>- 램프업 타임 : 30s
>- 루프 카운트 : 30

### 3. 테스트 진행 및 결과

#### (1) JMeter 테스트 스레드 설정

<img width="75%" alt="테스트환경설정" src="https://github.com/user-attachments/assets/22fd7fd4-6e62-4d2d-bf1f-8af74ef0b2e4" />

#### (2) 실행 결과 OutOfMemoryError 발생

<img width="75%" alt="OOE로그확인" src="https://github.com/user-attachments/assets/f2d7be2a-c1ec-4ec7-8ded-f80cbf425dea" />

메모리 누수 이슈와 마찬가지로 OutOfMemoryError가 발생한다. 이 OOE가 발생한 이유는 시스템에서 할당할 수 있는 메모리가 부족하고 많은 스레드가 대기 상태에 진입함으로써 JVM의 힙 메모리가 부족해지면서 발생한다. 로그에 대해서는 아래에서 조금 더 상세히 분석해본다.


```bash
java.lang.OutOfMemoryError: unable to create native thread: possibly out of memory or process/resource limits reached] with root cause
```

네이티브 스레드, 즉 자바 스레드 모델을 동작시키기 위한 실제 커널 스레드를 생성하는 데에 실패했다는 메세지를 나타낸다.



```bash
Failed to start thread "Unknown thread" - pthread_create failed (EAGAIN) for attributes: stacksize: 2048k, guardsize: 16k, detached.
```

스택 메모리 및 보호 메모리 크기가 설정된 환경에서 스레드를 `pthread_create` 함수를 통해 새로 생성하려 했으나 실패했음을 나타낸다.

#### (3) 테스트 전후 스레드 덤프 비교

- 테스트 전

<img width="75%" alt="테스트전스레드덤프" src="https://github.com/user-attachments/assets/f38afe06-f93a-4c14-83a9-9ff77603e1ed" />


- 테스트 후

<img width="75%" alt="테스트후스레드덤프1" src="https://github.com/user-attachments/assets/eabd061d-707b-488e-86d2-5b40ab1a15f4" />
<img width="75%" alt="테스트후스레드덤프2" src="https://github.com/user-attachments/assets/7ab941a6-73ed-4886-993d-bc44a24e97b1" />

테스트 전후로 비교했을 때, 스레드 수가 굉장히 많이 늘어났으며(무분별한 생성) 생성된 대부분의 스레드가 `Object.wait()` 메소드로 인해 대기(`WAITING`) 상태에 상주하고 있다. 즉, 무분별한 대기 상태에 놓여져 있음을 알 수 있다.


#### (4) 테스트 전후 힙 덤프 비교

- 테스트 전

<img width="75%" alt="테스트전힙덤프" src="https://github.com/user-attachments/assets/d84abd44-cbb3-4bf0-bb50-c88ec8b228b6" />


- 테스트 후

<img width="75%" alt="테스트후힙덤프" src="https://github.com/user-attachments/assets/41343364-e57d-4b98-96a4-b658f00cdea7" />


직접 생성된 `Thread` 관련 객체들이 얼만큼 메모리 비중을 차지하고 있는지 확인한다. 테스트 전의 개별 `Thread`의 Retained된 값은 약 14.7KB에 불과하지만, 테스트 시행 직후에 얻은 힙 덤프에서는 Retained된 값이 약 2MB로 급증했음을 알 수 있다. 생성된 스레드의 개수를 생각하면 기가바이트 단위로 확 올랐음을 짐작할 수 있다. 즉, 스레드의 자원 관리가 효율적으로 이뤄지지 않고 있다.

### 4. 테스트 분석

사실 웬만하면 **스레드 풀**을 활용해서 스레드 생성과 대기를 효율적으로 관리하는 데에는, 큐 자료구조를 통해 작업의 순서와 대기에 있어 최적화를 이뤄낼 수 있기 때문이다. 단위 스레드를 생성하는 것 또한 방법 중 하나지만 경쟁 조건에 취약하다보니 동기화가 필수적이고, 이는 성능 저하로 이어질 수 있다.

JDK 21에서는 가상 스레드를 활용해서 조금 더 최적화된 스레드 풀을 활용할 수 있으니 이를 참고해서 스레드 생성 작업에 투입하는 것이 메모리 관리 측면에서도 옳은 방향일 것이다. 참고로 스레드 풀에서 수많은 스레드 생성으로 스레드 풀과 작업 큐의 용량을 초과하면 `RejectedExecutionException`을 발생시키며 예외로 처리한다.

## Issue 4. GC 옵션에 따른 동작 분석

JVM의 GC는 다양한 종류가 있다. 현재 JDK 21의 디폴트 GC는 **G1 GC**(Garbage First GC)이며, 동일한 GC여도 다양한 실행 인자를 부여하여 애플리케이션에 최적화된 GC 옵션을 제공할 수 있다. 즉, 메모리 관리를 효율적으로 수행하려면 GC의 옵션 고려 역시 중요 사항에 속한다.

이번 테스트는 GC의 선택을 다르게 해서 VisualVM의 Visual GC 플러그인을 통해 객체 정리 그래프가 어떻게 출력되는지 확인해본다.

### 1. G1 vs Parallel

둘의 개념을 정리하는 건 생략한다. G1은 저지연 및 예측 가능한 GC 시간을 목표로 설계됐고, Parllel, 즉 병렬 GC는 높은 처리량을 목적으로 설계됐다. 시기상으로는 병렬 GC가 앞서기 때문에 조금 더 구식인 느낌이 있지만 실제로는 전술한 애플리케이션의 설계 방향에 따라 오히려 병렬 GC가 고효율의 성능을 나타낼 수 있다.

그렇기 때문에 사실 다양한 테스트 코드를 작성하고 비교 분석하는 것이 조금 더 정확한 테스트가 되겠으나 현재는 스터디의 목적에 충실하게 일단 그래프 분석을 우선으로 삼아 테스트를 진행해볼 예정이다.

### 2. 테스트 세팅

테스트 코드에는 간단히 객체를 생성하고 연산하면서 정리하는 비즈니스 로직과 컨트롤러 호출을 세팅한다.

```java
@Slf4j
@Service
public class GcService {

    public void performGcIntensiveTask() {
        for (int i = 0; i < 100; i++) {
            // 리스트 세팅
            List<Integer> numbers = new ArrayList<>();

            for (int j = 0; j < 10_000; j++) {
                numbers.add(j);
            }

            // 간단한 연산
            int sum = numbers.stream().mapToInt(Integer::intValue).sum();
            log.info("현재 작업 연산값: {}", sum);

            // 메모리 제거
            numbers.clear();
        }
    }
}
```
```java
@RestController
@RequestMapping("/gc")
@RequiredArgsConstructor
public class GcController {

    private final GcService gcService;

    /**
     * Visual GC 플러그인 활용 -> GC 옵션 바꿀 수 있으면 바꿔보기
     */
    @GetMapping("/test")
    public String testGcIssue() {
        gcService.performGcIntensiveTask();
        return "GC 작동";
    }
}
```

이번 테스트는 동일한 앱 내에서 GC의 동작 및 결과 차이를 확인하기 위해서므로, 실행 옵션에 GC와 관련된 파라미터를 부여한다. 인텔리제이 IDE는 VM Option을 설정에서 별도로 제공하기 때문에 쉽게 파라미터 부여가 가능하다.

<img width="75%" alt="VM옵션파라미터부여" src="https://github.com/user-attachments/assets/f8de4b8d-65c3-4458-9545-66e85ff739c2" />

트래픽 테스트 실행 환경은 아래와 같다.

>- 트래픽 발생 툴 : JMeter
>- 가상 사용자 수 : 200
>- 램프업 타임 : 30s
>- 루프 카운트 : 30

### 3. 테스트 진행 및 결과

#### (1) JMeter 테스트 스레드 설정

<img width="75%" alt="테스트설정" src="https://github.com/user-attachments/assets/c88cac58-8904-49d2-a6d8-336ebcc672d7" />

#### (2) G1 GC 테스트

##### 실행 옵션

```bash
-XX:+UseG1GC -verbose:gc -Xlog:gc*:file=gc.log:time,uptime,level,tags
```

##### 로그 확인

<img width="75%" alt="G1GC로그출력" src="https://github.com/user-attachments/assets/4055868a-8df0-48f2-810c-5a327cb06f0c" />

##### Visual GC 그래프

<img width="75%" alt="G1GC영역분석" src="https://github.com/user-attachments/assets/b2dbb1a9-22f7-416e-8ed5-3d5c42850717" />

#### (3) Parallel GC 테스트

##### 실행 옵션

```bash
-XX:+UseParallelGC -XX:ParallelGCThreads=8 -XX:MaxGCPauseMillis=200 -verbose:gc -Xlog:gc*:file=gc.log:time,uptime,level,tags
```

##### 로그 확인

<img width="75%" alt="병렬GC로그출력" src="https://github.com/user-attachments/assets/b9cbb31a-baad-4125-b770-11a302172124" />

##### Visual GC 그래프

<img width="75%" alt="병렬GC영역분석" src="https://github.com/user-attachments/assets/4e9b31a4-ccda-4abd-81c9-94a8e9ab0264" />


### 4. 테스트 분석

#### (1) GC 수행시간

> **G1 GC**\
>GC Time: 195 collections, 4.178s Last Cause: G1 Evacuation Pause
> 
> **Parallel GC**\
>GC Time: 162 collections, 1.964s Last Cause: Allocation Failure

병렬 GC가 더 적은 수의 GC를 수행하고 더 짧은 시간 동안 완료됐다. 병렬 GC가 여러 스레드를 사용해 GC 작업을 병렬로 처리하여 성능 향상을 확인할 수 있다. 반면, G1 GC는 더 많은 GC를 수행했으며, GC 시간이 길어졌는데 이는 G1이 더 세밀하게 메모리 영역을 관리하려는 특성 때문일 수 있다.

G1 GC에서는 Evacuation Pause가 원인이 되어 GC 시간이 길어졌고, Young에서 Old 영역으로의 객체 이동 과정에서 발생한 멈춤으로 볼 수 있다. 병렬 GC에서는 Allocation Failure가 발생하여, 힙 공간 부족으로 인해 GC가 실행되었는데, Young 영역의 공간 부족으로 인해 GC가 수행된 것이며 이를 해결하기 위해 메모리 공간을 정리하는 작업이 이뤄졌다.

#### (2) Eden 영역

> **G1 GC**\
> Eden Space (4.000G, 1.576G): 948.000M, 195 collections, 4.178s
>
> **Parallel GC**\
> Eden Space (1.332G, 1.274G): 332.012M, 160 collections, 1.890s

둘 다 모두 Eden 영역에서 많은 메모리 할당을 다뤘지만, 병렬 GC는 빠르게 처리된 반면 G1 GC는 여러 차례의 세밀한 GC를 수행한 것을 확인할 수 있다. 병렬 GC는 메모리를 한 번에 많이 처리할 수 있지만, G1 GC는 조금 더 세밀한 관리를 수행하는 것이 주요 원인으로 생각된다.

#### (3) Survivor 영역

> **G1 GC**\
> Survivor 0 (0, 0): 0\
> Survivor 1 (4.000G, 6.000M): 4.584M
>
> **Parallel GC**\
> Survivor 0 (455.000M, 29.500M): 957.156K\
> Survivor 1 (455.000M, 30.000M): 0

가장 두드러지는 특징이 Survivor 영역에서 나타났다. G1은 **Survivor 영역을 세밀히 관리하여 Eden에서 Old로 직접 이동시키는 것을 최대한 지연**하려고 하는 반면, 병렬 GC는 **객체를 빠르게 Old 영역으로 옮겨 Survivor1을 비움으로써 빠른 GC를 유도하려 하기 때문**이다.

Survivor 영역에서 G1 GC는 효율적인 관리를 통한 메모리 분배 경향을, 병렬 GC는 빠른 GC 성능을 우선시하면서 메모리 압박 우선 해결 경향을 보이는 것을 확인할 수 있다.

#### (4) Old 영역

> **G1 GC**\
> Old Gen (4.000G, 952.000M): 35.766M, 0 collections, 0s
>
> **Parallel GC**\
> Old Gen (2.667G, 171.000M): 64.503M, 2 collections, 73.634ms

결과적으로 G1은 Old 영역의 메모리 활용을 최대한 덜하며 GC 수행 시간이 적었던 반면, 병렬 GC는 Old 영역을 빠르게 소진시키면서 GC 수행 횟수가 증가하고 해당 영역의 사용량도 증가한 것을 볼 수 있다.

#### (5) 결론

테스트 코드의 트래픽 테스트에서는 병렬 GC가 더 적합할 것이다. Young 영역의 빠른 GC 회수 덕분에 성능이 개선될 수 있기 떄문이다.

다만 GC가 너무 자주 발생하면 G1이 더 안정적인 성능을 제공할 수 있으므로, 메모리의 크기나 사용 패턴에 따라 적합한 GC를 선택하는 것이 중요할 것이고 이 과정은 테스트를 통해 근거를 확보하는 것이 옳을 것이다.
