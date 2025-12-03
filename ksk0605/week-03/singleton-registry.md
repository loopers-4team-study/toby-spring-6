# 싱글톤 레지스트리

스프링의 **싱글톤 레지스트리(Singleton Registry)**는 스프링 IoC 컨테이너의 핵심 원리 중 하나로,
“**애플리케이션 전역에서 단 하나의 객체 인스턴스를 생성·관리하는 메커니즘**”이다.

그리고 중요한 점은  
➡ **스프링은 ‘디자인 패턴의 싱글톤’을 사용하지 않는다.**  
➡ **대신 ‘컨테이너 기반의 싱글톤 레지스트리’를 사용한다.**  

---

# 1. 스프링의 싱글톤 레지스트리란?

일반적인 싱글톤 패턴(singleton pattern)처럼 new 생성자를 막거나 static instance를 관리하지 않는다.
**지정한 scope(기본값 = singleton)** 에 따라 스프링 컨테이너가 객체를 하나 만들고,
이 객체를 **ConcurrentHashMap** 형태의 캐시에 저장한 뒤, 필요한 모든 곳에 동일한 인스턴스를 반환한다.

```text
개발자가 static 싱글톤을 구현하는 것이 아니라  
스프링 컨테이너가 대신 “객체 하나만 생성해서 공유”하는 구조
```

---

# 2. 스프링 싱글톤 레지스트리의 핵심 원리 (동작 방식)

## 2.1 BeanDefinition 읽기

컨테이너가 시작될 때 `@Bean`, `@Component` 등에 대한 **BeanDefinition**을 만든다.

## 2.2 createBean() 과정에서 싱글톤 캐시 확인

스프링은 bean을 요청 받으면 다음과 같이 처리한다:

```java
Object getBean(String beanName) {
    Object sharedInstance = getSingleton(beanName); // 1. 먼저 싱글톤 캐시 확인
    if (sharedInstance != null) {
        return sharedInstance;                      // 이미 있으면 그대로 반환
    }
    return createAndRegisterBean(beanName);         // 없으면 생성
}
```

## 2.3 getSingleton() 캐시 구조

스프링은 다음과 같은 3단계 캐시를 가진다.

| 캐시                    | 설명                      |
| --------------------- | ----------------------- |
| singletonObjects      | 완성된 싱글톤 저장소             |
| earlySingletonObjects | 순환참조 방지를 위한 “조기 노출” 저장소 |
| singletonFactories    | 객체 팩토리 저장소              |

가장 중요한 것은 **singletonObjects**, 즉 완성된 싱글톤이 들어가는 곳이다.

```java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

스프링 전체 싱글톤의 핵심이다.

---

# 3. 스프링 실제 소스 코드 분석

📌 **Spring Framework `DefaultSingletonBeanRegistry`** 내부 코드를 가져와 설명합니다.
*출처: [https://github.com/spring-projects/spring-framework](https://github.com/spring-projects/spring-framework)*

---

## 3.1 getSingleton() 구현

```java
public Object getSingleton(String beanName) {
    return this.singletonObjects.get(beanName);
}
```

딱 이 한 줄이다.
**“이미 만들어져 있는 싱글톤인지 먼저 확인”** 하는 메서드.

---

## 3.2 싱글톤 생성 및 등록

컨테이너가 bean을 만들 때는 다음 과정을 거친다.

```java
protected Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        Object singletonObject = this.singletonObjects.get(beanName);

        if (singletonObject == null) {
            singletonObject = singletonFactory.getObject(); // 1. 진짜 bean 생성

            this.singletonObjects.put(beanName, singletonObject); // 2. 캐시에 저장
        }
        return singletonObject;
    }
}
```

### 요약하면:

1. **singletonObjects 에 beanName 이 있는지 확인**
2. 없으면 bean 생성 (`singletonFactory.getObject()`)
3. **ConcurrentHashMap 에 등록**
4. 동일 beanName 요청 시 언제나 동일 인스턴스 반환

✔ 이것이 “스프링 싱글톤 레지스트리”의 핵심이다.

---

# 4. 중요한 포인트: 스프링은 ‘진짜 싱글톤 패턴’을 쓰지 않는다

“싱글톤 패턴”의 단점:

* private 생성자로 확장 어려움
* 테스트 어려움
* 전역 상태 위험
* Lazy 초기화/해제 유연성 부족

스프링의 “컨테이너 기반 싱글톤”의 장점:

* 생성 주기 관리 용이
* DI 가능 (true singleton 에서는 불가능)
* 테스트에서 손쉽게 mock/override 가능
* Lazy 또는 Eager 전략 조절 가능
* 스코프 변경 가능 (prototype, request, session 등)

---

# 5. 스프링이 싱글톤 레지스트리에서 활용하는 디자인 패턴

### ✔ 1) **팩토리 메서드 패턴**

* `@Bean`, `FactoryBean`, ObjectFactory 등에서 객체를 생성하는 역할

### ✔ 2) **템플릿 메서드 패턴**

* bean 생성 전에 후처리기 처리 → createBean 과정이 hook 메서드처럼 확장됨

### ✔ 3) **프록시 패턴**

* AOP, 트랜잭션 관리 때 싱글톤 bean을 감싸서 기능 확장

### ✔ 4) **전략 패턴**

* BeanName 생성 전략
* Scope 전략
* Autowire 전략 등

---

# 6. 스프링이 IoC/DI 와 함께 싱글톤을 왜 사용하는가?

싱글톤 레지스트리를 사용하는 이유는 간단하다.

1. 대부분의 스프링 빈은 **상태가 없는(stateless)** 서비스 빈
2. 이런 서비스 객체를 계속 생성할 이유가 없음
3. 요청마다 새로 만들어야 하는 HTTP 요청/세션 스코프 외에는 대부분 싱글톤 사용
4. 메모리·성능 최적화 크게 향상

**즉, “스프링 애플리케이션은 본질적으로 싱글톤 기반 구조”다.**

---

# 7. 싱글톤 레지스트리는 어디까지 싱글톤인가?

* JVM 내부에서만 싱글톤
* 여러 서버가 띄워지면 서버별 컨테이너마다 각각 별도의 싱글톤 생성
* 즉 “어플리케이션 인스턴스 단위 싱글톤”



# 스프링 전체 초기화 흐름

스프링 전체 초기화 흐름 속에서 싱글톤 레지스트리가 ‘언제’, ‘어떻게’ 작동하는지**
**DispatcherServlet → ApplicationContext → BeanFactory → Singleton Registry** 순서로 아주 깊게 설명드리겠습니다.

> **“스프링 빈은 생성 시점부터 싱글톤 레지스트리에 의해 관리된다.”**  
> 스프링 MVC 앱에서 첫 요청이 들어오기 훨씬 전에 대부분의 싱글톤 빈은 이미 만들어져 있다.

---

# 전체 큰 흐름

아래는 Spring MVC 애플리케이션을 기준으로 **싱글톤 레지스트리가 실제로 개입하는 시점 전체 흐름**이다:

```
Tomcat 기동
  └─ DispatcherServlet 생성
        └─ WebApplicationContext 생성
             └─ BeanFactory 초기화
                 └─ BeanDefinition 로딩
                     └─ (중요) 모든 singleton bean 미리 생성(createBean)
                           └─ singletonObjects 캐시에 저장
                                 └─ 이후 getBean() 호출은 모두 같은 인스턴스 반환
```

이제 단계별로 뜯어보자.

---

# 1단계. Tomcat이 DispatcherServlet 을 생성한다

Spring MVC에서는 `DispatcherServlet`이 가장 최초로 생성되는 핵심 컴포넌트다.

**Tomcat → ServletContainer → DispatcherServlet 생성자 실행**

DispatcherServlet 생성자 안에는 아래 코드가 있다:

```java
public DispatcherServlet() {
    super();
    setDispatchOptionsRequest(true);
}
```

아무것도 안 하는 것 같지만
**이 시점에서 아직 context는 없다.**

---

# 2단계. DispatcherServlet.init() 실행 → ApplicationContext 생성

서블릿 초기화 과정에서 실행되는 `init()` 내부:

```java
this.webApplicationContext = initWebApplicationContext();
```

### 내부에서 하는 일

1. WebApplicationContext 생성
2. 부모 컨텍스트(ApplicationContext) 연결
3. refresh() 호출하여 BeanFactory 초기화

핵심은 **refresh() 호출**이다.

---

# 3단계. ApplicationContext.refresh() 호출

**싱글톤 레지스트리가 작동하는 핵심 지점**

아래는 축약된 실제 스프링 코드이다.

```java
public void refresh() {
    // 1. BeanFactory 생성
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // 2. BeanFactory 후처리기 실행
    invokeBeanFactoryPostProcessors(beanFactory);

    // 3. BeanPostProcessor 등록
    registerBeanPostProcessors(beanFactory);

    // 4. (중요) 싱글톤 preload — 모든 싱글톤 빈 생성
    finishBeanFactoryInitialization(beanFactory);
}
```

### **finishBeanFactoryInitialization() → 프리로딩(Pre-instantiation)**

스프링은 기본적으로 싱글톤 bean을 모두 **미리 생성**한다.
이 과정에서 싱글톤 레지스트리가 등장한다.

---

# 4단계. DefaultListableBeanFactory → createBean() 호출

(싱글톤 레지스트리가 실제로 동작하는 시점)

`finishBeanFactoryInitialization()` 내부 핵심 부분:

```java
beanFactory.preInstantiateSingletons();
```

이 메서드 안에서 모든 싱글톤 빈에 대해 `getBean(beanName)` 호출이 일어난다.

---

# 5단계. getBean() → 싱글톤 레지스트리 확인

여기서 드디어 싱글톤 레지스트리 등장!

```java
Object sharedInstance = getSingleton(beanName);
```

### 동작 방식

1. **singletonObjects**(캐시)에 이미 완성된 빈 있으면 → 그대로 반환
2. 없으면 → 새로 생성

---

# 6단계. 빈이 없다면 createBean() 수행 → 다시 싱글톤 레지스트리에 저장

싱글톤이 없을 때 스프링은 다음을 수행한다.

```java
Object singletonObject = singletonFactory.getObject(); // 실제로 객체 생성
this.singletonObjects.put(beanName, singletonObject);  // 등록
```

이 시점에 스프링 전체 라이프사이클 훅이 작동한다.

* 생성자 호출
* @Autowired 주입
* @PostConstruct 실행
* AOP 프록시 적용

프록시 적용 결과물 역시 싱글톤 레지스트리에 들어간다는 점이 중요하다.

---

# 7단계. 이후 모든 getBean() 요청은 캐시에서 바로 반환

애플리케이션이 정상 작동할 때
컨트롤러·서비스·레포지토리 등에서 getBean()이 호출되면 다음이 일어난다:

```
singletonObjects.get(beanName) → 바로 반환
```

객체 생성 비용이 0이 된다.
이것이 스프링 IoC 컨테이너가 싱글톤 기반으로 빠르게 동작하는 이유다.

---

# 8단계. 실제 코드 흐름을 한 문장으로 정리하면

```
ApplicationContext.refresh() 에서  
모든 싱글톤 bean이 createBean() 으로 한번만 생성되고  
singletonObjects(HashMap)에 저장되어  
DispatcherServlet 의 요청 처리 과정에서 계속 꺼내쓴다.
```

---

# + 스프링이 사용하는 디자인 패턴 설명 (싱글톤 레지스트리 내부)

### ✅ 1) 템플릿 메서드 패턴

createBean → doCreateBean → populate → initialize
계층화된 서브 단계마다 후처리 가능하게 함.

### ✅ 2) 팩토리 메서드 패턴

ObjectFactory로 bean 생성 로직을 분리.

### ✅ 3) 프록시 패턴

트랜잭션, AOP 적용된 프록시를 싱글톤으로 등록.

### ✅ 4) 전략 패턴

BeanNameGenerator, Scope 전략 등이 교체 가능.

---

# 🔥 전체 원리 정리

### ✔ DispatcherServlet 초기화 시 ApplicationContext 생성

### ✔ ApplicationContext.refresh()가 BeanFactory 생성 및 초기화 실행

### ✔ finishBeanFactoryInitialization()에서 모든 싱글톤 bean을 미리 생성

### ✔ 생성된 bean은 DefaultSingletonBeanRegistry 의 singletonObjects(ConcurrentHashMap)에 저장

### ✔ 그 이후의 모든 getBean() 요청은 해당 맵에서 동일 인스턴스 반환

### ✔ 결과: 스프링 애플리케이션은 싱글톤 기반 IoC 컨테이너 구조를 가진다

---

