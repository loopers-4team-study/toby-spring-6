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

#### 싱글톤 레지스트리


    
