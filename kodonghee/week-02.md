## 🏃🏻‍♀️ 토비의 스프링 6 - 이해와 원리
### 섹션 3️⃣. 오브젝트와 의존관계
#### 오브젝트와 의존관계
- 오브젝트 (Object-Oriented Programming, 객체, 클래스?)
- 오브젝트: 실제로 프로그램을 실행하면 만들어져서 동작하는 무엇인가
- 클래스: 오브젝트를 만들어 내기 위해서 필요한 것 (우리는 클래스를 코딩한다.)
- 클래스(오브젝트에 대한 상세한 정보 존재, 청사진) ➡️ 오브젝트 (생성)
- 인스턴스: 앞에 나온 추상적인 것에 대한 "실체"
- 자바에서는 배열(Array)도 오브젝트
  
🟣 **클래스의 인스턴스(Class Instance) = 오브젝트**
||객체|클래스|
|--|---|---|
|정의|클래스의 인스턴스 (=메모리에 할당된 실체)|객체가 만들어질 수 있는 **템플릿** 같은 존재|
|생성 가능 개수|필요에 따라 여러번 생성 가능|한 번만 선언 가능|
|메모리 할당|생성될 때 메모리에 할당|생성될 때 메모리에 할당되지 않음|
|생성 방법|new 키워드, newInstance 메소드, 팩토리 메소드, 역직렬화|class 키워드를 통해 정의|

- 의존관계 (Dependency): A --> B
  - 클래스 사이의 의존관계 (= 코드 레벨의 의존관계)
    - Client --> Supplier
    - Client 클래스 코드는 Supplier 클래스 코드가 존재해야지만 제대로 동작 가능
    - Supplier가 변경되면 Client 코드가 영향을 받는다.
  - 오브젝트 사이의 의존관계 (= 런타임 레벨의 의존관계)
    - 클래스 레벨의 의존관계와 런타임 레벨의 의존관계가 다를 수 있다.
   
#### 관심사의 분리 (Separation of Concerns: SoC)
``` java
public Payment prepare(Long orderId, String currency, BigDecimal foreignCurrencyAmount) throws IOException {
        // 환율 가져오기
        URL url = new URL("https://open.er-api.com/v6/latest/" + currency);
        HttpURLConnection connection = (HttpURLConnection) url.openConnection();
        BufferedReader br = new BufferedReader(new InputStreamReader(connection.getInputStream()));
        String response = br.lines().collect(Collectors.joining());
        br.close();

        ObjectMapper mapper = new ObjectMapper();
        ExRateData data = mapper.readValue(response, ExRateData.class);
        BigDecimal exRate = data.rates().get("KRW");

        // 금액 계산
        BigDecimal convertedAmount = foreignCurrencyAmount.multiply(exRate);

        // 유효 시간 계산
        LocalDateTime validUntil = LocalDateTime.now().plusMinutes(30);

        return new Payment(orderId, currency, foreignCurrencyAmount, exRate, convertedAmount, validUntil);
    }
```
- prepare 메소드에 두 가지 관심사가 존재함
- 환율 가져오는 방식이 변경되는 시점과 원화 환산 로직 변경 시점이 다름 ➡️ 분리되어야 함
- 코드가 무슨 역할을 하는지 주석 없이 잘 이해가 되어야 함

#### 상속을 통한 확장
- 관심사 분리에 이어 PaymentService 클래스에서 getExRate 기능을 분리해야 함
  - **재사용**이 가능하도록 하는 것이 좋음
  - 재사용: 소스코드를 건드리지 않고 계속해서 가져다 사용할 수 있어야 함 (ex) 라이브러리)
- 환율을 가져오는 방식이 변경되어도 PaymentService 코드는 변하지 않도록 하는 것이 좋음 (확장성 🆙)
- 상속을 이용: 어떤 기능을 확장해서 사용할 수 있도록 만들어줌
  - PaymentService 안에서 getExRate 메소드를 구체적으로 구현하지 않아도 되게 abstract 메소드로 변경 ➡️ PaymentService 클래스도 abstract로 변경 (PaymentService에 new 키워드 붙일 수 없음)
  - getExRate를 구체적으로 구현한 신규 클래스 생성 extends PaymentService (슈퍼 클래스)
    - 나머지 기능은 PaymentService에 있는 것을 이용하고 getExRate는 신규 클래스에서 구체적으로 구현 (= 재정의 @Override)
  - 상속을 통한 확장의 한계
    > Java의 창시자인 제임스 고슬링(James Arthur Gosling)이 한 인터뷰에서 "내가 자바를 만들면서 가장 후회하는 일은 상속을 만든 점이다" 라고 말할 정도 이다.조슈야 블로크의 Effective Java에서는 상속을 위한 설계와 문서를 갖추거나, 그럴 수 없다면 상속을 금지하라는 조언을 한다.따라서 추상화가 필요하면 인터페이스로 implements 하거나 객체 지향 설계를 할땐 합성(composition)을 이용하는 것이 추세이다.
    - 결합도가 높아짐
    - 불필요한 기능 상속, 부모 클래스의 결함이 그대로 넘어옴
    - 클래스 폭발 또는 조합의 폭발: 자식 클래스가 부모 클래스의 구현과 강하게 결합되도록 강요하는 상속의 근본적인 한계 때문에 발생

#### 클래스의 분리
- 추상 메소드 제거
- 상속 제거 후 환율 가져오는 방식을 각각 클래스로 분리 (WebApiExRateProvider, SimpleExRateProvider)
- 각 클래스에 메소드 정의 (getWebExRate, getExRate)
- PaymentService: prepare 메소드에서 원하는 방식을 구현한 클래스 생성 후 해당 메소드 호출
- prepare 호출할 때마다 환율 가져오는 클래스가 생성되므로 prepare 메소드 밖에 한 번만 생성해 두는 것이 나음
- private WebApiExRateProvider exRateProvider --> final 꼭 붙여야 하는가 고민 필요

📍 환율 가져오는 방식이 바뀌면 PaymentService 변경해야 함 (좋은 방법은 아님)

#### 인터페이스 도입
- ExRateProvide 인터페이스 도입
- WebApiExRateProvider, SimpleExRateProvider implements ExRateProvider 인터페이스를 구현하도록 코드 변경
- PaymentService에서 객체 타입을 인터페이스인 ExRateProvider로 고정할 수 있음
- 하지만 여전히 WebApiExRateProvider, SimpleExRateProvider 둘 중 하나를 생성해야 함 (결합도 여전히 높음)

📍 환율 가져오는 방식이 바뀌면 PaymentService 변경해야 함 (좋은 방법은 아님)

#### 관계설정 책임의 분리
- 인터페이스를 사용해도 PaymentService는 Provider 클래스에 의존하는 코드가 되버림
- 관계설정 책임이 어디에 있는가 ? ➡️ 환율 가져오는 방식을 결정하는 곳이 어딘지
- Client 객체에 관계설정 책임을 이전

#### 오브젝트 팩토리
- Client 객체도 결국 두 가지 관심사를 가지게 됨: 관계설정 책임, PaymentSerive 이용 ➡️ 분리해야 함
- ObjectFactory 클래스 신규 생성하여 관계설정 책임을 이전
  ``` java
  public class ObjectFactory {
    public PaymentService paymentService() {
      return new PaymentService(exRateProvider());
    }

    public ExRateProvider exRateProvider() {
      return new WebApiExRateProvider();
    }
  }
    ```

#### 원칙과 패턴
- 객체 지향 설계 원칙 (SOLID)
  - 개방 폐쇄 원칙 (Open-Closed Principle: OCP)
    - **클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다.**
    - = 클래스는 클래스의 기능을 확장할 때, 그 클래스의 코드는 변경되면 안된다.
      - ex) PaymentService ➡️ 환율을 가져오는 방식이 확장되어도 코드가 변하지 않음
  - 높은 응집도와 낮은 결합도 (단일 책임 원칙 SRP, 의존 역전 원칙 DIP 연결됨)
    - 응집도가 높다는 것은 하나의 모듈이 하나의 책임 또는 관심사에 집중되어있다는 뜻 ➡️ 변화가 일어날 때 해당 모듈에서 변하는 부분이 크다.
    - 책임과 관심사가 다른 모듈과는 낮은 결합도, 즉 느슨하게 연결된 형태를 유지하는 것이 바람직하다.
  - 제어권 이전 (Inversion of Control)
    - 제어권 이전을 통한 제어 관계 역전 - 프레임워크의 기본 동작 원리
      - ex) PaymentService가 가지고 있던 제어권을 Client에 이전
- 디자인 패턴
  - 전략 패턴 (Strategy Pattern)
    - 자신의 기능 맥락(context)에서, 필요에 따라서 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용할 수 있게 하는 디자인 패턴
    - 실행(런타임) 중에 알고리즘 전략을 선택하여 객체 동작을 실시간으로 바뀌도록 할 수 있게 하는 행위 디자인 패턴
      <img width="1035" height="416" alt="image" src="https://github.com/user-attachments/assets/0337a565-44d0-456d-b4a1-55cecf0dd745" />
    - 주의점
      - 알고리즘이 많아질수록 관리해야할 객체의 수가 늘어남
      - 만일 app 특성이 알고리즘이 많지 않고 자주 변경되지 않는다면, 새로운 클래스와 인터페이스를 만들어 프로그램을 복잡하게 만들 이유가 없음
      - 개발자는 적절한 전략을 선택하기 위해 전략 간의 차이점을 파악하고 있어야 함 (복잡도 ⬆️)
