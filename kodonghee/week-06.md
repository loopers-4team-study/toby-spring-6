#### 예외를 다루는 방법
##### 예외
- 예외는 정상적인 프로그램 흐름을 방해하는 사건
- 예외적인 상황에서만 사용
- 많은 경우 예외는 프로그램 오류, 버그 때문에 발생 ➡️ 테스트 실패
##### 예외가 발생하면
- ❓ 예외 상황을 복구해서 정상적인 흐름으로 전환할 수 있는가?
  1. 재시도 (pg 연동 실패 같은 경우, 네트워크 문제 등)
  2. 대안 (환율정보 가져오는 다른 방식 마련, 캐시 등)
- ❓ 버그인가?
  1. 예외가 발생한 코드의 버그인가?
  2. 클라이언트의 버그인가?
- ❓ 제어할 수 없는 예외상황인가?
##### 예외를 잘못 다루는 코드
- 예외를 무시하는 코드
  - catch 문이 비어있거나 catch문에서 print만 하는 경우
  ``` java
  catch(SQLException e) {
  }

  catch(SQLException e) {
    e.printStackTrace();
  }
  ```
  - 복구를 하거나 예외를 밖으로 던져야 함 ➡️ 최종적으로 예외가 던져진 것을 볼 수 있도록 해야 함
- 무의미하고 무책임한 throws
  - 예외가 발생했을 때 이를 어떻게 처리할지 고민하지 않고, 단순히 상위 메소드로 책임을 떠넘기는 행위임
  - 결국 최상위 컨트롤러나 메인 메소드까지 예외가 전달되어 어디서 왜 문제가 생겼는지 파악하기 힘들어짐
  - 해결책
    1. Checked Exception 최소화
    2. 커스텀 예외 활용
    3. Spring `@RestControllerAdvice` 활용
##### 예외의 종류
- Error
  - 시스템에 비정상적인 상황이 발생
  - OutOfMemoryError
  - ThreadDeath
  - 컨트롤할 수 없는 부분이므로 catch하면 안됨
- Exception(checked)
  - catch나 throws를 강요
  - 초기 라이브러리의 잘못된 예외 설계/사용 ➡️ **요즘엔 잘 쓰지 않는 추세**
  - 복구할 수 없다면 RuntimeException이나 적절한 추상화 레벨의 예외로 전환해서 던질 것
- RuntimeException(unchecked)
##### 예외의 추상화와 전환
- 사용 기술에 따라 같은 문제에 대해 다른 종류의 예외 발생
- 적절한 예외 추상화와 예외 번역이 필요

#### JPA를 이용한 Order 저장
<img width="1015" height="509" alt="image" src="https://github.com/user-attachments/assets/21e6b03a-bdad-4908-84ae-2f4e5294a49e" />

- DataSource, OrderRepository, EntityManagerFactory: 싱글톤, 스프링의 빈으로 등록됨
- Order, EntityManager: 스프링의 빈으로 등록되지 않음
- DataSource, EntityManagerFactory ➡️ **인프라 스트럭쳐 빈**
  - JPA라는 기술이 애플리케이션 안에서 잘 동작할 수 있도록 기반을 만들어 줌
  - 스프링부트에서는 자동으로 만들어짐
 
#### DataAccessException과 예외 추상화
##### JDBC SQLException
- JDBC를 기반으로 하는 모든 기술에서 발생하는 예외
- JDBC, MyBatis, JPA, ...
- DB의 에러코드에 의존하거나, 데이터 기술에 의존적인 예외처리 코드
##### DataAccessException (스프링)
- DB의 에러코드와 데이터 액세스 기술에 독립적인 예외 구조
- 적절한 예외 번역(exception translation) 도구를 제공

🟣 **체계적인 예외 구조를 만들고 적절한 예외 처리 방법을 사용하고 있는지 살펴보자 !**
