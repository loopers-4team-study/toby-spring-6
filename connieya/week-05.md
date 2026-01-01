# 5주차 - 콜백(Callback)과 템플릿/콜백 패턴

---

## 1. 콜백(Callback)이란?

> **나중에 호출되도록 전달되는 실행 가능한 코드**

- 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달
- 값을 참조하기 위한 것이 아니라 **특정 로직을 담은 코드를 실행**시키는 것이 목적

### 콜백의 형태

| 언어       | 콜백의 형태                |
| ---------- | -------------------------- |
| JavaScript | 콜백 **함수**              |
| Java       | 콜백 **객체** (인터페이스) |

### 핵심 키워드

> **"매개변수를 통해 전달한다"**

그런데... 매개변수로 전달하려면 뭔가 조건이 필요하지 않을까? 🤔

---

## 2. 일급 객체(First-Class Object)

> **값처럼 다룰 수 있는 객체**

"매개변수로 전달"이라는 키워드를 보니 **일급 객체**가 떠올랐다.

### 일급 객체의 4가지 조건

| 조건 | 설명                                                | JavaScript 예시                |
| ---- | --------------------------------------------------- | ------------------------------ |
| 1    | 무명의 리터럴로 생성할 수 있다 (런타임 생성 가능)   | `function() { return 1; }`     |
| 2    | 변수나 자료구조에 저장할 수 있다                    | `const add = (a, b) => a + b;` |
| 3    | **함수의 매개변수에 전달할 수 있다** ← 콜백과 연결! | `arr.map(x => x * 2)`          |
| 4    | 함수의 반환값으로 사용할 수 있다                    | `return function() { ... }`    |

---

---

## 3. JavaScript - 함수가 일급 객체

JavaScript에서 함수는 일급 객체이다.

```javascript
// 1. 무명 리터럴로 생성
const add = function (a, b) {
  return a + b;
};

// 2. 변수에 저장
const multiply = (a, b) => a * b;

// 3. 매개변수로 전달 (콜백!)
function template(callback) {
  console.log("작업 시작");
  callback(); // 전달받은 함수를 호출
  console.log("작업 끝");
}
template(() => console.log("콜백 실행!"));

// 4. 반환값으로 사용
function createMultiplier(x) {
  return function (n) {
    return n * x;
  };
}
```

→ 함수가 일급 객체이므로 **콜백 함수**를 직접 전달할 수 있다!

---

## 4. 함수형 프로그래밍 vs 객체지향 프로그래밍

일급 객체, JavaScript 하면 **함수형 프로그래밍**이 떠오른다.

| 패러다임                | 핵심               | 함수가 일급 객체? | 대표 언어           |
| ----------------------- | ------------------ | ----------------- | ------------------- |
| **함수형 프로그래밍**   | 함수 중심, 불변성  | ✅                | JavaScript, Haskell |
| **객체지향 프로그래밍** | 객체와 클래스 중심 | ❌                | Java, C++           |

### 함수형 프로그래밍의 특징

```javascript
// 고차 함수: 함수를 인자로 받거나 반환하는 함수
const numbers = [1, 2, 3, 4, 5];

numbers
  .filter((n) => n % 2 === 0) // 콜백 함수
  .map((n) => n * 2) // 콜백 함수
  .reduce((a, b) => a + b); // 콜백 함수
```

---

## 5. Java는 어떤가?

Java는 **객체지향 프로그래밍** 언어이다.

### Java에서 함수는 일급 객체가 아니다

```java
// ❌ 함수를 직접 변수에 저장할 수 없음
int add(int a, int b) { return a + b; }
??? func = add;  // 불가능!

// ❌ 함수를 직접 매개변수로 전달할 수 없음
template(add);  // 불가능!
```

### Java 8 - 람다의 등장

하지만 Java 8부터 **람다**가 도입되면서 **함수형 스타일**로 코딩이 가능해졌다.

```java
// Java 7 이전: 익명 클래스 (장황함)
list.sort(new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
});

// Java 8 이후: 람다 (간결함)
list.sort((a, b) -> a.length() - b.length());
```

### 주의: 람다도 내부적으로는 객체

```java
// 람다처럼 보이지만, 실제로는 Comparator 객체
Comparator<String> comp = (a, b) -> a.length() - b.length();
```

> Java는 객체지향 언어이지만, 람다 덕분에 **함수형 스타일**로 구현할 수 있다.

---

## 6. Java에서의 콜백 - 인터페이스로 우회

Java에서는 함수를 직접 전달할 수 없으므로 **인터페이스(객체)**로 감싸서 전달한다.

```java
// 콜백 인터페이스 정의
public interface ApiExecutor {
    String execute(URI uri) throws IOException;
}

// 콜백을 매개변수로 받음
public BigDecimal getExRate(String url, ApiExecutor apiExecutor) {
    String response = apiExecutor.execute(uri);  // 콜백 호출
}
```

### 비교 정리

| 언어       | 함수가 일급 객체? | 콜백 구현 방식               |
| ---------- | ----------------- | ---------------------------- |
| JavaScript | ✅                | 함수 직접 전달               |
| Python     | ✅                | 함수 직접 전달               |
| **Java**   | ❌                | **인터페이스로 감싸서 전달** |

---

## 7. 템플릿/콜백 패턴

> **메소드 하나만 가진 전략 인터페이스를 사용하는 전략 패턴**

### 전략 패턴과의 관계

| 전략 패턴 | 템플릿/콜백 |
| --------- | ----------- |
| 컨텍스트  | 템플릿      |
| 전략      | 콜백        |

### 구조

```
┌─────────────────────────────────────────┐
│              ApiTemplate                │
│  ┌───────────────────────────────────┐  │
│  │  1. URI 생성 (고정)               │  │
│  │  2. apiExecutor.execute() (콜백)  │  │
│  │  3. exRateExtractor.extract() (콜백) │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

- **템플릿**: 고정된 흐름 (URI 생성 → API 호출 → 응답 파싱)
- **콜백**: 변하는 부분 (API 호출 방식, 파싱 방식)

---

## 8. 코드 예시

### 템플릿

```java
public class ApiTemplate {
    public BigDecimal getExRate(String url, ApiExecutor apiExecutor, ExRateExtractor exRateExtractor) {
        URI uri = new URI(url);                          // 고정
        String response = apiExecutor.execute(uri);      // 콜백 1
        return exRateExtractor.extract(response);        // 콜백 2
    }
}
```

### 콜백 인터페이스

```java
public interface ApiExecutor {
    String execute(URI uri) throws IOException;
}

public interface ExRateExtractor {
    BigDecimal extract(String response) throws JsonProcessingException;
}
```

### 콜백 구현체

```java
// HttpClient 사용
public class HttpClientApiExecutor implements ApiExecutor {
    @Override
    public String execute(URI uri) throws IOException {
        HttpClient client = HttpClient.newBuilder().build();
        HttpRequest request = HttpRequest.newBuilder().uri(uri).GET().build();
        return client.send(request, HttpResponse.BodyHandlers.ofString()).body();
    }
}

// URLConnection 사용
public class SimpleApiExecutor implements ApiExecutor {
    @Override
    public String execute(URI uri) throws IOException {
        HttpURLConnection connection = (HttpURLConnection) uri.toURL().openConnection();
        try (BufferedReader br = new BufferedReader(new InputStreamReader(connection.getInputStream()))) {
            return br.lines().collect(Collectors.joining());
        }
    }
}
```

### 람다로 간결하게

```java
// 익명 클래스 (Java 8 이전)
apiTemplate.getExRate(url, new ApiExecutor() {
    @Override
    public String execute(URI uri) throws IOException {
        return httpClient.send(...);
    }
});

// 람다 (Java 8 이후)
apiTemplate.getExRate(url, uri -> httpClient.send(...));
```

---

## 9. 템플릿/콜백의 장점

| 장점              | 설명                                |
| ----------------- | ----------------------------------- |
| **관심사의 분리** | 템플릿은 흐름, 콜백은 로직에 집중   |
| **코드 재사용**   | 템플릿 한 번 작성, 여러 콜백과 조합 |
| **유연성**        | 새로운 구현체 추가 용이 (OCP)       |
| **테스트 용이**   | 콜백을 Mock으로 교체 가능           |

---

## 10. Spring에서의 템플릿/콜백

Spring Framework의 대표적인 템플릿/콜백 패턴 적용 예시

- `JdbcTemplate`
- `RestTemplate`
- `TransactionTemplate`
- `RedisTemplate`

---
