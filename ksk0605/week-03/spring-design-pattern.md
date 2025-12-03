아주 좋은 질문입니다.
스프링 핵심 구조를 이해하려면 **BeanFactory / BeanRegistry / ApplicationContext**의 관계와
스프링이 자주 사용하는 **계층적 추상화 패턴**을 정확히 이해하는 것이 필수입니다.

아래 내용은 스프링 프레임워크 실제 소스 구조를 기반으로 “원리 중심”으로 설명드립니다.

---

# 1. BeanFactory vs BeanRegistry vs ApplicationContext

스프링 컨테이너의 계층 구조를 정확히 구분해보자.

---

# ✔ (1) BeanFactory — "DI 컨테이너의 가장 최소 단위"

### 핵심 역할

* **빈을 생성하고 공급(getBean)하는 최소한의 기능만 제공**
* DI 컨테이너의 스펙을 정의하는 **인터페이스**

핵심 메서드:

```java
Object getBean(String name);
boolean containsBean(String name);
Class<?> getType(String name);
```

BeanFactory는 “어떻게 빈을 등록할지” 책임이 없다.
즉, **BeanDefinition 생성·등록은 BeanFactory의 역할이 아님**.

---

# ✔ (2) BeanRegistry (정확히는 BeanDefinitionRegistry)

> “BeanFactory가 사용할 BeanDefinition을 등록하는 역할”

스프링에는 `BeanRegistry` 이름의 타입은 없고
정확한 이름은 **BeanDefinitionRegistry**다.

### BeanDefinitionRegistry의 역할

* BeanDefinition 을 등록/제거하는 기능을 제공
* 즉, **어떤 빈을 만들지 스펙(정의)를 저장**

메서드 대표 예:

```java
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition);
void removeBeanDefinition(String beanName);
BeanDefinition getBeanDefinition(String beanName);
```

### 즉:

* BeanFactory = “빈을 만들어서 관리”
* BeanDefinitionRegistry = “빈 정보를 등록하고 저장”

두 책임이 완전히 다르다.

---

# ✔ (3) ApplicationContext — “BeanFactory + 그 외 모든 스프링 기능”

ApplicationContext는 다음을 전부 포함한다:

| 기능                  | 설명                                |
| ------------------- | --------------------------------- |
| DI/IoC 컨테이너         | BeanFactory 상속                    |
| BeanDefinition 등록   | BeanDefinitionRegistry 포함         |
| 환경 관리               | Environment, PropertySource       |
| 국제화                 | MessageSource                     |
| 이벤트 발행              | ApplicationEventPublisher         |
| AOP 자동설정            | BeanPostProcessor 관리              |
| 리소스 로딩              | ResourceLoader                    |
| AnnotationConfig 처리 | @ComponentScan, @Configuration 해석 |

즉 **ApplicationContext 는 BeanFactory보다 훨씬 상위 개념**이다.

정리:

```
ApplicationContext
    ㄴ BeanFactory (DI 컨테이너 기능)
    ㄴ BeanDefinitionRegistry (빈 정의 등록 기능)
    ㄴ MessageSource (국제화)
    ㄴ ApplicationEventPublisher (이벤트)
    ㄴ ResourceLoader (리소스 로딩)
    ㄴ EnvironmentCapable (환경)
    ㄴ 기타 수많은 인프라
```

---

# 📌 한 줄 정리

* **BeanFactory**: 빈을 생성/조회하는 핵심 DI 컨테이너
* **BeanDefinitionRegistry**: 빈의 정의(메타정보)를 등록하는 저장소
* **ApplicationContext**: 빈 생성 + 빈 정의 등록 + 메시지/환경/이벤트 등 스프링의 확장 기능 모두 포함하는 최상위 컨테이너

스프링에서 우리가 대부분 쓰는 건 ApplicationContext이다.

---

# 2. 스프링의 “인터페이스 → 추상 클래스 → 구현체” 구조는 어떤 패턴인가?

스프링 곳곳에서 아래 같은 구조가 반복된다:

```
ApplicationContext
  ↑
AbstractApplicationContext
  ↑
AnnotationConfigApplicationContext
```

또는:

```
BeanFactory
  ↑
AbstractBeanFactory
  ↑
DefaultListableBeanFactory
```

이 구조는 단순 상속이 아니라 **스프링 컨테이너 아키텍처 전체를 관통하는 설계 패턴**이다.
이 구조는 두 가지 패턴의 조합으로 이해해야 한다.

---

# ✔ 패턴 1) Template Method Pattern (템플릿 메서드 패턴)

### 구조

* **상위 추상 클래스(AbstractXXX)** 가 전체 알고리즘의 흐름을 정의하고
* **하위 구현체(XXXImpl)** 가 특정 단계만 오버라이딩해서 확장한다.

스프링에서 가장 유명한 예:

```java
public abstract class AbstractApplicationContext {
    public void refresh() {
        prepareRefresh();
        obtainFreshBeanFactory();
        postProcessBeanFactory(beanFactory);
        invokeBeanFactoryPostProcessors(beanFactory);
        registerBeanPostProcessors(beanFactory);
        finishBeanFactoryInitialization(beanFactory);
    }
}
```

하위 구현체는 필요한 부분만 확장:

```java
public class AnnotationConfigApplicationContext extends AbstractApplicationContext {
    @Override
    protected void prepareRefresh() {
        // annotation 기반 스캔 준비
    }
}
```

### 왜 스프링이 템플릿 메서드를 좋아하나?

* 스프링같은 대규모 프레임워크는
  “공통 틀은 유지하면서 확장 포인트를 열어둬야 하기 때문”

---

# ✔ 패턴 2) Hook Pattern (헐리우드 원칙)

> “우리가 당신을 호출할게, 당신이 우리를 호출하지 마라”

상위 클래스가 전체 플로우를 제어하고
하위 클래스는 callback(hook) 형태로 일부만 제공한다.

스프링의 AbstractXXX 대부분이 이 철학을 따른다.

---

# ✔ 패턴 3) Strategy Pattern + Interface 기반 추상화

`XXX` 인터페이스가 역할(계약)을 정의하고
`AbstractXXX` 가 기본 구현/공통 흐름을 제공
`ConcreteXXX`(구현 클래스)가 전략을 확정한다.

이 조합이 스프링 구조의 핵심이다.

---

# 🔥 이름을 한 줄로 정리하면?

스프링의 구조는 **템플릿 메서드 패턴 기반의 계층적 추상화(Hierarchical Abstraction)** 구조이다.

좀 더 정확히 말하면:

> **“인터페이스 기반 전략 패턴 + 추상클래스 기반 템플릿 메서드 패턴의 결합”**

스프링 프레임워크에서 가장 많이 쓰이는 구조다.

---

# 🔥 최종 요약

## ✔ BeanFactory / BeanDefinitionRegistry / ApplicationContext

* BeanFactory = 빈 생성/조회 담당
* BeanDefinitionRegistry = 빈 정의 등록/관리 담당
* ApplicationContext = BeanFactory + Registry + 이벤트, 메시지, 환경까지 포함한 “종합 컨테이너”

## ✔ 스프링 클래스 구조의 패턴

* **템플릿 메서드 패턴 (핵심)**
* 인터페이스 기반 전략 패턴
* 계층적 추상화(Hierarchical Abstraction)
* Hook Method pattern(헐리우드 원칙)

이 조합으로 스프링은 매우 유연하면서 확장 가능한 구조를 만든다.

---

원하면 다음도 설명드릴 수 있어요:

🔥 “ApplicationContext.refresh() 내부 구조 전체 흐름도”
🔥 “BeanFactoryPostProcessor와 BeanPostProcessor는 어느 단계에서 개입하나?”
🔥 “왜 BeanDefinitionRegistry와 BeanFactory가 분리되어야 하는가?”

궁금한 부분 있으시면 더 깊게 들어가 보죠!
