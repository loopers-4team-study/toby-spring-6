#### 다시 보는 개방 폐쇄 원칙
##### 개방 폐쇄 원칙(OCP)
- 클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다.
- 변화의 특성이 다른 부분을 구분하고 각각 다른 목적과 이유에 의해 다른 시점에 독립적으로 변경될 수 있는 효율적인 구조를 만들어야 한다.
##### 템플릿
- 코드 중에서 **변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분**(=템플릿)을 **자유롭게 변경되는 성질을 가진 부분**(=콜백)으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법
- 템플릿과 콜백(Callback)이 항상 쌍으로 같이 나타나며 **템플릿-콜백 패턴**이라고도 부름

#### WebApiExRateProvider 리팩터링
##### 리팩터링
- 리팩터링 하고자 하는 코드가 외부로 노출하고 있는 **기능을 변경하지 않은 채**로 구조만 개선하는 작업
- 더 이해하기 쉬운 코드로, 변경하기 더 좋은 코드로 만들기 위해 진행
- URL ➡️ URI : java 21부터 `java.net.URL` 클래스의 생성자들이 사용 중단(deprecated)되었으며, `java.net.URI` 클래스를 사용하도록 권장됨
- `throws IOException` 제거
  - IOException은 파일 입출력이나 네트워크 통신(API 호출)처럼 외부 장치와 데이터를 주고받을 때 주로 발생됨
  - API를 호출하지 않고 내부적으로 데이터를 가공하거나 비즈니스 로직만 수행하는 코드라면 IOException 던질 필요가 없음
- `getExRate`메소드 예외 처리
  - 예외적인 상황이라는 것을 시스템 앞단으로 던져줘야 하는데 `catch` 해서 처리할 게 없는 경우 ➡️ **Unchecked Exception**
  ``` java
  String response;
  try {
      HttpURLConnection connection = (HttpURLConnection) uri.toURL().openConnection();
      BufferedReader br = new BufferedReader(new InputStreamReader(connection.getInputStream()));
      response = br.lines().collect(Collectors.joining());
      br.close();
  } catch (IOException e) {
      throw new RuntimeException(e);         // Checked Exception을 Unchecked Exception으로 전환
  }
  ```
- `try-with-resources` 문 처리
  - `try` 블록에 괄호()를 추가하여 파일을 열거나 자원을 할당하는 명령문을 명시하면, 해당 `try` 블록이 끝나자마자 자동으로 파일을 닫거나 할당된 자원을 해제해 줌
  - BufferedReader ➡️ **Autocloseable** 인터페이스를 구현 받고 있음
  ``` java
  try(BufferedReader br = new BufferedReader(new InputStreamReader(connection.getInputStream()))) {
      response = br.lines().collect(Collectors.joining());
  }
  ```
##### Checked Exception vs Unchecked Exception
|구분|Checked Exception|Unchecked Exception|
|------|---|---|
|처리 여부|반드시 처리해야 함 (try-catch 혹은 throws)|명시적인 처리를 강제하지 않음|
|확인 시점|컴파일 시점에 확인|런타임(실행) 시점에 확인|
|발생 원인|프로그램 외적 요인 (파일 없음, 네트워크 단절)|프로그래머의 실수 (0으로 나누기, null 참조)|
|대표 예시|IOException, SQLException|NullPointerException, IndexOutOfBoundsException|

#### 변하는 코드 분리하기 - 메소드 추출
##### WebApiExRateProvider의 구성
1. URI를 준비하고 예외처리를 위한 작업을 하는 코드 ➡️ API로부터 환율 정보를 가져오는 코드의 기본 틀 **(잘 바뀌지 않음)**
2. API를 실행하고, 서버로부터 받은 응답을 가져오는 코드 ➡️ API를 호출하는 기술과 방법이 변경될 수 있음
   ``` java
   HttpURLConnection connection = (HttpURLConnection) uri.toURL().openConnection();

   try(BufferedReader br = new BufferedReader(new InputStreamReader(connection.getInputStream()))) {
       response = br.lines().collect(Collectors.joining());
   }
   ``` 
3. JSON 문자열을 파싱하고 필요한 환율정보를 추출하는 코드 ➡️ API 응답의 JSON 구조에 따라 정보를 추출하는 방식이 변경
   ``` java
   ObjectMapper mapper = new ObjectMapper();
   ExRateData data = mapper.readValue(response, ExRateData.class);
   return data.rates().get("KRW");
   ```
🔴 2, 3 부분을 메소드로 추출

#### 변하지 않는 코드 분리하기 - 메소드 추출
##### 템플릿(Template)
- 어떤 목적을 위해 미리 만들어둔 모양이 있는 틀
- 고정된 틀 안에 바꿀 수 있는 부분을 넣어서 사용하도록 만들어진 오브젝트
- 템플릿 메소드 패턴도 일종의 템플릿
##### 템플릿 메소드 패턴?
- 고정된 틀의 로직을 가진 템플릿 메소드를 슈퍼클래스에 두고, 이 클래스를 상속하여 바뀌는 부분을 서브클래스의 메소드에 두는 구조로 이루어진다.
- 변하는 부분이 여러 개가 되고 점점 복잡해지면 **상속을 이용한 템플릿 메소드 패턴은 한계가 있음**
🔴 메소드로 분리했지만 같은 클래스 안에서 분리했기 때문에 결국 클래스 변경이 필요한 것은 마찬가지임 ➡️ 확장성이 떨어짐

#### ApiExecutor 콜백과 메소드 주입
##### 콜백(Callback)
- 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트
- 파라미터로 전달되지만 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행시키는 것이 목적
- 하나의 메소드를 가진 인터페이스 타입(Single Abstract Method = SAM)의 오브젝트 또는 람다 오브젝트
##### 템플릿/콜백은 전략 패턴의 특별한 케이스
- [전략 패턴](https://github.com/loopers-4team-study/toby-spring-6/blob/main/kodonghee/week-02.md)
- 템플릿은 전략 패턴의 컨텍스트
- 콜백은 전략 패턴의 전략
- 템플릿/콜백은 메소드 하나만 가진 전략 인터페이스를 사용하는 전략 패턴
##### 메소드 주입
- 의존 오브젝트가 메소드 호출 시점에 파라미터로 전달되는 방식
- 의존관계 주입의 한 종류
- 메소드 호출 주입(method call injection)이라고도 한다
##### 템플릿/콜백의 작업 흐름
- 클라이언트가 콜백을 직접 만들어서 템플릿을 호출한다.
<img width="786" height="346" alt="image" src="https://github.com/user-attachments/assets/1d23febd-1eae-42b5-b85b-fd017103903f" />

#### ApiTemplate 분리
##### ApiTemplate
- 환율정보 API로부터 환율을 가져오는 기능을 제공하는 오브젝트
- API 호출과 정보 추출의 기본 틀 제공
- 두 가지 콜백을 이용
- 유사한 여러 오브젝트에서 재사용 가능
- 별도의 클래스로 분리

#### 디폴트 콜백과 템플릿 빈 (재사용 가능한 Template Bean)
##### Singleton Bean
- [스프링의 싱글톤](https://github.com/loopers-4team-study/toby-spring-6/blob/main/kodonghee/week-03.md)
- ApiTemplate이 어플리케이션 레벨에서 공유 가능한 오브젝트로 사용이 된다면 스프링 컨테이너 안의 싱글톤 빈으로 올려 사용하는 것을 고려해 볼 수 있음
- Config 파일에 ApiTemplate을 Spring Bean으로 등록함

#### 스프링이 제공하는 템플릿
##### RestTemplate
- HTTP API 요청을 처리하는 템플릿
  - HTTP Client 라이브러리 확장: ClientHttpRequestFactory
  - Message Body를 변환하는 전략: HttpMessageConverter
- ClientHttpRequestFactory
  - HTTP Client 기술을 사용해서 ClientHttpRequest를 생성하는 전략
    - SimpleClientHttpRequest (HttpURLConnection) - 디폴트
    - JdkClientHttpRequest (HttpClient)
    - NettyClientRequest
    - JettyClientRequest
    - OkHttp3ClientRequest
- doExecute()
  - HTTP API 호출 workflow를 가지고 있는 템플릿 메소드로 두 개의 콜백을 받음
    - RequestCallback
    - ResponseExtractor
    - execute(), getForObject(), postForEntity(), ..등등의 편리한 메소드 제공
##### 스프링의 Template
- JdbcTemplate
- JmsTemplate
- TransactionTemplate
- HibernateTemplate
##### MyBatis
- SqlSessionTemplate

#### 면접 문제
##### ❓ URI, URL, URN의 차이점은 무엇인가요?
<img width="504" height="451" alt="image" src="https://github.com/user-attachments/assets/4128f502-4fee-461c-b06d-2e16eb6308c9" />

- URI (Uniform Resource Identifier)
  - 인터넷에서 자원을 식별하기 위한 문자열, URL과 URN을 포함하는 상위 개념
  - 특정 자원을 식별하기 위한 포괄적인 방법을 제공하며, 자원의 위치나 이름을 나타낼 수 있음
- URL (Uniform Resource Locator)
  - URI의 한 형태로 인터넷상에서 자원의 위치를 나타내는 방식
  - 자원이 어디에 있는지를 설명하는데 사용되며, 자원에 접근하기 위한 프로토콜을 포함
  - 웹페이지의 URL은 해당 페이지가 위치한 서버의 주소와 접근 방법(예: HTTP)을 포함 ex) `https://www.example.com/path/to/resource`
- URN (Uniform Resource Name)
  - URI의 또 다른 형태로 자원의 위치와 상관없이 자원의 이름을 식별하는 방식
  - 자원의 위치가 변하더라도 동일한 식별자를 유지할 수 있게 함
  - 특정 스키마를 따르며, 자원에 대한 영구적인 식별자 제공 ex) `urn:isbn:0451450523`
##### ❓ 개방 폐쇄 원칙(OCP)에 대해 설명하고 이를 코드로 어떻게 구현할 수 있는지 말씀해 주세요.
OCP는 "확장에는 열려 있고 변경에는 닫혀 있어야 한다"는 원칙입니다. 즉, 기존의 코드를 수정하지 않으면서 새로운 기능을 추가할 수 있도록 설계해야 한다는 뜻입니다. 이를 구현하기 위해서는 변하는 부분과 변하지 않는 부분을 구분하는 것이 핵심입니다. 인터페이스를 통해 역할을 정의하고, 구체적인 구현체는 외부에서 주입(DI)받는 방식이나, 이번 학습에서 다룬 템플릿-콜백 패턴을 사용하여 고정된 로직(템플릿)은 유지하되 세부 로직(콜백)만 교체하는 방식으로 구현할 수 있습니다.
##### ❓ 템플릿-콜백 패턴이란 무엇이며, 전략 패턴과는 어떤 차이가 있나요?
템플릿-콜백 패턴은 전략 패턴의 변형으로, 복잡하지만 변하지 않는 로직의 흐름(템플릿) 속에 변하는 부분(콜백)을 삽입하여 사용하는 방식입니다. 주로 익명 내부 클래스나 람다를 사용하여 메소드 레벨에서 의존성을 주입하는 것이 특징입니다. 전략 패턴은 주로 오브젝트 레벨에서 전략을 교체하며 사용하지만, 템플릿-콜백 패턴은 하나의 메소드를 가진 인터페이스(SAM)를 통해 호출 시점에 로직을 전달하므로 훨씬 유연하고 코드의 중복을 획기적으로 줄여줍니다.
##### ❓ 템플릿 메소드 패턴을 사용하다가 템플릿-콜백 패턴으로 리팩터링해야 하는 이유는 무엇인가요?
템플릿 메소드 패턴은 상속을 이용합니다. 하지만 상속은 클래스 간의 결합도가 높고, 만약 변하는 부분(서브클래스에서 구현할 부분)이 많아지면 클래스 개수가 불필요하게 늘어나는 단점이 있습니다. 반면 템플릿-콜백 패턴은 위임(Delegation)을 사용하며, 람다 등을 활용해 클래스 생성 없이도 동적으로 로직을 주입할 수 있어 훨씬 유연하고 확장성이 좋습니다.
##### ❓ Checked Exception과 Unchecked Exception의 차이를 설명하고, Checked Exception을 RuntimeException으로 전환하여 던지는 이유가 무엇인가요?
Checked Exception은 컴파일 시점에 처리를 강제하며 주로 복구 가능한 외부 요인(IO, SQL)에서 발생합니다. 반면 Unchecked Exception은 실행 시점에 확인되며 주로 개발자의 실수로 발생합니다. 실제 개발 시 IOException 같은 Checked Exception을 만났을 때, 해당 레이어에서 처리할 방법이 없다면 무책임하게 throws로 상위 메소드에 전달하기보다는 RuntimeException으로 포장(Wrapping)해서 던지는 것이 좋습니다. 이는 불필요한 예외 전파를 막아 API의 결합도를 낮추고, 서비스 로직을 더 깔끔하게 유지하기 위함입니다.
##### ❓ 스프링에서 제공하는 RestTemplate이나 JdbcTemplate이 템플릿-콜백 패턴의 좋은 예시인 이유는 무엇인가요?
예를 들어 RestTemplate은 HTTP 연결 설정, 리소스 해제, 응답 스트림 읽기 같은 반복적이고 복잡한 작업(템플릿)을 내부적으로 처리해 줍니다. 개발자는 실제 어떤 URL로 요청할지, 받은 데이터를 어떻게 파싱할지 같은 핵심 로직(콜백)에만 집중하면 됩니다. 이처럼 로직의 분리를 통해 개발 효율성을 높이고 실수를 방지하기 때문에 대표적인 예시라고 할 수 있습니다.
##### ❓ Java 21에서 URL 생성자가 deprecated된 이유와 try-with-resources의 장점에 대해 말씀해 주세요.
java.net.URL은 생성 시점에 유효성 검사가 미흡하고 보안상 취약점이 있어, 더 엄격한 구문 검사를 수행하는 java.net.URI 사용이 권장됩니다. try-with-resources는 AutoCloseable을 구현한 자원들을 자동으로 닫아줍니다. 기존의 finally 블록에서 수동으로 close()를 호출할 때 발생할 수 있는 실수나 코드의 복잡함을 해결해 주며, 예외가 발생하더라도 안전하게 자원을 해제해 줍니다.
