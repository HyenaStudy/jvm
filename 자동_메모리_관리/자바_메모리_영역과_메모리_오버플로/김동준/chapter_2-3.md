# 2.3

## JVM에서의 객체 생명주기

### 1. 객체 생성

- 클래스 로딩
- 메모리 할당
- 객체 헤더 초기화
- 생성자 호출

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
