# 왜 JPA Entity에서는 ID 를 다룰때 privitive 타입이 아닌 Wrapper Class 를 사용할까? 

> JPA가 이미 저장된 Entity 와 새로운 Entity 를 구분하는 방법이 ID 의 null 여부이기 때문이다. 

JPA 는 엔티티를 저장할 때, 
* id가 null 이면 -> 새 엔티티로 인식하여 insert
* id가 있으면 -> 이미 존재하는 엔티티로 인식하여 update 
즉, id 의 존재여부로 신규/기존을 구분한다. 

그러나 primtive type은 null표현이 불가능하다. 초기화를 해주지 않으면 기본 값이 설정된다. 
(long 의 경우 0)

#### JPA 식별자 생성전략 호환성

JPA `@GeneratedValue`는 
1. 엔티티 저장 전까지 id 값은 null 
2. persist() 시점에 DB에서 생성된 값(시퀀스, auto-increment등)을 주입
즉, 영속화를 기점으로 id 를 초기화하게 된다. 

---
추가 키워드 
* transient entity
