#### 스프링 컨테이너와 의존관계 주입 (Dependency Injection)
- 스프링 컨테이너 ➡️ **IOC/DI**를 제공하는 컨테이너
  - IOC 🟰 Inversion Of Control
  - DI 🟰 Dependency Injection 🟰 외부에서 제3의 오브젝트가 실제 애플리케이션이 동작하는 데 사용되는 두 개의 오브젝트를 생성하고 그 의존관계를 맺어주는 것
<img width="901" height="293" alt="image" src="https://github.com/user-attachments/assets/3fe90cd6-c196-4136-b338-2c8415670279" />

- IOC: PaymentService ➡️ ObjectFactory
- ObjectFactory 자리에 **BeanFactory** 삽입
  - 자바 빈: 자바의 컴포넌트 오브젝트 모델에 일반적으로 붙이는 용어, 애플리케이션의 기능을 담당하고 제공하는 핵심 클래스의 오브젝트
    - 빈(Bean) 🟰 PaymentService, WebApiExRateProvider 🟰 애플리케이션이 시작될 때 오브젝트로 만들어서 사용되어지는 클래스
  - BeanFactory는 Spring이 제공함 (직접 만들지 않음) ➡️ PaymentService나 WebApiExRateProvider에 대한 정보를 알지 못함
  - ObjectFactory 필요 ➡️ 어떤 클래스의 오브젝트를 만들고 어떻게 연결할 것인가에 대한 정보를 가진 오브젝트, BeanFactory가 ObjectFactory를 참고
    - 구성정보(Configuration): 빈 클래스 `@Bean`, 의존관계(런타임) `@Configuration`
    ``` java
    public static void main(String[] args) throws IOException {
        BeanFactory beanFactory = new AnnotationConfigApplicationContext(ObjectFactory.class);
        PaymentService paymentService = beanFactory.getBean(PaymentService.class);

        Payment payment = paymentService.prepare(100L, "USD", BigDecimal.valueOf(50.7));
        System.out.println(payment);
    }
    ```
- ObjectFactory만 사용했을 때 vs BeanFactory로 대체했을 때 차이❓
  - ObjectFactory만 사용했을 때는 오브젝트를 생성하는 메소드를 바로 호출하여 cliet가 사용
  - BeanFactory로 대체할 경우 언제 오브젝트를 만들었는지는 모르겠지만 BeanFactory가 참고하는 구성정보를 이용해서 오브젝트를 반환하라고 시킬 수 있음
  - BeanFactory가 처음 만들어질 때 이미 Bean 오브젝트들이 만들어지고 BeanFactory가 이것들을 내부에 가지고 있음 ➡️ 필요로 할 때 제공
  - ⭐️ BeanFactory 장점 ⭐️: 개발자가 객체 생성 시점을 신경 쓰지 않아도됨 🟰 의존 객체가 많아질수록 유지보수가 훨씬 쉬워짐
  - BeanFactory 🟰 스프링 IoC/DI 컨테이너

#### 구성정보를 가져오는 다른 방법
- `@Component`: Bean 오브젝트로 만들 대상 클래스에 직접 붙여줌 ➡️ ObjectFactory에서 `@Bean` 메소드 만들지 않아도 됨
- `@ComponentScan`: 직접 Bean을 등록하지 않고 스프링이 애노테이션을 참고해서 **자동으로** 빈을 찾아 등록하게 하고 싶을 때 사용
  <img width="519" height="165" alt="image" src="https://github.com/user-attachments/assets/24d0415e-a7aa-4733-8b35-c877cf5561a1" />

<img width="684" height="498" alt="image" src="https://github.com/user-attachments/assets/e081afea-ff2b-45e5-a1ed-766f28a7d389" />


#### 싱글톤 레지스트리 (Singleton Registry)
- 스프링 컨테이너가 가진 독특한 특징
##### 싱글톤 패턴
- 애플리케이션 전반에 걸쳐서 **딱 하나**의 오브젝트만 존재해야 하는 경우
- 보통 `public static method`를 통해 최초로 오브젝트가 만들어진 이후에는 반복적으로 똑같은 오브젝트를 리턴하는 방식으로 설계
- 어디에서나 해당 오브젝트를 쉽게 가져다 쓸 수 있음
- 단점
  - 오브젝트 간의 의존관계가 분명하지 않음
  - 마구잡이로 오브젝트들을 이용하게 됨 ➡️ 전역적인 접근이 일어남 (global)
  - static 메소드라 테스트하는 환경을 만들기 어려움
- 🟣 싱글톤 패턴이 아닌 **스프링**을 사용하면 된다 !
##### 스프링의 싱글톤
- 스프링은 많은 경우에 그 안에서 생성하고 구성하는 오브젝트들을 **싱글톤**으로 만듬 🟰 딱 한 개만 만듬
- 스프링은 대부분 서버 애플리케이션을 만들기 때문에 사용자 요청마다 매번 새로운 오브젝트를 만들면 굉장히 메모리를 많이 먹고 비용 낭비가 심할 수 있음
```java
PaymentService paymentService = beanFactory.getBean(PaymentService.class);
PaymentService paymentService2 = beanFactory.getBean(PaymentService.class);
```
- `paymentService`와 `paymentService2`는 동일한 오브젝트
- 스프링 컨테이너가 PaymentService 타입의 오브젝트를 딱 한 개만 만들어서 가지고 있다는 의미
```java
ObjectFactory objectFactory = beanFactory.getBean(ObjectFactory.class);
PaymentService paymentService1 = objectFactory.PaymentService();
PaymentService paymentService2 = objectFactory.PaymentService();
```
- `paymentService`와 `paymentService2`는 동일한 오브젝트
- objectFactory에서 생성자가 두 번 호출되었음에도 두 오브젝트는 동일함 ➡️ 스프링에서 `@Configuration` 클래스 안의 메소드 호출을 여러 번해도 특별한 다른 지시가 없다면 딱 하나의 오브젝트만 생성되도록 만듬

#### DI와 디자인 패턴 (1)
##### 의존성 주입 (Dependency Injection)
##### 디자인 패턴
- 패턴의 목적
- 패턴의 Scope
  - Class 패턴: 상속
  - Object 패턴: 합성
    - 스프링의 의존관계 주입(DI)을 사용
- ❓환율 정보가 필요할 때 매번 Web API를 호출해야 할까?
  - 환율 정보가 필요한 기능 증가/응답 시간/환율 변동 주기
  - 💡 환율 정보 캐시(Cache)의 도입 ➡️ 디자인 패턴을 잘 응용하면 WebApiExRateProvider 코드를 수정하지 않고도 캐시 도입 가능 (**데코레이터(Decorator)**)
- 데코레이터(Decorator) 디자인 패턴
  - 코드를 건드리지 않고 오브젝트에 부가적인 기능/책임을 동적으로 부여
<img width="921" height="349" alt="image" src="https://github.com/user-attachments/assets/af288dbb-8ad9-42ba-8f3d-7e89a00c82b9" />

#### DI와 디자인 패턴 (2)

