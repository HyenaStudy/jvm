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
