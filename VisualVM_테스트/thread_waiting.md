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
