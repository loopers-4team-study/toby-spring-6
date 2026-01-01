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
