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
