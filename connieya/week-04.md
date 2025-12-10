# 4주차 정리

## 수동 테스트의 한계

- `Client.java`를 실행해 콘솔 로그를 눈으로 확인하는 방식은 수동 테스트다.
- 사람이 직접 확인해야 해서 불편하고, 대상 코드가 많아질수록 검증 범위와 시간이 폭증한다.
- UI까지 포함해 손으로 확인하면 실패 지점을 추적하기 어렵고 부정확하다.
- 반복 실행 시 일관성이 떨어지고 회귀(regression)를 잡기 어렵다.

## 왜 자동화 테스트인가

- 실행과 검증을 코드로 작성해 자동으로 돌리면 빠른 피드백을 얻는다.
- 실패 지점을 좁혀서 원인 파악이 쉬워지고, 회귀를 안정적으로 감지할 수 있다.
- 작은 단위로 쪼개면(단위 테스트) 실패가 난 곳을 즉시 파악 가능하다.

## 작게 쪼개서 검증하기

- 외부 자원(네트워크, DB)을 인터페이스/주입으로 분리하고 Mock으로 대체.
- 한 테스트는 한 가지 규칙만 검증하여 실패 메시지가 명확하도록 작성.
- 테스트 코드 작성과 실행을 개발 흐름의 일부로 반복한다.

## JUnit 5

- @Test 테스트 메소드
- @BeforeEach 테스트
- 각 테스트 전에 실행 된다.
- 테스트마다 새로운 인스턴스가 생성된다.

## List 와 불변 리스트 (Immutable List)

```java
public class Sort {

    public List<String> sortByLength(List<String> list) {
        list.sort((s1, s2) -> s1.length() - s2.length());
        return list;
    }
}

```

위 메서드를 테스트 하려고 아래와 같이 작성 했을 때

```java
sort.sortByLength(List.of("aa", "b"));
```

다음 오류가 발생한다.

```java
java.lang.UnsupportedOperationException
```

왜 이런 오류가 발생할까?

### List 인터페이스란?

Java 의 List 인터페이스는 순서(index)가 있는 데이터 집합을 다루기 위한 규약이다.

> List 인터페이스는 내부 자료구조가 배열인지 또는 연결 리스트인지 규정하지 않는다.
> 실제 동작 방식은 각 구현체가 결정한다.

### List 주요 구현체

| 구현체               | Java 버전 | 내부 구조               | 특징                     | add/remove | set/get | 불변성                |
| -------------------- | --------- | ----------------------- | ------------------------ | ---------- | ------- | --------------------- |
| ArrayList            | 1.2+      | 동적 배열(Object[])     | 가장 일반적              | ⭕         | ⭕      | 가변(Mutable)         |
| LinkedList           | 1.2+      | 이중 연결 리스트        | 삽입/삭제 빠름           | ⭕         | ⭕      | 가변(Mutable)         |
| Arrays.asList()      | 1.2+      | 원본 배열 참조(배열 뷰) | 크기 고정 리스트         | ❌         | ⭕      | 크기-고정(Fixed-size) |
| List.of()            | 9+        | JVM 최적화 불변 구조    | 완전 불변 리스트         | ❌         | ❌      | 완전 불변(Immutable)  |
| CopyOnWriteArrayList | 5+        | 배열 복사 기반          | 쓰기 적고 읽기 많은 환경 | ⭕         | ⭕      | 가변(Mutable)         |

### Arrays.asList() - 배열 기반 크기-고정 리스트

작동 원리 : 배열 뷰(view)

```java
String[] arr = {"aa", "b"};
List<String> list = Arrays.asList(arr);  // 배열을 감싼 뷰 생성
```

#### 정렬이 가능한 이유

```java
List<String> list = Arrays.asList("aa", "b");
sort.sortByLength(list);

```

실행해도 에러가 나지 않는다.

그 이유는 배열 기반 리스트 (backed-by-array) 이기 때문이다.

- list 내부에 원본 배열(arr) 이 그대로 저장됨
- list.set(i, value) → 실제로 arr[i] = value 실행
- 정렬(sort)은 내부에서 요소 교환(swap)을 위해 set()을 호출함 → 정상 동작
- 따라서 요소 변경은 가능하나, 크기 변경은 불가

### Arrays.asList() 사용 시 주의사항

#### 크기 변경 (add/remove) 불가

배열 크기는 고정이므로 다음은 예외를 던진다.

```java
List<String> list = Arrays.asList("a", "b");
list.add("c");    // UnsupportedOperationException
list.remove(0);   // UnsupportedOperationException
```

#### 원본 배열과의 공유 (공유 참조)

```java
String[] arr = {"a", "b"};
List<String> list = Arrays.asList(arr);

// List 변경 → 배열 반영
list.set(0, "x");
System.out.println(arr[0]);  // "x"

// 배열 변경 → List 반영
arr[1] = "y";
        System.out.println(list.get(1));  // "y"
```

#### 반환되는 List 는 ArrayList 가 아니다.

```java
List<String> list = Arrays.asList("a", "b");
System.out.println(list.getClass().getName());
// "java.util.Arrays$ArrayList" (내부 클래스)
// java.util.ArrayList와 다름!
```

이것은 java.util.Arrays$ArrayList 라는 내부 클래스이며
java.util.ArrayList 와 다르다.

=> java.util.Arrays의 private static 내부 클래스

#### null 허용

```java
List<String> list = Arrays.asList("a", null, "b");  // 가능
```

### List.of() - 완전 불변 리스트 (Java 9+)

작동 원리: JVM 최적화 불변 구조

```java
List<String> list = List.of("aa", "b");  // 완전 불변 리스트 생성
```

#### ❌ 정렬이 불가능한 이유

```java
sort.sortByLength(List.of("aa", "b"));  // UnsupportedOperationException
```

내부 메커니즘:

- 완전 불변(Immutable) 설계
- set(), add(), remove() 메서드 자체가 구현되지 않음
- 변경 시도 시 바로 예외 발생
- 내부 데이터 구조는 JVM이 최적화 (배열과 독립적

### 특징

#### 완전 불변성

```java
List<String> immutable = List.of("a", "b");
immutable.set(0, "x");    // UnsupportedOperationException
immutable.add("c");       // UnsupportedOperationException
immutable.remove(0);      // UnsupportedOperationException
```

#### 원본 배열과 독립적

```java
String[] arr = {"a", "b"};
List<String> list = List.of(arr);

arr[0] = "x";  // 배열 변경
System.out.println(list.get(0));  // 여전히 "a" (변경되지 않음)
```

#### null 불허용

```java
List<String> list = List.of("a", null, "b");  // NullPointerException
```

#### 최적화된 메모리 사용

- 요소 개수에 따라 특별한 내부 클래스 사용 (List0, List1, List2, ListN)
- 0-2개 요소는 전용 객체로 최적화

List.of()는 요소 개수에 따라 특별히 최적화된 내부 클래스를 사용
=> 불필요한 배열 할당을 피하고 메모리 사용을 최소화하기 위한 JVM의 고도화된 최적화 전략

List0 (빈 리스트)

- 단 하나의 싱글톤 인스턴스만 존재

List12 (1~2개 요소)

- 배열을 사용하지 않고 필드 2개에 직접 저장하는 구조

-> 배열 할당 비용 없음,  GC pressure 감소


ListN (3개 이상)
- 내부적으로 Object[] 배열 사용


### Arrays.asList vs List.of

| 비교 항목      | Arrays.asList()                          | List.of()                               |
| -------------- | ---------------------------------------- | --------------------------------------- |
| Java 버전      | 1.2+                                     | 9+                                      |
| 불변성         | ❌ 크기-고정(Fixed-size), 요소 변경 가능 | ⭕ 완전 불변(Immutable), 모든 변경 불가 |
| set()          | ⭕ 가능 (원본 배열 변경)                 | ❌ 불가 (예외 발생)                     |
| add()/remove() | ❌ 불가 (예외 발생)                      | ❌ 불가 (예외 발생)                     |
| 원본 배열 공유 | ⭕ 배열 뷰 - 동일 데이터 참조            | ❌ 독립적 복사본 생성                   |
| null 허용      | ⭕ 허용                                  | ❌ NullPointerException                 |
| 정렬 가능      | ⭕ 가능 (set() 사용)                     | ❌ 불가 (예외 발생)                     |
| 메모리         | 가벼움 (배열 참조만)                     | 상황별 최적화                           |
| 반환 타입      | java.util.Arrays$ArrayList               | java.util.ImmutableCollections$ListN 등 |


### 실무에서의 올바른 사용 패턴

불변 상수 리스트 (읽기 전용)

```java
// 통화 코드, 국가 코드 등 불변 데이터
public static final List<String> CURRENCIES = List.of("USD", "EUR", "JPY");
public static final List<String> COUNTRIES = List.of("KR", "US", "JP");
```


배열을 일시적 리스트로 변환 -  원본 배열과의 연결 유지

```java
String[] userArray = getUserArrayFromExternal();
List<String> tempList = Arrays.asList(userArray);
// 원본 배열과 동기화 필요할 때만 사용
```

가변 리스트로 변환하여 사용

```java
// 정렬이 필요한 경우
List<String> mutableList = new ArrayList<>(List.of("aa", "b"));
sort.sortByLength(mutableList);  // 정상 동작

// 수정이 필요한 경우  
List<String> modifiable = new ArrayList<>(Arrays.asList("a", "b"));
modifiable.add("c");  // 가능
```

메서드 파라미터 전달 시

```java
// 읽기만 하는 경우 - 안전
processItems(List.of("item1", "item2"));

// 수정이 필요한 경우 - 명시적 변환
processItems(new ArrayList<>(List.of("item1", "item2")));
```


❌ 피해야 할 패턴


```java
// 위험: 원본 배열과의 불필요한 공유
String[] config = loadConfig();
List<String> configList = Arrays.asList(config);
// config 배열이 다른 곳에서 수정되면 configList도 영향 받음

// 위험: 불변 리스트 수정 시도
List<String> constants = List.of("A", "B", "C");
constants.sort(comparator);  // 런타임 예외

// 권장: 명시적 변환
List<String> safeList = new ArrayList<>(List.of("A", "B", "C"));
safeList.sort(comparator);  // 안전
```