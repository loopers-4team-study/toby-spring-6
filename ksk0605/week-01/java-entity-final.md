# 왜 JPA Entity 에서는 final 키워드를 붙이지 않을까? 

> JPA Entity는 리플렉션 기반 초기화를 사용하기 때문이다. 

* JPA 는 Entity 를 인스턴스화 시킬때 기본 생성자(No Arg Constructor)로 먼저 객체를 생성한다. 
* 이후에 리플렉션으로 필드를 초기화를 할때, 필드가 final이면 값을 변경하는 것이 불가능하다. 
* 이 경우 다음과 같은 에러를 던질 수 있다. 
	```
	org.hibernate.InstantiationException: Cannot set final field ...
	```

---
> Entity class 에도 final 은 사용해서는 안된다. 

* JPA 엔티티는 런타임에 Proxy 객체로 만들어져서 작동한다. 
* 이는 Dirty Checking, Lazy Loading 등 다양한 기능을 제공하기 위함이다. 
* Proxy 를 내부에서 생성하기 위해선 기존 클래스를 상속하여 Subclass 로 만들어야하는데 이때 final 키워드가 붙어있으면 상속이 불가능하다. 

#### 예시 
```
Member member = entityManager.getReference(Member.class, 1L);
System.out.println(member.getClass()); 
// class com.example.Member$HibernateProxy$123abc  ← 실제 Member가 아닌 프록시 객체
```
