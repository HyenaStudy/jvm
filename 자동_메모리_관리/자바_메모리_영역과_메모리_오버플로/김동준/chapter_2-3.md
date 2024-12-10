# 2.3

## JVM에서의 객체 생명주기

### 1. 객체 생성

다음 코드를 기반으로 JVM에서의 객체 생성 과정을 확인한다. 현재 `psvm` 실행 전이기 때문에 런타임 데이터 영역에 아무 것도 없는 상태다.

```java
public class Main {
    public static void main(String[] args) {
        Customer customer;
        customer = new Customer();

        customer.name = "KIM";
        customer.age = 10;
        customer.address = "Seoul";
    }
}

class Customer {
    String name;
    int age;
    String address;

    public void sayHi() {
    }
}
```

#### (1) Main 클래스, main 정적 메소드

```java
public class Main {
    public static void main( // ...
```

자바는 동적 로딩을 하는 언어이기 때문에 런타임 시점에서 클래스를 로드한다. 따라서 컴파일 단계에서 자바 바이트 코드를 생성한 뒤에, JVM에서 인터프리터를 통해 줄 단위로 실행하게 된다. 실행 시점에서 `main` 메소드를 찾아서 실행해야 하므로 이를 갖는 클래스인 `Main` 클래스를 로드하면서 시작된다. 각각 클래스 정보와 정적 메소드이기 때문에 JVM의 메타스페이스에 저장된다.

<img src="https://github.com/user-attachments/assets/3a36837b-6e35-444f-8a30-b5a922454ebc" width="50%" />

#### (2) args[] 메소드 파라미터

```java
    public static void main(String[] args) { // ...
```

`main` 정적 메소드는 `String[]` 타입의 파라미터가 존재한다. 즉 메소드의 파라미터 정보가 있기 때문에 이 시점에서 `main` 메소드의 스택 프레임(정확히는 지역변수 영역)이 할당된다. 스택 프레임에 `main` 메소드의 파라미터 정보가 저장된다. 여기까지의 JVM 런타임 데이터 영역을 살펴보면 아래와 같다.

<img src="https://github.com/user-attachments/assets/14bfdbb6-55e5-4d00-bb34-1a92aa6d0fc3" width="50%" />

#### (3) Human 변수 및 인스턴스

```java
        Customer customer;
        customer = new Customer();
        // ...
```

`customer` 변수는 메소드의 지역 변수이기 때문에 스택의 지역변수 영역에 저장된다. 즉, 아까 파라미터 정보가 저장된 스택 프레임에 `customer` 변수도 같이 저장된다. 이제 `customer` 변수에 `Customer` 클래스의 생성자 메소드가 호출된 것이기 때문에, 스택에 또 다른 스택 프레임이 할당된다. 즉, `new` 키워드는 새로운 메소드를 생성하는 것과 똑같다. 또한 파라미터가 없기 때문에 별도의 지역변수 영역으로 할당되지는 않는다.

`new` 키워드와 동시에 `Customer` 클래스가 생성되어야 하므로, `Customer` 클래스 정보가 역시 메타스페이스에 저장된다. 여기까지의 JVM 런타임 데이터 영역을 들여보면 아래와 같을 것이다.

<img src="https://github.com/user-attachments/assets/032317c2-5ae4-4376-bb59-e37aef0b0792" width="50%" />


`Customer`라는 클래스 데이터가 메타스페이스에서 동적으로 로딩될 때, `Customer` 클래스 내의 정보(필드, 메소드)들 역시 메타데이터로써 포함되고, 여기서 필드의 실제 값은 힙에 할당되게 된다. 그리고 코드 상에서는 초기화 값을 별도로 지정하지 않았기 때문에 힙에서는 해당 변수들이 `null`로 초기 할당이 이뤄지게 된다. 단, `age`는 원시 타입이기 때문에 0으로 초기화가 이뤄지면서 생성된 해당 인스턴스의 참조 주소값이 `new` 스택 프레임의 `this`에 저장된다.

<img src="https://github.com/user-attachments/assets/b2d9d88e-0eef-489a-8397-88f84bd7da08" width="50%" />

#### (4) 인스턴스 필드 변경

```java
        customer.name = "KIM";
        customer.age = 10;
        customer.address = "Seoul";
```

이제 인스턴스가 생성됐기 때문에 해당 참조 주소값을 `main` 스택 프레임의 지역변수인 `consumer`가 바라볼 수 있도록 할당 후, `new`는 생성자로써의 역할이 끝난(즉, 호출이 종료된) 상태이므로 현재 스택에서 사라지게 된다. 그리고 기존의 0으로 초기화된 원시 타입의 변수(`age`)는 새로운 값이 들어오면 해당 값으로 대체되고, `null`이 할당됐던 래퍼 타입들의 변수가 바라보는 힙의 공간에 해당 타입의 새로운 값으로 대체된다.

<img src="https://github.com/user-attachments/assets/f0da84c9-31d4-4901-b27b-36ca023ba97a" width="50%" />

### 2. 객체 메모리 레이아웃

#### (1) 구조

- 헤더(Header)
- 인스턴스 데이터(Instance Data)
- 패딩(Padding)

#### (2) 관련 자료구조

*https://xephysis.tistory.com/6*

### 3. 객체 접근방식

- 참조 vs offset
- `Unsafe` 클래스의 역할
