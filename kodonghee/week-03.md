#### 스프링 컨테이너와 의존관계 주입 (Dependency Injection)
- 스프링 컨테이너 ➡️ **IOC/DI**를 제공하는 컨테이너
  - IOC 🟰 Inversion Of Control
  - DI 🟰 Dependency Injection
<img width="901" height="293" alt="image" src="https://github.com/user-attachments/assets/3fe90cd6-c196-4136-b338-2c8415670279" />

- IOC: PaymentService ➡️ ObjectFactory
- ObjectFactory 자리에 **BeanFactory** 삽입
  - 자바 빈: 자바의 컴포넌트 오브젝트 모델에 일반적으로 붙이는 용어, 애플리케이션의 기능을 담당하고 제공하는 핵심 클래스의 오브젝트
    - 빈(Bean) 🟰 PaymentService, WebApiExRateProvider 🟰 애플리케이션이 시작될 때 오브젝트로 만들어서 사용되어지는 클래스
  - BeanFactory는 Spring이 제공함 (직접 만들지 않음) ➡️ PaymentService나 WebApiExRateProvider에 대한 정보를 알지 못함
  - ObjectFactory 필요 ➡️ 어떤 클래스의 오브젝트를 만들고 어떻게 연결할 것인가에 대한 정보를 가진 오브젝트, BeanFactory가 ObjectFactory를 참고
    - 구성정보(Configuration): 빈 클래스, 의존관계(런타임)
    
