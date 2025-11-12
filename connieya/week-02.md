## 2주차 스터디 정리

### 기존 접근: 상속 기반 템플릿 메서드

- `PaymentService`에서 환율 조회와 결제 준비 흐름을 모두 담당하고, 세부 환율 로직은 하위 클래스(`SimpleExRatePaymentService`, `WebApiExRatePaymentService`)가 구현하도록 템플릿 메서드 패턴을 적용했음.
- 결제 준비 로직을 재사용할 수 있다는 장점은 있었지만, 새로운 환율 정책이 필요하면 하위 클래스를 추가하거나 기존 클래스를 수정해야 했음.
- 상위 클래스 변화에 따라 하위 구현 전체가 민감하게 흔들리는 구조라 결합도가 높았고, 파트너사가 JAR로 제공받을 때 환율 부분만 교체하기 어려웠음.

<details>
<summary>상속의 단점 </summary>

- 부모 구현을 그대로 물려받는 화이트박스 재사용이라 내부 구현을 세세하게 알아야 안전하게 확장할 수 있음.
- 부모 변경이 곧바로 자식 클래스 전체에 파급되므로 구조가 쉽게 경직되고, 결합도가 높아짐.
- 단일 상속만 가능하기 때문에 한 번 선택한 계층에 발이 묶이고 다른 축의 역할을 추가하기 어려움.
- 새로운 요구가 생길 때마다 추상 메서드를 추가하거나 계층을 늘려야 해서 클래스 수가 기하급수적으로 증가할 위험이 있음.

</details>

<br/>

<details>
<summary>한계 인식</summary>

- 환율 정책이 바뀔 때마다 하위 클래스를 새로 만들거나 기존 클래스를 수정해야 해서 배포 단위(JAR) 전체를 교체해야 했음.

_(예: `WebApiExRatePaymentService` 대신 파트너사의 내부 환율 API를 쓰려면 새 하위 클래스를 작성하고 JAR를 다시 배포해야 했음.)_

- 상속 계층이 깊어지면서 결제 준비와 환율 전략이 한 클래스에 묶여 있어 역할이 뒤섞이고 테스트가 어려워졌음.

_(예: 템플릿 메서드 안에서 환율 API 호출과 금액 계산이 함께 있어 Mock 주입이나 단위 테스트가 복잡해졌음.)_

- 클라이언트 코드가 어떤 환율 구현을 사용할지 직접 결정해야 해서 실행 책임과 조립 책임이 분리되지 않았음.

_(예: `Client`에서 `new WebApiExRatePaymentService()`를 명시해야 했고, 샘플 환율로 바꾸려면 클라이언트 코드를 수정해야 했음.)_

</details>

### 리팩터링 : 인터페이스 + 조립으로 전환

- 환율 조회 책임을 `ExRateProvider` 인터페이스로 분리하고, 실제 구현(`SimpleExRateProvider`, `WebApiExRateProvider`)은 전략처럼 교체 가능하게 설계.
- `PaymentService`는 더 이상 상속 기반 템플릿이 아니라, 생성자를 통해 `ExRateProvider`를 주입받는 형태로 변경하여 OCP 준수.
- 객체 간 관계 설정은 `ObjectFactory`에서 담당해 `Client`는 실행만 책임지도록 관심사를 분리함.

### 📌 상속과 합성

<details>
<summary>재사용 관점에서 상속 vs 합성</summary>

- 상속: 부모 클래스의 구현 자체를 물려받아 재사용(화이트박스 재사용)
    - → 부모 코드를 그대로 활용할 수 있지만, 내부 구현을 이해하고 유지해야 함.
    - → 상속 계층이 깊어질수록 부모 변경이 자식 전체에 파급됨.
- 합성: 포함된 객체의 퍼블릭 인터페이스에 의존해 기능을 조합(블랙박스 재사용)
    - → 구현 세부사항을 몰라도 계약만 맞으면 다른 전략으로 손쉽게 교체 가능.
    - → 테스트 더블/모의 객체를 주입하기 좋아 단위 테스트가 단순해짐.

</details>

<details>
<summary>결합도 관점</summary>

- 상속: 부모 내부 구현을 알아야 하고 변경이 파급돼 결합도가 높음.
    - ```java
    public class WebApiExRatePaymentService extends PaymentService {
        @Override
        protected BigDecimal getExchangeRate(String currency) {
            // 부모 템플릿 메서드 내부 동작에 의존
            return callExternalApi(currency);
        }
    }
    ```
- 합성: 인터페이스 계약에만 의존해 구현 변경이 닿지 않으므로 결합도가 낮음.

    - ```java
    public class PaymentService {
        private final ExRateProvider exRateProvider;

        public PaymentService(ExRateProvider exRateProvider) {
            this.exRateProvider = exRateProvider;
        }

        public Payment prepare(...) {
            BigDecimal rate = exRateProvider.getExRate(...);
            // ...
        }
    }
    ```

</details>

<details>
<summary>확장/교체 용이성</summary>

- 상속: 단일 상속 제약 때문에 다른 축으로 확장하거나 동적으로 교체하기 어려움.
    - → 예: 이미 `PaymentService`를 상속 중이면 다른 베이스 클래스를 추가로 상속할 수 없음.
- 합성: 전략 구현을 갈아 끼우기 쉬워 테스트, 고객 맞춤형 정책 적용에 유리함.

    - ```java
    ExRateProvider provider = new SimpleExRateProvider();
    PaymentService paymentService = new PaymentService(provider);

    // 필요 시 다른 구현으로 교체
    paymentService = new PaymentService(new WebApiExRateProvider());
    ```

</details>

<details>
<summary>이번 프로젝트에 적용했을 때 차이</summary>

- 상속 기반 템플릿에서는 `PaymentService`를 확장해야 환율 정책을 바꿀 수 있었음.
    - → 파트너사 맞춤 환율 정책을 적용하려면 새 하위 클래스를 작성해 배포해야 했음.
- 합성 구조로 바꾸면서 `PaymentService`는 `ExRateProvider` 인터페이스에만 의존하고, 환율 구현체는 외부에서 주입해 자유롭게 교체 가능해짐.
    - → `ObjectFactory`에서 `SimpleExRateProvider`, `WebApiExRateProvider` 중 원하는 전략을 조립하여 공급.

</details>

<details>
<summary>컴파일 타임 vs 런타임 의존성</summary>

- 상속: 컴파일 타임에 부모 타입이 고정되므로 의존성이 정적으로 엮임.
    - → 빌드 시 이미 `extends PaymentService` 같은 선언으로 부모 구현에 종속.
- 합성: 생성 시점에 어떤 구현을 주입하느냐에 따라 런타임 의존성이 결정됨.
    - → `new PaymentService(new WebApiExRateProvider())`처럼 실행 시 전략을 선택 가능.
    - → 필요하면 DI 컨테이너/팩토리가 런타임 구성으로 다른 구현을 공급할 수 있음.

</details>

### 최종 구조

- `PaymentService`는 환율 제공자에 대한 의존성을 인터페이스 형태로만 알고 있으며, 결제 준비 로직에 집중함.

```java
public class PaymentService {

  private ExRateProvider exRateProvider;

  public PaymentService(ExRateProvider exRateProvider) {
    this.exRateProvider = exRateProvider;
  }

  public Payment prepare(Long orderId, String currency, BigDecimal foreignCurrencyAmount) throws IOException {
    BigDecimal exRate = exRateProvider.getExRate(currency);
    BigDecimal convertedAmount = foreignCurrencyAmount.multiply(exRate);

    LocalDateTime validUntil = LocalDateTime.now().plusMinutes(30);
    return new Payment(orderId, currency, foreignCurrencyAmount, exRate, convertedAmount, validUntil);
  }
}
```

- 환율 전략은 독립된 구현체로 확장 가능하며, 필요 시 새로운 정책 클래스를 추가하면 됨.

```java
public class SimpleExRateProvider implements  ExRateProvider {

  @Override
  public BigDecimal getExRate(String currency) throws IOException {
    if (currency.equals("USD")) return BigDecimal.valueOf(1000);
    throw new IllegalArgumentException("지원되지 않는 통화입니다.");
  }
}
```

- 객체 조립 책임은 `ObjectFactory`가 맡아 `Client`는 단순 실행 역할만 담당.

```java
public class ObjectFactory {
    public PaymentService paymentService() {
        return new PaymentService(exRateProvider());
    }

    public ExRateProvider exRateProvider() {
        return new WebApiExRateProvider();
    }
}
```

- 상속은 "확장"이 주 목적이며, 단순 재사용 용도로 남용하면 구조가 쉽게 굳어버린다.
- 전략을 인터페이스로 분리하고 외부에서 주입하면 OCP와 관심사 분리를 자연스럽게 달성할 수 있다.
- "관계 설정 책임 분리"를 통해 애플리케이션 조립과 실행 책임을 분리하면 구조가 훨씬 명확해진다.

## 면접 질문

<details>
<summary>DI란 무엇인가요?</summary>

DI는 Dependency Injection의 약자로, 의존하는 대상을 객체 스스로 직접 생성하지 않고 제3자가 외부에서 주입해 주는 방식을 말합니다. 스프링 환경에서는 이 제3자가 스프링 컨테이너입니다. 컨테이너가 인터페이스에 맞는 구현체를 찾아 주입해 주므로, 합성을 자동으로 지원한다고 볼 수 있습니다.

<details>
<summary>의존성을 직접 생성하지 않고 외부에서 주입하는 이유는?</summary>

의존성을 직접 생성하면 요구사항이 조금만 바뀌어도 해당 객체를 수정해야 해서 변경에 취약합니다. 관심사도 한 곳에 엉켜버리죠. 반면 외부에서 주입받으면 추상 인터페이스에 의존하게 되어 세부 구현 변경의 영향을 받지 않습니다. 자연스럽게 관심사가 분리되고, 결과적으로 OCP(개방-폐쇄 원칙)를 지킬 수 있습니다.

</details>

</details>
