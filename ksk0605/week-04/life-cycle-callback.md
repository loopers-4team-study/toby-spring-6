JUnit 5의 **생명주기(Lifecycle)**와 **콜백(Callback)**은 테스트의 실행 순서, 인스턴스 생성 방식, 그리고 테스트 실행 전후에 관여하는 확장 기능을 제어하는 핵심 개념입니다. 제공된 자료를 바탕으로 이 두 가지 개념을 상세히 설명해 드리겠습니다.

### 1. 테스트 인스턴스 생명주기 (Test Instance Lifecycle)

JUnit 5는 테스트 간의 격리성을 보장하기 위해 기본적으로 **메서드 단위**의 생명주기를 가집니다. 하지만 필요에 따라 **클래스 단위**로 변경할 수도 있습니다.

*   **메서드 단위 (PER_METHOD - 기본값):**
    *   JUnit은 각 **테스트 메서드**를 실행하기 전에 해당 테스트 클래스의 **새로운 인스턴스**를 생성합니다.
    *   이는 개별 테스트가 서로의 상태(인스턴스 변수 등)에 영향을 주지 않도록 하여 부작용(Side effects)을 방지하기 위함입니다.
    *   `@Disabled` 등으로 비활성화된 테스트 메서드라 하더라도, 조건에 따라 인스턴스가 생성될 수 있습니다.

*   **클래스 단위 (PER_CLASS):**
    *   테스트 클래스당 **하나의 인스턴스**만 생성하여 모든 테스트 메서드가 이를 공유합니다.
    *   이 모드를 사용하려면 클래스에 `@TestInstance(Lifecycle.PER_CLASS)`를 명시하거나 구성 파일(`junit-platform.properties`)을 통해 기본값을 변경해야 합니다,.
    *   **장점:**
        *   인스턴스 상태를 공유하므로 초기화 비용이 큰 작업에 유리할 수 있으나, `@BeforeEach`나 `@AfterEach`에서 상태를 초기화해야 할 수도 있습니다.
        *   `@BeforeAll`과 `@AfterAll` 어노테이션을 **static이 아닌 일반 메서드**나 인터페이스의 default 메서드에도 사용할 수 있게 됩니다.
        *   `@Nested` 내부 클래스에서도 `@BeforeAll`/`@AfterAll`이 동작하게 할 수 있습니다.

### 2. 표준 생명주기 어노테이션 (Standard Lifecycle Annotations)

사용자가 테스트 클래스 내에 직접 정의하는 코드로, 실행 시점은 다음과 같습니다.

*   **`@BeforeAll`**: 현재 클래스의 **모든** 테스트 메서드 실행 **전**에 한 번 실행됩니다. 기본적으로 `static`이어야 합니다 (PER_CLASS 모드 제외).
*   **`@BeforeEach`**: **각각의** 테스트 메서드(`@Test`, `@RepeatedTest` 등) 실행 **전**에 실행됩니다.
*   **`@Test`**: 실제 테스트 로직이 담긴 메서드입니다.
*   **`@AfterEach`**: **각각의** 테스트 메서드 실행 **후**에 실행됩니다.
*   **`@AfterAll`**: 현재 클래스의 **모든** 테스트 메서드 실행 **후**에 한 번 실행됩니다. 기본적으로 `static`이어야 합니다.

### 3. 확장 모델과 콜백 (Extension Model & Callbacks)

JUnit 5의 강력한 점은 `Extension` API를 통해 테스트 생명주기의 특정 지점에 코드를 삽입할 수 있다는 것입니다. 이는 JUnit 4의 `Runner`나 `Rule`보다 유연하고 일관된 모델을 제공합니다.

#### 주요 콜백 인터페이스
확장 기능(Extension)을 구현할 때 다음 인터페이스들을 사용하여 특정 시점에 개입할 수 있습니다,.

*   `BeforeAllCallback` / `AfterAllCallback`: 컨테이너(클래스)의 전체 실행 전후
*   `BeforeEachCallback` / `AfterEachCallback`: 각 테스트 실행 전후
*   **`BeforeTestExecutionCallback` / `AfterTestExecutionCallback`**: 테스트 메서드 실행 **직전과 직후**.
    *   *차이점:* `@BeforeEach`는 테스트 준비(Setup)를 위한 것이라면, 이 콜백은 타이밍 측정이나 로깅처럼 실제 테스트 실행 '순간'을 감싸는 데 적합합니다.
*   `TestExecutionExceptionHandler`: 테스트 중 발생한 예외를 처리(무시하거나 다시 던짐)할 수 있습니다.

### 4. 사용자 코드와 확장의 실행 순서 (Relative Execution Order)

테스트 클래스를 실행할 때, 사용자 코드(어노테이션이 붙은 메서드)와 등록된 확장(Extension) 콜백은 정해진 순서대로 인터리빙(Interleaving) 되어 실행됩니다.

단일 테스트 메서드 실행 시의 흐름은 다음과 같은 **12단계**로 이루어집니다 -:

1.  **Extension:** `BeforeAllCallback`
2.  **User:** `@BeforeAll`
3.  **Extension:** `BeforeEachCallback`
4.  **User:** `@BeforeEach`
5.  **Extension:** `BeforeTestExecutionCallback` (테스트 실행 직전)
6.  **User:** `@Test` (실제 테스트)
7.  **Extension:** `TestExecutionExceptionHandler` (예외 발생 시)
8.  **Extension:** `AfterTestExecutionCallback` (테스트 실행 직후)
9.  **User:** `@AfterEach`
10. **Extension:** `AfterEachCallback`
11. **User:** `@AfterAll`
12. **Extension:** `AfterAllCallback`

**핵심 인사이트:** 확장(Extension) 코드가 사용자 코드를 감싸는 형태로 실행됩니다. 예를 들어, `BeforeAll` 단계에서는 확장이 먼저 실행되지만, `AfterAll` 단계에서는 사용자 코드가 먼저 실행되고 확장이 나중에 실행됩니다.

### 5. 특수 시나리오 (다이나믹 테스트 및 파라미터 테스트)

*   **다이나믹 테스트 (Dynamic Tests):** `@TestFactory`로 생성되는 다이나믹 테스트는 일반적인 생명주기 콜백을 따르지 않습니다. `@BeforeEach`나 `@AfterEach`는 `@TestFactory` 메서드 자체에 대해서는 실행되지만, 생성된 **개별 다이나믹 테스트마다 실행되지는 않습니다**.
*   **파라미터화된 테스트 (Parameterized Tests):** 각 파라미터에 의한 호출(Invocation)은 일반 `@Test` 메서드와 동일한 생명주기를 가집니다. 즉, 매 호출마다 `@BeforeEach`/`@AfterEach`가 실행됩니다.

---

### 💡 이해를 돕기 위한 비유: 고급 레스토랑의 코스 요리

JUnit 5의 생명주기와 콜백을 **레스토랑의 코스 요리**에 비유해 보겠습니다.

*   **테스트 인스턴스 (PER_METHOD):** 손님 한 명(테스트 메서드)이 올 때마다 **새로운 테이블**을 세팅해 줍니다. 앞사람이 흘린 음식(이전 상태)이 다음 사람에게 영향을 주지 않기 위함입니다.
*   **`@BeforeAll` / `BeforeAllCallback`:** 레스토랑 문을 열고 조명을 켜는 작업입니다. 모든 손님이 오기 전에 딱 한 번만 수행합니다. 확장이 먼저 "문 열 준비"를 하고, 사용자(매니저)가 "영업 시작"을 선언합니다.
*   **`@BeforeEach`:** 각 요리가 나오기 전에 커틀러리(포크, 나이프)를 세팅하는 과정입니다.
*   **`BeforeTestExecutionCallback`:** 웨이터가 접시를 테이블에 내려놓기 **직전**에 "맛있게 드세요"라고 말하는 순간입니다.
*   **`@Test`:** 손님이 실제로 음식을 먹는(테스트 수행) 행위입니다.
*   **`AfterTestExecutionCallback`:** 손님이 식사를 마치자마자 웨이터가 접시를 치우는 순간입니다.
*   **`@AfterEach`:** 다음 요리를 위해 테이블을 닦고 정돈하는 과정입니다.
*   **`@AfterAll` / `AfterAllCallback`:** 영업이 끝나고 문을 닫는 작업입니다.

이처럼 JUnit 5는 정교하게 짜인 순서에 따라 테스트라는 '요리'가 완벽하게 서빙되도록 보장합니다.