# 3주차 스터디 정리

## 1. 스프링 IoC 컨테이너

### 1) 제어가 옮겨가는 과정

- **직접 생성 단계**  
  PaymentService가 내부에서 `new WebApiExRateProvider()`를 호출하며 의존성을 직접 관리하던 시점. 제어권이 서비스 내부에 있었다.
- **ObjectFactory 도입**  
  객체 조립 역할을 별도 팩토리(`ObjectFactory`)로 분리하면 `PaymentService`는 인터페이스만 알고, 필요한 구현체는 외부에서 주입받는다. 여기서 IoC(제어의 역전)가 시작된다.
- **스프링 컨테이너 도입**  
  `ObjectFactory`를 `@Configuration` 클래스로 두고 `new AnnotationConfigApplicationContext(ObjectFactory.class)`로 컨테이너를 띄우면, 스프링이 동일한 조립 책임을 자동으로 맡는다.

```java
BeanFactory beanFactory = new AnnotationConfigApplicationContext(ObjectFactory.class);
PaymentService paymentService = beanFactory.getBean(PaymentService.class);
```

### 2) 구성 정보를 전달하는 이유

**핵심 개념:**

- **BeanFactory**는 스프링이 만든 클래스로, 우리가 만든 `PaymentService`, `WebApiExRateProvider`, `SimpleExRateProvider` 같은 클래스들을 알지 못한다
- BeanFactory는 "어떤 빈을 만들고 연결할지" 모른다
- 따라서 **구성 정보(Configuration)**를 담은 클래스가 필요하다

**Bean이란?**

- Bean은 자바 빈(JavaBean)에서 따온 용어
- **Bean = Object (오브젝트)**로 생각해도 무방하다
- 스프링이 관리하는 객체를 빈이라고 부른다

**ObjectFactory의 역할:**

`ObjectFactory`는 **구성 정보를 담는 클래스**입니다:

```java
@Configuration
public class ObjectFactory {
    @Bean
    public PaymentService paymentService() {
        return new PaymentService(cachedExRateProvider());
    }

    @Bean
    ExRateProvider cachedExRateProvider() {
        return new CacheExRateProvider(exRateProvider());
    }

    @Bean
    public ExRateProvider exRateProvider() {
        return new WebApiExRateProvider();
    }
}
```

이 클래스에는 다음 정보가 들어있습니다:

- **어떤 클래스가 런타임에 빈이 되는가**: `PaymentService`, `ExRateProvider` 등
- **어떤 빈이 어떤 빈에 의존하는가**: `PaymentService`가 `ExRateProvider`에 의존
- **어떤 구현체를 사용할 것인가**: `WebApiExRateProvider` vs `SimpleExRateProvider`

**스프링에 구성 정보 전달:**

```java
BeanFactory beanFactory = new AnnotationConfigApplicationContext(ObjectFactory.class);
```

이 코드의 동작 과정:

1. `AnnotationConfigApplicationContext`가 `ObjectFactory.class`를 파라미터로 받음
2. 스프링이 `ObjectFactory` 클래스를 읽어서 `@Configuration`과 `@Bean` 어노테이션을 찾음
3. `@Bean` 메서드들을 분석하여:
    - 어떤 빈을 생성할지
    - 어떤 의존성이 있는지
    - 어떤 구현체를 사용할지
      를 파악
4. 이 정보를 바탕으로 BeanFactory를 구성하고 빈들을 생성하여 레지스트리에 등록

**왜 필요한가?**

- 파라미터 없이 생성하면: `new AnnotationConfigApplicationContext()` → 빈 정의가 없으므로 `getBean()` 호출 시 `NoSuchBeanDefinitionException` 발생
- 구성 정보를 전달해야: 스프링이 우리 애플리케이션의 구조를 이해하고 빈을 생성·관리할 수 있음

### 3) 전략 패턴과 런타임 구성

**전략 패턴(Strategy Pattern) 적용:**

현재 프로젝트에서는 **오브젝트 합성(Object Composition)**을 이용한 전략 패턴을 사용하고 있습니다.

**구조:**

```java
// PaymentService는 ExRateProvider 인터페이스에 의존
public class PaymentService {
    private ExRateProvider exRateProvider;

    public PaymentService(ExRateProvider exRateProvider) {
        this.exRateProvider = exRateProvider;  // 합성을 통한 의존성 주입
    }

    public Payment prepare(...) {
        BigDecimal exRate = exRateProvider.getExRate(currency);  // 전략 사용
        // ...
    }
}

// 두 가지 전략 구현체
public class WebApiExRateProvider implements ExRateProvider {
    // 실제 API를 호출하여 환율 정보를 가져옴
}

public class SimpleExRateProvider implements ExRateProvider {
    // 간단한 하드코딩된 환율 정보 제공
    @Override
    public BigDecimal getExRate(String currency) {
        if (currency.equals("USD")) return BigDecimal.valueOf(1000);
        // ...
    }
}
```

**런타임 구성:**

구성 정보(`ObjectFactory`)에 어떤 구현체를 등록하느냐에 따라 **런타임에 동작이 결정**됩니다.

```java
@Configuration
public class ObjectFactory {
    @Bean
    public PaymentService paymentService() {
        return new PaymentService(exRateProvider());
    }

    // 구성 정보에서 구현체 선택
    @Bean
    public ExRateProvider exRateProvider() {
        return new WebApiExRateProvider();  // 또는 new SimpleExRateProvider()
    }
}
```

**구현체 변경:**

- **WebApiExRateProvider 사용 시**: 실제 API를 호출하여 환율 정보를 가져옴
- **SimpleExRateProvider 사용 시**: 하드코딩된 환율 정보를 사용

구성 정보만 변경하면 코드 수정 없이 **런타임에 다른 전략이 적용**됩니다.

**장점:**

1. **OCP 준수**: `PaymentService` 코드를 수정하지 않고도 전략을 교체 가능
2. **런타임 유연성**: 애플리케이션 실행 시점에 어떤 구현을 사용할지 결정 가능
3. **테스트 용이성**: 테스트 시 `SimpleExRateProvider`나 Mock 객체를 쉽게 주입 가능
4. **확장성**: 새로운 전략 구현체를 추가하기 쉬움

## 2. @Configuration과 @ComponentScan

### @Configuration 어노테이션

**역할:**

- `ObjectFactory`가 **구성 정보(Configuration)**임을 스프링에 알려주는 역할
- 클래스 위에 `@Configuration`을 붙이면 스프링이 "이 클래스는 구성 정보구나!"라고 인식
- 스프링이 이 클래스를 읽어서 빈을 생성하는 지시사항(`@Bean` 메서드)을 찾음

**메타프로그래밍 기법:**

- 어노테이션(Annotation)은 **메타프로그래밍 기법**입니다
- 코드에 붙어있는 메타데이터(정보)로, 런타임에 리플렉션을 통해 읽혀서 처리됩니다
- `@Configuration`, `@Bean`, `@Component`, `@ComponentScan` 모두 어노테이션입니다

**예시:**

```java
@Configuration  // "이 클래스는 구성 정보야!"라고 스프링에 알려줌
public class ObjectFactory {
    @Bean
    public PaymentService paymentService() {
        return new PaymentService(exRateProvider());
    }
}
```

### @ComponentScan 어노테이션

**역할:**

- 지정한 패키지에서 `@Component`가 붙은 클래스를 찾아 빈으로 등록하도록 지시
- 실제 스캔은 `AnnotationConfigApplicationContext`가 수행

**중요: @ComponentScan이 없으면 @Component를 못 찾음!**

```java
@Configuration
@ComponentScan(basePackages = "tobyspring.hellospring")  // 이게 없으면?
public class ObjectFactory {
    // @Bean 메서드들...
}
```

- `@ComponentScan`이 **없으면**: `@Component`가 붙어있어도 스프링이 찾지 못함
- `@ComponentScan`이 **있으면**: 지정한 패키지에서 `@Component`를 찾아서 빈으로 등록

**동작 흐름:**

```
1. new AnnotationConfigApplicationContext(ObjectFactory.class)
   ↓
2. 스프링 컨테이너가 ObjectFactory 클래스를 읽음
   ↓
3. @Configuration 발견 → "이건 구성 정보 클래스구나!"
   ↓
4. @ComponentScan(basePackages = "tobyspring.hellospring") 발견
   ↓
5. "아, tobyspring.hellospring 패키지를 스캔해야겠구나"
   ↓
6. 해당 패키지에서 @Component가 붙은 클래스들을 찾음
   ↓
7. 찾은 클래스들을 빈으로 등록
```

**@ComponentScan이 없는 경우:**

```java
@Configuration  // @ComponentScan 없음!
public class ObjectFactory {
    // @Bean 메서드들...
}

// PaymentService에 @Component가 붙어있어도
@Component  // 이걸 못 찾음!
public class PaymentService {
    // ...
}
```

→ `PaymentService`는 빈으로 등록되지 않음!

## 3. @Component와 자동 스캔

### @Component 어노테이션

- 클래스에 붙여서 스프링 빈으로 등록
- **`@ComponentScan`이 설정되어 있어야만** 스프링이 찾을 수 있음
- `@ComponentScan`이 없으면 `@Component`가 붙어있어도 무시됨

**사용 예시:**

```java
// ObjectFactory에서 @Bean 대신 @Component 사용 가능
@Configuration
@ComponentScan(basePackages = "tobyspring.hellospring")
public class ObjectFactory {
    // @Bean 메서드들을 주석 처리하고
    // 대신 각 클래스에 @Component를 붙임
}

@Component  // 이렇게 붙이면 자동으로 빈으로 등록됨
public class PaymentService {
    // ...
}

@Component  // WebApiExRateProvider와 SimpleExRateProvider 중 선택
public class WebApiExRateProvider implements ExRateProvider {
    // ...
}

// 또는

@Component
public class SimpleExRateProvider implements ExRateProvider {
    // ...
}
```

**어떤 구현체를 사용할지 결정:**

- `WebApiExRateProvider`에 `@Component`를 붙이면 → WebApiExRateProvider가 빈으로 등록
- `SimpleExRateProvider`에 `@Component`를 붙이면 → SimpleExRateProvider가 빈으로 등록
- 둘 다 붙이면? → 둘 다 빈으로 등록되지만, `ExRateProvider` 타입으로 조회 시 어떤 것이 주입될지 모호함 (이 경우 `@Primary` 또는 `@Qualifier` 사용 필요)

### @Bean vs @Component

**@Bean 방식:**

```java
@Configuration
public class ObjectFactory {
    @Bean
    public PaymentService paymentService() {
        return new PaymentService(exRateProvider());
    }

    @Bean
    public ExRateProvider exRateProvider() {
        return new WebApiExRateProvider();
    }
}
```

- 수동으로 빈을 정의
- 복잡한 초기화 로직이 필요할 때 유용

**@Component 방식:**

```java
@Component
public class PaymentService {
    private ExRateProvider exRateProvider;

    @Autowired
    public PaymentService(ExRateProvider exRateProvider) {
        this.exRateProvider = exRateProvider;
    }
}

@Component
public class WebApiExRateProvider implements ExRateProvider {
    // ...
}
```

- 자동으로 빈으로 등록
- 의존성은 `@Autowired`로 자동 주입
- 더 간결하고 편리

## 4. 싱글톤 레지스트리

### 싱글톤 패턴이란?

**싱글톤(Singleton)**은 애플리케이션 전체에서 **단 하나의 인스턴스만 존재**하도록 보장하는 디자인 패턴입니다.

**전통적인 싱글톤 패턴 구현:**

```java
public class PaymentService {
    private static PaymentService instance;  // static 필드

    private PaymentService() {}  // private 생성자로 외부 생성 방지

    public static PaymentService getInstance() {  // static 메서드
        if (instance == null) {
            instance = new PaymentService();
        }
        return instance;
    }
}
```

### 싱글톤 패턴의 단점

GoF 디자인 패턴의 공동 저자인 **에리히 감마(Erich Gamma)**는 싱글톤 패턴에 대해 우려를 표명하고, 디자인 패턴 목록에서 제외하고 싶다는 의견을 밝힌 바 있습니다.

**주요 단점:**

1. **테스트 어려움**

    - static 필드로 인한 전역 상태
    - 테스트마다 상태가 공유되어 테스트 격리 어려움
    - Mock 객체로 대체하기 어려움
    - 테스트 순서에 의존적

2. **의존성 숨김**

   ```java
   public class OrderService {
       public void processOrder() {
           PaymentService service = PaymentService.getInstance();  // 의존성이 숨겨짐
           // ...
       }
   }
   ```

    - 생성자나 파라미터로 의존성이 드러나지 않음
    - 코드만 봐서는 어떤 의존성이 있는지 파악 어려움
    - 의존성 역전 원칙(DIP) 위반

3. **유연성 저하**

    - 항상 하나의 인스턴스만 존재
    - 다중 인스턴스가 필요한 경우 대응 어려움
    - 상속과 다형성 활용 제한

4. **멀티스레드 환경의 복잡성**
    - 동시성 제어를 위한 추가 코드 필요
    - 잘못 구현 시 성능 저하나 버그 가능

### 스프링 싱글톤 레지스트리

스프링은 싱글톤 패턴의 단점을 해결하면서도 싱글톤의 이점을 활용할 수 있는 **싱글톤 레지스트리**를 제공합니다.

**핵심 개념:**

- 스프링 컨테이너가 빈을 **하나만 생성**해서 **컨테이너 내부 레지스트리에 저장**
- 같은 빈을 요청하면 **저장된 인스턴스를 꺼내서 재사용**
- 개발자는 싱글톤 패턴을 직접 구현하지 않아도 됨
- 일반 클래스(POJO)처럼 작성해도 스프링이 싱글톤으로 관리

**동작 방식:**

```java
@Bean
public PaymentService paymentService() {
    return new PaymentService(exRateProvider());  // 첫 번째 호출
}

@Bean
public OrderService orderService() {
    return new OrderService(exRateProvider());    // 두 번째 호출
}

@Bean
public ExRateProvider exRateProvider() {
    return new WebApiExRateProvider();
}
```

**동작 과정:**

1. **컨테이너 초기화 시:**

   ```
   new AnnotationConfigApplicationContext(ObjectFactory.class)
   ↓
   스프링이 @Bean 메서드들을 찾아서 실행
   ↓
   exRateProvider() 실행 → WebApiExRateProvider 인스턴스 생성
   ↓
   생성된 인스턴스를 컨테이너 내부 레지스트리에 저장
   ┌─────────────────────────────────────┐
   │  스프링 컨테이너 (싱글톤 레지스트리)  │
   ├─────────────────────────────────────┤
   │  ExRateProvider 인스턴스 → 저장     │
   └─────────────────────────────────────┘
   ↓
   paymentService() 실행 → exRateProvider() 호출
   ↓
   레지스트리에서 찾아서 반환 (새로 생성하지 않음!)
   ```

2. **이후 getBean() 호출 시:**

   ```java
   PaymentService service1 = beanFactory.getBean(PaymentService.class);
   PaymentService service2 = beanFactory.getBean(PaymentService.class);

   service1 == service2  // true (같은 인스턴스 재사용)
   ```

    - 컨테이너가 레지스트리를 확인
    - 이미 저장된 인스턴스를 찾아서 반환
    - 새로 생성하지 않고 기존 인스턴스 재사용

**결과:**

- 코드상으로는 `exRateProvider()`를 두 번 호출하지만
- 실제로는 한 번만 생성되고, 두 번째는 기존 인스턴스를 재사용
- `paymentService.exRateProvider == orderService.exRateProvider` → `true`

### 스프링 싱글톤 레지스트리의 장점

1. **POJO 유지**

    - 일반 클래스처럼 작성 가능 (static 필드, private 생성자 불필요)
    - 테스트 시 자유롭게 인스턴스 생성 가능

2. **명시적 의존성**

   ```java
   public class OrderService {
       private PaymentService paymentService;

       // 의존성이 명확하게 드러남
       public OrderService(PaymentService paymentService) {
           this.paymentService = paymentService;
       }
   }
   ```

3. **테스트 용이성**

   ```java
   @Test
   void testPaymentService() {
       ExRateProvider mockProvider = mock(ExRateProvider.class);
       PaymentService service = new PaymentService(mockProvider);  // 자유롭게 생성 가능
       // ...
   }
   ```

4. **유연성**

    - 필요 시 프로토타입 스코프로 변경 가능
    - 스프링이 관리하므로 필요에 따라 다른 인스턴스 제공 가능

5. **메모리 효율성**: 인스턴스를 하나만 유지
6. **성능**: 생성 비용이 한 번만 발생
7. **일관성**: 같은 인스턴스를 공유하므로 상태 관리가 일관됨

### 정리

- **싱글톤 패턴**: 직접 구현해야 하고, 테스트 어렵고, 의존성이 숨겨지는 단점
- **스프링 싱글톤 레지스트리**: 싱글톤의 이점(메모리 효율, 성능)을 유지하면서 단점(테스트 어려움, 의존성 숨김)을 해결
- **컨테이너에 저장하고 재사용**: 스프링 컨테이너가 빈을 레지스트리에 저장하고, 같은 빈을 요청하면 저장된 인스턴스를 꺼내서 재사용

## 5. 데코레이터 패턴

### 문제 상황

- `WebApiExRateProvider`가 매번 API를 호출하는 것은 비효율적
- 캐시 기능을 추가하고 싶지만, `WebApiExRateProvider`를 직접 수정하는 것은 OCP 위반

### 해결: 데코레이터 패턴

```java
public class CacheExRateProvider implements ExRateProvider {
    private final ExRateProvider target;

    public CacheExRateProvider(ExRateProvider target) {
        this.target = target;
    }

    @Override
    public BigDecimal getExRate(String currency) throws IOException {
        // 캐시 확인
        // 캐시 미스 시 target.getExRate() 호출 후 캐시 저장
        // 캐시 히트 시 캐시된 값 반환
        return target.getExRate(currency);
    }
}
```

**구조:**

```
PaymentService
    ↓
CacheExRateProvider (데코레이터)
    ↓
WebApiExRateProvider (실제 구현체)
```

### 장점

1. **OCP 준수**: `WebApiExRateProvider`를 수정하지 않고 기능 추가
2. **단일 책임 원칙**: 각 클래스가 하나의 책임만 가짐
3. **유연성**: 캐시를 켜고 끌 수 있음
4. **확장성**: 여러 데코레이터를 조합 가능
   ```java
   new CacheExRateProvider(
       new LoggingExRateProvider(
           new WebApiExRateProvider()
       )
   )
   ```

### ObjectFactory에서의 조합

```java
@Bean
public PaymentService paymentService() {
    return new PaymentService(cachedExRateProvider());
}

@Bean
ExRateProvider cachedExRateProvider() {
    return new CacheExRateProvider(exRateProvider());
}

@Bean
public ExRateProvider exRateProvider() {
    return new WebApiExRateProvider();
}
```

## 6. Policy Layer vs Mechanism Layer

### 레이어 구분

**Policy Layer (정책 계층)**

- 비즈니스 정책과 흐름을 결정
- 예: `PaymentService`
- "무엇을 할 것인가"를 정의

**Mechanism Layer (메커니즘 계층)**

- 실제 동작을 수행하는 구현
- 예: `ExRateProvider`, `WebApiExRateProvider`, `SimpleExRateProvider`
- "어떻게 할 것인가"를 구현

### 패키지 구조

```
payment/          (Policy Layer)
  ├── PaymentService
  └── Payment

exrate/           (Mechanism Layer)
  ├── ExRateProvider (인터페이스)
  ├── WebApiExRateProvider
  ├── SimpleExRateProvider
  └── CacheExRateProvider
```

### 관계

- **Policy Layer가 상위 모듈**, **Mechanism Layer가 하위 모듈**
- Policy Layer가 Mechanism Layer를 사용하지만, 인터페이스를 통해 의존

## 7. DIP (Dependency Inversion Principle)

### DIP의 핵심

- **상위 수준 모듈이 하위 수준 모듈의 구현에 의존하지 않고, 추상화(인터페이스)에 의존해야 한다**

### 현재 구조의 DIP 준수 여부

**현재 구조:**

- `ExRateProvider` 인터페이스: `exrate` 패키지 (Mechanism Layer)
- `PaymentService`: `payment` 패키지 (Policy Layer)
- `PaymentService`가 `exrate.ExRateProvider`를 import해서 사용

**두 가지 관점:**

1. **현재 구조도 DIP 준수 (느슨한 관점)**

    - `PaymentService`는 구체 클래스가 아닌 인터페이스(`ExRateProvider`)에 의존
    - 구현체를 직접 new 하지 않고 외부에서 주입받음
    - ✅ 인터페이스에 의존하므로 DIP 준수

2. **더 엄격한 DIP를 위해서는 인터페이스를 Policy Layer로 이동 (엄격한 관점)**
    - Policy Layer가 필요한 추상화를 정의해야 함
    - 의존성 방향: `payment` → `ExRateProvider` (자신의 패키지), `exrate` → `ExRateProvider` (구현)
    - Policy Layer가 Mechanism Layer 패키지를 전혀 알 필요가 없음
    - 패키지 의존성 그래프가 더 명확해짐

**엄격한 DIP 구조:**

```
payment/
  ├── PaymentService
  └── ExRateProvider (인터페이스) ← Policy Layer가 정의

exrate/
  ├── WebApiExRateProvider implements ExRateProvider
  ├── SimpleExRateProvider implements ExRateProvider
  └── CacheExRateProvider implements ExRateProvider
```

**의존성 방향:**

- `PaymentService` → `ExRateProvider` (같은 패키지)
- `WebApiExRateProvider` → `ExRateProvider` (다른 패키지, 구현)

이렇게 하면 Policy Layer가 Mechanism Layer를 전혀 알 필요가 없어집니다.

### 결론

- 현재 구조도 동작하고 인터페이스에 의존하므로 기본적인 DIP는 준수
- 더 엄격한 DIP를 원한다면 인터페이스를 Policy Layer로 이동하는 것이 좋음
- 패키지 의존성 측면에서 완전한 DIP를 위해서는 Policy Layer가 추상화를 정의해야 함

## 핵심 정리

1. **스프링 IoC 컨테이너**: BeanFactory와 ApplicationContext를 통해 빈을 관리
2. **@Configuration과 @ComponentScan**: 설정 클래스와 자동 스캔을 통한 빈 등록
3. **싱글톤 레지스트리**: 빈을 하나만 생성해서 재사용하여 효율성 향상
4. **데코레이터 패턴**: 기존 코드 수정 없이 기능을 추가하는 확장 방법
5. **레이어 분리**: Policy Layer와 Mechanism Layer로 관심사 분리
6. **DIP**: 인터페이스를 통한 의존성 역전으로 유연한 구조 구성
