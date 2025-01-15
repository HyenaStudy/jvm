## 10.2 javac 컴파일러
### 10.2.1 javac 소스 코드와 디버깅
**컴파일 단계**<br>
0. 플러그인 애너테이션 처리기들 초기화<br>
1. 구문 분석 및 심벌 테이블 채우기<br>
1-1. 어휘 및 구문 분석 : 소스 코드를 토큰화하여 추상 구문 트리 구성<br>
1-2. 심벌 테이블 채우기 : 심벌 주소와 심벌 정보 생성<br>
2. 플러그인 애너테이션 처리기들로 애너테이션 처리<br>
3. 의미 분석 및 바이트코드 생성<br>
3-1. 특성 검사 : 문법의 정적 정보 확인<br>
3-2. 데이터 흐름 및 제어 흐름 분석 : 프로그램의 동적 실행 과정 확인<br>
3-3. 편의 문법 제거 : 코드를 단순화하는 편의 문법을 원래 형식으로 복원<br>
3-4. 바이트코드 생성 : 지금까지 생성된 정보를 바이트코드로 변환<br>

- javac 컴파일 담당 : com.sun.tools.javac.main.JavaCompiler 클래스
<br><br>

### 10.2.2 구문 분석과 심벌 테이블 채우기
> 1. 어휘 및 구문 분석

**어휘 분석**
- 소스 코드의 문자 스트림을 토큰 집합으로 변환하는 일<br>
- 컴파일에서는 키워드, 변수명,  리터럴, 연산자 같은 토큰이 가장 작은 단위
- 담당 : com.sun.tools.javac.parser.Scanner 클래스

**구문 분석**
- 일련의 토큰에서 추상 구문 트리를 구성하는 과정
- 생성된 추상 구문 트리는 com.sun.tools.javac.tree.JCTree 클래스로 표현
- 담당 : com.sun.tools.javac.parser.Parser 클래스

**추상 구문 트리**
- 코드의 문법 구조를 트리 형태로 기술하는 기법
- 노드는 코드의 구문 구조를 나타냄
- 코드를 잘 구조화해 표현하지만 의미 체계가 논리적인지까지는 보장 X
<br>

> 2. 심벌 테이블 채우기

**심벌 테이블**
- 심벌 주소와 심벌 정보의 집합으로 구성된 데이터 구조
- 추상 구문 트리의 최상위 노드와 package-info.java의 최상위 노드 목록 생성
- 담당 : com.sun.tools.javac.comp.Enter 클래스
<br><br>

### 10.2.3 애너테이션 처리
- 애너테이션은 코드에 추가 정보를 제공하며 컴파일러가 이를 처리
- 플러그인 애너테이션 처리기는 컴파일 과정에서 추상 구문 트리의 요소를 읽고 수정하고 추가할 수 있는 컴파일러용 플러그인
- 라운드 : 애너테이션 처리 중 구문 트리를 수정하면 컴파일러가 1단계로 돌아가는 한 번의 반복<br>
-> 처리기가 구문 트리를 수정하지 않을 때까지 반복
- 롬복 : 코딩 효율 개선 도구. 게터/세터 생성, null 체크, equals()와 hashCode() 자동 생성 등
<br><br>

### 10.2.4 의미 분석과 바이트코드 생성
**의미 분석**
- 구조적으로 올바른 소스가 맥락상으로 올바른지 확인하는 것.
- ex) 타입 검사, 제어 흐름 검사, 데이터 흐름 검사 등
- IDE에서 빨간 밑줄로 오류 표시해주는 게 여기서 발견
<br>

> 1. 특성 검사
- 변수를 사용하기 전 선언이 되어있는지, 변수와 할당된 데이터 타입이 일치하는지 등 확인
- 상수 접기 최적화 수행

  **상수 접기**
  - 컴파일러가 상수 값을 미리 계산해 실행 성능을 향상시키는 것
  - 상수끼리의 연산이어야 하고 런타임에 변경되지 않는 불변 값이어야 함
<br>

> 2. 데이터 흐름 분석과 제어 흐름 분석
- 프로그램이 맥락상 논리적으로 올바른지 확인하는 추가 검사
- ex) 지역변수가 사용되기 전 값이 할당되었는지, 메서드의 모든 실행 경로에서 값을 반환하는지, 검사 예외는 올바르게 처리되는지 등
<br>

> 3. 편의 문법 제거
- 개발자가 언어를 쉽게 사용할 수 있도록 프로그래밍 언어에 추가된 구문을 원래의 기본 구문 구조로 복원하는 것
- ex) 제네릭, 가변 길이 매개변수, 오토박싱/언박싱 등
<br>

> 4. 바이트코드 생성
- javac 컴파일 과정의 마지막 단계
- 이전 단계에서 생성한 정보를 바이트코드 명령어로 변환해 저장소에 기록
<br><br>

## 10.3 자바 편의 문법의 재미난 점
### 10.3.1 제네릭
- 특수한 매개 변수를 사용해 작업 대상의 데이터 타입을 지정할 수 있게 하는 것
- 데이터 타입에 구애받지 않는 알고리즘을 작성할 수 있어 타입 시스템과 추상화 능력 크게 향상
<br>

> 자바와 C#의 제네릭

**자바**
- 타입 소거 제네릭. 타입 정보 사라짐
- 제네릭이 필요한 모든 기존 타입을 제네릭 버전으로 변경

**C#**
- 구체화된 제네릭. 구체적 정보 남아있음
- 제네릭이 필요한 타입 중 일부는 그대로 두고 제네릭 버전을 따로 추가
- 자바보다 성능이 뛰어남
<br><br>

### 10.3.2 오토박싱, 오토언박싱, 개선된 for문
**컴파일로 편의 문법 제거**
- 제네릭 -> 원시 타입
- 오토박싱, 언박싱 -> 래퍼 타입과 복원 메서드
- 개선된 for문 -> Iterator 사용 코드
- 가변 길이 매개 변수 -> 배열 형태
<br>

**편의문법 제거 전**
```
Integer a = 1;
Integer b = 2;
Integer c = 3;
Integer d = 3;
Integer e = 321;
Integer f = 321;
Long g = 3L;

System.out.println(c == d);		// true
System.out.println(e == f);		// false
System.out.println(c == (a + b));	// true
System.out.println(c.equals(a + b));	// true
System.out.println(g == (a + b));	// true
System.out.println(g.equals(a + b));	// false
```

**편의문법 제거 후(println 결과 동일)**
```
Integer a = Integer.valueOf(1);
Integer b = Integer.valueOf(2);
Integer c = Integer.valueOf(3);
Integer d = Integer.valueOf(3);
Integer e = Integer.valueOf(321);
Integer f = Integer.valueOf(321);
Long g = Long.valueOf(3L);

System.out.println(c == d);  
System.out.println(e == f);  
System.out.println(c == (a.intValue() + b.intValue()));  
System.out.println(c.equals(Integer.valueOf(a.intValue() + b.intValue())));  
System.out.println(g == (a.intValue() + b.intValue()));  
System.out.println(g.equals(Long.valueOf(a.intValue() + b.intValue())));
```
<br>

### 10.3.3 조건부 컴파일
- 자바 컴파일러는 컴파일 단위 전체를 포괄하는 구문 트리를 만들어 최상위 노드부터 하나씩 컴파일
- if문의 조건에 상수를 넣어 조건부 컴파일 가능