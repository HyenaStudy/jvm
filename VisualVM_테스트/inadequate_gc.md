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
