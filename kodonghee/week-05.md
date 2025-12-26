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
##### Checked Exception vs Unchecked Exception
|구분|Checked Exception|Unchecked Exception|
|------|---|---|
|처리 여부|반드시 처리해야 함 (try-catch 혹은 throws)|명시적인 처리를 강제하지 않음|
|확인 시점|컴파일 시점에 확인|런타임(실행) 시점에 확인|
|발생 원인|프로그램 외적 요인 (파일 없음, 네트워크 단절)|프로그래머의 실수 (0으로 나누기, null 참조)|
|대표 예시|IOException, SQLException|NullPointerException, IndexOutOfBoundsException|

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
