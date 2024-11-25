# 12.4
## 스레드 개념

### 1. 운영체제의 멀티 스레딩 모델

![image](https://github.com/user-attachments/assets/2c23e71f-5cf2-4548-9e70-45de0e59696c)

#### (1) 커널 스레드(Kernel Thread)
   - 운영체제의 커널(핵심 프로그램)이 직접 관리하고 스케줄링하는 스레드.
   - 장점: 커널에서 직접 스케줄링하므로 멀티프로세서 환경에서 성능이 뛰어나고, 각 스레드가 독립적으로 자원을 관리할 수 있다.
   - 단점: 커널의 복잡성을 증가시키고, 시스템 자원 소모가 많아질 수 있다.

#### (2) 사용자 스레드(User Thread)
   - 애플리케이션에서 관리되는 스레드로, 커널은 스레드를 구분하지 않고 단일 프로세스로 처리한다.
   - 장점: 스레드 생성과 관리가 빠르고, 애플리케이션에서 세밀하게 제어할 수 있다.
   - 단점: 하나의 사용자 스레드가 블록되면 전체 프로세스가 블록될 수 있어 성능에 제한이 있을 수 있다.

#### (3) 하이브리드 스레드(Hybrid Thread)
   - 커널 스레드와 사용자 스레드를 결합하여, 사용자 스레드들이 커널 스레드로 매핑되는 방식.
   - 장점: 커널 스레드와 사용자 스레드의 장점을 결합하여, 자원을 효율적으로 관리하고 성능을 최적화할 수 있다.
   - 단점: 구현이 복잡하고, 성능 향상이 어려울 수 있다.

요약하자면, 사용자가 관리하면 사용자 스레드, 커널(OS)이 관리하면 커널 스레드, 둘 다 합쳐진 케이스면 하이브리드 스레드.
기존의 자바 스레드인 **플랫폼 스레드**는 커널 스레드를 사용하고, JDK 21에 도입된 **가상 스레드**는 사용자 스레드 기반으로 동작한다.

## JNI

일단 스레드를 공부하기 전에 **JNI**(Java Native Interface)를 공부해야 소스들이 이해될 것 같아서 얘부터 공부하자.<br />
JNI는 **자바와 네이티브 코드(C, C#) 간의 상호작용**을 가능하게 하는 인터페이스다.<br />
JVM 공부하는 데에 왜 뜬금없이 C가 나오나 싶지만, JVM과의 관계를 먼저 파악해야 한다.

### 1. JNI와 JVM 간의 관계

![image](https://github.com/user-attachments/assets/fc8433a9-5393-4aff-a1e0-9df7fb4ce347)

앞서 말했듯, JNI는 자바와 네이티브 코드 간의 상호작용을 조율한다.<br />
그리고 애시당초 JVM은 자바의 특징 중 하나인 **플랫폼 독립성**을 위해 설계됐다.<br />
이 두 내용을 바탕으로 조금 더 세부적으로 파악해보자.

>1. **JVM** 자체가 하나의 독립적인 플랫폼으로써 존재 의의가 있다.
>2. 이를 통해, 자바 프로그램이 모든 플랫폼에서 동작할 수 있는 것이다.
>3. JVM은 현재 동작하는 로컬의 운영체제(OS) 및 하드웨어 세부 사항을 추상화하여 플랫폼을 구축한다.
>4. 그러나 실제 OS 내에서 조율되는 저수준 기능 및 고성능 처리는 자바로 조율하는 것이 불가능하다.
>5. 대신 **C**, **C++** 등은 가능하다. 그렇기 때문에 자바가 불가능한 부분을 C, 즉 네이티브 코드가 맡는다.
>6. 이로써 네이티브 코드가 맡는 기능과 자바 프로그램 간의 상호작용을 위해 **JNI**가 필요한 것이다.

### 2. JNI 실습

자바 코드의 키워드 중, `native`라는 키워드가 있다.<br />
얘는 메소드에서 선언하며 해당 메소드가 자바가 아닌 다른 코드(C, C++)로 구현됐음을 명시한다.<br />
통상 시스템 자원에 직접 접근하는 저수준 기능, 고성능 처리를 위할 때나 외부 라이브러리 사용 목적으로 쓴다.<br />

이제 실습해보자.

#### (1) 네이티브 메소드 선언

```java
package com.example.jni;

public class NativeTest {
    public native void nativeMethod();  // 네이티브 코드에서 구현할 시그니처 제공

    static {
        System.loadLibrary("native-lib");  // "native-lib" 네이티브 라이브러리 로드
    }

    public static void main(String[] args) {
        new NativeTest().nativeMethod();  // 네이티브 라이브러리에서 구현된 코드를 실행하도록 트리깅
    }
}
```

#### (2) 헤더 파일 생성

`NativeTest` 자바 파일의 경로에서 헤더 파일을 생성한다.

```bash
# javac -h {헤더 파일 생성 경로} {자바 파일 경로} 
javac -h . NativeTest.java
```

<img width="1088" alt="헤더파일및디컴파일된파일생성" src="https://github.com/user-attachments/assets/8fa019a5-b88f-4191-9457-6d65a8d13a19">

헤더 파일과 디컴파일된 파일이 생성된다.

#### (3) C 기반 네이티브 메소드 작성

C로 `com_example_jni_NativeTest.c` 파일을 생성하고 네이티브 메소드를 구현한다.<br />
헤더 파일이 위치한 경로에서 생성한다.

```c
#include <jni.h>  // JNI 필수 헤더(Java <-> 네이티브 코드(C) 간의 인터페이스 정의 역할)
#include "com_example_jni_NativeTest.h"  // 생성된 헤더 파일
#include <stdio.h>  // 표준 입출력 사용을 위해 필요

// 네이티브 메소드 구현
JNIEXPORT void JNICALL Java_com_example_jni_NativeTest_nativeMethod
  (JNIEnv *env, jobject obj) {
    // 출력 메시지
    printf("네이티브 메소드 구현\n");
}
```

#### (4) 네이티브 라이브러리 컴파일

C 소스 파일을 네이티브 라이브러리로 컴파일한다.

```bash
# gcc -I$JAVA_HOME/include -I$JAVA_HOME/include/darwin -shared -m64 -o {생성할 라이브러리 이름} {C 소스 파일 경로}
gcc -I$JAVA_HOME/include -I$JAVA_HOME/include/darwin -shared -m64 -o libnative-lib.dylib com_example_jni_NativeTest.c
```

<img width="1094" alt="c컴파일" src="https://github.com/user-attachments/assets/0ed8368a-af1d-4eac-9a44-23baa6373e18">

`.dylib` 확장자는 macos에서 사용하는 동적 라이브러리 확장자명이다.

#### (5) 자바 프로그램 실행

자바 프로그램을 실행하면서, 네이티브 라이브러리를 로드하고 클래스 경로를 설정한다.

```bash
# java -Djava.library.path={생성한 라이브러리 위치} -classpath {컴파일된 클래스 파일 위치} {실행할 클래스명}
java -Djava.library.path=/Users/kimdongjun/Desktop/BackEnd/jvm-study/src/main/java/com/example/jni \
     -classpath /Users/kimdongjun/Desktop/BackEnd/jvm-study/src/main/java \
     com.example.jni.NativeTest
```

<img width="1098" alt="실행" src="https://github.com/user-attachments/assets/ba98a4eb-cb33-4af2-a75c-1cec8b811c86">

`nativeMethod()`가 호출되면서, C로 작성된 네이티브 코드가 실행된다.


## 자바 스레드

아까 위에서 JVM 및 JNI의 역할과 존재 의의에 대해 알아봤다.<br />
근데 이것이 스레드와 무슨 관계가 있냐면, 자바 코드의 스레드를 넘어 실제 운영체제의 스레드까지 알아야 한다.

### 1. 자바 스레드의 동작

자바의 코드 레벨에서 작성되는 스레드(`Thread` 클래스, `Runnable` 인터페이스)는 추상적인 모델이다.<br />
그리고 이 스레드 모델은 실제 운영체제에서 실제로 동작하는 커널 스레드와 1대 1로 매핑된다.<br />
이 과정에서 실제 커널 스레드가 생성되는 것에서 JNI가 관여한다. `Thread` 클래스를 탐색해 보면...

```java
// Thread class

    public void start() {
        synchronized (this) {
            // zero status corresponds to state "NEW".
            if (holder.threadStatus != 0)
                throw new IllegalThreadStateException();
            start0();
        }
    }

    // ...

    private native void start0();
```

네이티브 메소드인 `start0()`가 선언되어 있다. 즉, **매핑돼는 커널 스레드의 생성에서 JNI가 관여(중재)**한다.<br />
자바 스레드의 스케줄링 및 상태 관리 등은 JVM이 맡고 네이티브 코드에서의 스레드 제어를 JNI가 맡게 된다.<br />

### 2. 기존 스레드의 문제점

![image](https://github.com/user-attachments/assets/53fbca7b-8a23-461d-8722-295660e0ee53)

JVM 및 JNI의 관여 과정을 통해 자바 스레드가 어떻게 생성되고 관리되는 지를 확인했다.<br />
그러나 그 과정에서 기존 자바 스레드의 문제점이 대두된다. 기존 스레드의 문제점들을 간략히 정리해보자

#### (1) 커널 스레드 생성

자바의 스레드 모델에 매핑되는 커널 스레드 생성에도 **메모리 할당**, **운영체제 API** 호출 등의 비용이 발생한다.<br />
보통 자바 스레드의 스택은 1 MB 정도로 설정돼서, 10000개의 스레드를 생성하는 데에 10 GB의 메모리가 필요하다.

#### (2) 컨텍스트 스위치

**스레드 스케줄링**은 운영체제가 CPU를 여러 스레드 사이에서 분배하는 과정을 말한다.<br />
이 OS 스케줄러가 커널 스레드 간의 전환, 즉 **컨텍스트 스위치**를 처리하는 데 높은 비용이 든다. <br />
당연히 스레드 수가 많아질수록 전환 횟수와 비용이 증가해 성능 저하를 초래한다.

#### (3) 동시성 처리

동시성 문제의 해결은 스레드 수가 많을 때 의미가 뚜렷해진다.<br />
그런데 스레드를 생성할 수록 메모리 소모 및 스케줄링 부담이 커지게 된다.<br />
비동기 프로그래밍 등으로 극복할 수는 있지만, 이는 유지보수에 어려움을 유발한다.

위의 문제들을 해결하기 위해 JDK 21에서 **가상 스레드**가 도입된다.

---

*출처*<br />
*https://www.geeksforgeeks.org/difference-between-user-level-thread-and-kernel-level-thread/*<br />
*https://jofestudio.tistory.com/138*<br />
*https://letsmakemyselfprogrammer.tistory.com/98*<br />
*https://d2.naver.com/helloworld/1203723*<br />
*https://velog.io/@on5949/%EC%8A%A4%ED%84%B0%EB%94%94%EC%84%B8%EB%AF%B8%EB%82%98-Java%EC%9D%98-%EB%AF%B8%EB%9E%98-Virtual-Thread*
