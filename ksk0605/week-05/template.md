# 템플릿 콜백 패턴

전략 패턴의 특별한 형태. **전략을 파라미터로 전달**.

* 일반 전략 패턴: Context가 Strategy를 **필드(멤버 변수)**로 가지고 있는 경우가 많음. (생성 시점에 조립)
* 템플릿 콜백 패턴: Template이 Callback을 메서드 파라미터로 받음. (실행 시점에 주입)
* 변하는 부분만 개발자의 원하는 메서드를 주입하는 것이 가능

```java
public class FileReadTemplate {
    public <T> T read(String filepath, BufferedReaderCallback<T> callback) {
        // [변하지 않는 부분] 1. 리소스(파일) 열기 
        try (BufferedReader br = new BufferedReader(new FileReader(filepath))) {

            // [변하는 부분] 2. 핵심 로직 실행 (콜백 호출)
            return callback.doWithReader(br);

        } catch (IOException e) {
            // [변하지 않는 부분] 3. 예외 처리
            throw new RuntimeException("파일을 읽는 중 에러가 발생했습니다.", e);
        }
        // [변하지 않는 부분] 4. 리소스 종료 (try-with-resources가 자동으로 close 처리)
    }
}

package tobyspring.fileread;

import java.io.BufferedReader;
import java.io.IOException;

@FunctionalInterface
public interface BufferedReaderCallback<T> {
    T doWithReader(BufferedReader reader) throws IOException;
}


public class Client {
    public static void main(String[] args) {
        FileReadTemplate template = new FileReadTemplate();
        String path = "numbers.txt";

        // 사용 예시 1: 첫 번째 줄만 읽기
        String firstLine = template.read(path, (br) -> {
            return br.readLine(); // 핵심 로직
        });
        System.out.println("첫 줄: " + firstLine);


        // 사용 예시 2: 파일 내의 모든 숫자를 더하기
        Integer sum = template.read(path, (br) -> {
            int total = 0;
            String line;
            while((line = br.readLine()) != null) {
                total += Integer.parseInt(line); // 핵심 로직
            }
            return total;
        });
        System.out.println("총 합: " + sum);
    }
}
```

# 자바 함수형 인터페이스의 활용

스프링 5 이후의 템플릿들은 자바 8의 **함수형 인터페이스(Functional Interface)**와 **람다(Lambda)**를 적극적으로 수용

### @FunctionalInterface
인터페이스인데 추상 메서드가 오직 하나만 있는 인터페이스.

구현해야 할 기능이 딱 하나뿐이니, 굳이 클래스를 만들거나 메서드 이름을 명시할 필요 없이 로직(코드 블록)만 넘기는 것이 가능(**람다식(() -> {})**이 가능한 이유).

[자바 8 이전: 익명 내부 클래스 (Anonymous Inner Class)] 

```Java

// 변하는 로직을 전달하기 위해 불필요한 코드가 많음
jdbcTemplate.query("SELECT * FROM USERS", new RowMapper<User>() {
    @Override
    public User mapRow(ResultSet rs, int rowNum) throws SQLException {
        User user = new User();
        user.setName(rs.getString("name"));
        return user;
    }
});
```
[람다 (Lambda)] RowMapper가 함수형 인터페이스(mapRow 하나만 있음)이므로 람다로 대체 가능.

```Java
jdbcTemplate.query("SELECT * FROM USERS", (rs, rowNum) -> {
    User user = new User();
    user.setName(rs.getString("name"));
    return user;
});
```

# 람다

자바 람다의 핵심은 **컴파일 타임(Compile-time)**이 아니라 **런타임(Runtime)**과 **JVM의 메모리 관리** 방식. 람다가 내부적으로 어떻게 동작하는지, 익명 클래스와 결정적으로 무엇이 다른지 **바이트코드 레벨**에서 이해해야 함.

---

### 1. 익명 클래스 vs 람다: 바이트코드의 차이

#### (1) 익명 내부 클래스 (Anonymous Inner Class)

```java
// 코드
Runnable r = new Runnable() {
    @Override
    public void run() { System.out.println("Hello"); }
};

```

* **컴파일 결과:** `Main$1.class`라는 별도의 클래스 파일이 디스크에 생성.
* **동작:** 런타임에 이 클래스를 로딩하고, 인스턴스를 생성하여 힙 메모리에 올립니다. 클래스 파일이 많아지면 메타스페이스(Metaspace) 메모리를 많이 차지.

#### (2) 람다 (Lambda)

```java
// 코드
Runnable r = () -> System.out.println("Hello");

```

* **컴파일 결과:** **별도의 클래스 파일이 생성되지 않습니다.** (`Main$1.class` 없음)
* **동작:** 대신 해당 위치에 `invokedynamic`이라는 특별한 바이트코드 명령어가 삽입.

---

### 2. 핵심 기술: `invokedynamic`, LambdaMetafactory

1. **레시피만 적어둠:** 컴파일러는 람다 코드를 만나면 "여기서 람다 객체가 필요해"라는 표시(`invokedynamic`)와 "어떤 모양의 람다를 만들어야 하는지(Recipe)"만 바이트코드에 적어둠.
2. **런타임 생성 (Lazy):** JVM이 프로그램을 실행하다가 이 코드를 **처음 만나는 순간**, `java.lang.invoke.LambdaMetafactory`를 호출.
3. **즉석 클래스 생성:** 메타팩토리는 런타임에 메모리 상에서 해당 인터페이스를 구현한 클래스를 **즉석에서 만들고(Generate)** 객체를 반환합니다.

> * **최적화 유연성:** 만약 컴파일 시점에 클래스를 고정해버리면(익명 클래스처럼), 나중에 JVM이 더 좋은 방식으로 람다를 처리하고 싶어도 바꿀 수 없음. `invokedynamic`을 쓰면, 자바 버전에 따라 JVM 내부 전략만 바꾸면 성능 자동 개선 가능.
> 

---

### 3. 성능 최적화: 캡처링(Capturing) vs 비캡처링(Non-Capturing)

람다는 외부 변수를 사용하느냐에 따라 성능 차이가 확연히 갈림.

#### (1) 비캡처링 람다 (Non-Capturing Lambda)

외부 변수를 전혀 쓰지 않는 순수한 람다입니다.

```java
Function<String, String> f = (s) -> s.toUpperCase(); 
// 외부에 의존하는 것이 없음

```

* **최적화:** JVM은 이 람다를 **싱글톤(Singleton)**으로 만듬.
* **성능:** 코드가 100만 번 실행되어도 **객체는 딱 1개**만 생성되어 재사용됨. 익명 클래스(`new` 할 때마다 생성)보다 훨씬 빠르고 메모리를 아낌.

#### (2) 캡처링 람다 (Capturing Lambda)

외부의 지역 변수나 인스턴스 변수를 사용하는 경우.

```java
int factor = 10;
Function<Integer, Integer> f = (i) -> i * factor; 
// 외부의 factor 변수를 캡처함

```

* **동작:** 실행될 때마다 **새로운 객체를 생성**. (외부 변수 `factor`의 값을 가지고 있어야 하므로)
* **성능:** 익명 클래스와 비슷하게 객체 생성 비용이 듬. 따라서 **불필요한 캡처링은 피하는 것**이 성능 튜닝에서 중요.

---

### 4. 렉시컬 스코프 (Lexical Scoping): `this`의 의미 변화

익명 클래스와 람다의 결정적인 차이는 `this`가 가리키는 대상.

* **익명 클래스:** `this` = **익명 클래스 자기 자신**의 인스턴스
* **람다:** `this` = **람다를 감싸고 있는 외부 클래스**의 인스턴스

```java
public class ThisTest {
    public void test() {
        // 1. 익명 클래스
        Runnable r1 = new Runnable() {
            public void run() {
                System.out.println(this); // 결과: ThisTest$1 (익명 객체 자신)
            }
        };

        // 2. 람다
        Runnable r2 = () -> {
            System.out.println(this); // 결과: ThisTest (외부 클래스 인스턴스)
        };
    }
}

```

**의의:** 람다는 별도의 스코프(Scope)를 만들지 않고, 마치 함수가 그 자리에 그대로 있는 것처럼 동작. 이를 **렉시컬 스코프(Lexical Scope)**라고 함.

---

### 5. 람다의 바이트코드 변환 과정

1. **소스 코드:** `() -> System.out.println("A")`
2. **컴파일러:**
* 람다 본문을 `private static` 메서드로 분리 (`desugar` 과정).
* 호출 위치에 `invokedynamic` 명령어를 배치.


3. **런타임 (JVM):**
* `invokedynamic` 호출 감지.
* `LambdaMetafactory`가 `CallSite` 객체 생성.
* 메서드 핸들(Method Handle)을 통해 분리해둔 `private` 메서드와 연결.



---

### 6. 결론

1. **람다는 익명 클래스의 문법적 단축이 아니다.** `invokedynamic`을 사용하여 런타임에 클래스를 동적으로 생성하는 완전히 다른 기술이다.
2. **외부 변수를 안 쓰는(Stateless) 람다가 최고다.** 이 경우 싱글톤으로 최적화되어, 기존 익명 클래스보다 압도적으로 성능이 좋다.
3. **`this`가 헷갈리지 않는다.** 람다 안에서의 `this`는 그냥 바깥쪽 클래스를 가리킨다.

이제 템플릿 콜백 패턴을 짤 때, **"이 콜백이 외부 변수를 캡처하는가?"**를 의식하면서 코드를 짠다면 성능까지 고려할 수 있음.
