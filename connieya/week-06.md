# 추상화

- 구현의 복잡함과 디테일 감추고 중요한 것만 남기는 기법
- 여러 인프라 기술의 공통적이고 핵심적인 기능을 인터페이스로 정의하고 이를 구현하는 어댑터를 만들어 일관된 사용이 가능하게 만드는 것이 서비스 추상화

# OrderService 에서 기술 관련 코드 제거

- 데이터 기술이 변경되어도 기존 코드는 영향을 받지 않음
- TransactionTemplate , PlatformTransactionManager 와 같은 기술과 연관된 코드가 계속 등장함
- 트랜잭션 시작과 종료는 보통 애플리케이션 서비스 메소드 실행 전후

# 트랜잭션 테스트의 어려움

- 트랜잭션이 필요한 곳에 정확하게 적용되었는지 테스트 하기는 매우 어려움
- JDBC 처럼 자동 커밋이 되거나 Spring Data JPA처럼 기본 리포지토리 구현에서 트랜잭션을 알아서 적용해주는 기술을 사용할 때 트랜잭션이 바르게 적용되지 않는 것을 놓치기 쉬움
- 모든 작업이 성공하면 하나의 트랜잭션으로 진행된 것인지 여러개의 트랜잭션으로 쪼개진 것인지 확인하기 어려움
- 트랜잭션 중간에 실패하는 케이스를 만들 수 있다면 롤백 여부로 확인할 수 있음

# 트랜잭션이 쪼개지는 문제 예시

## 문제 상황

```java
public List<Order> createOrders(List<OrderReq> reqs) {
    return reqs.stream().map(req -> createOrder(req.no(), req.total()))
        .toList();
}

public Order createOrder(String no, BigDecimal total) {
    Order order = new Order(no, total);
    this.orderRepository.save(order);  // 각각이 별도 트랜잭션!
    return order;
}
```

## 문제점

- `createOrder()`에 `@Transactional`이 없음
- 각 `save()` 호출마다 **별도의 트랜잭션**으로 실행됨
- `createOrders()`에서 여러 `createOrder()`를 호출하면:
    - 첫 번째 성공 → 커밋 ✅
    - 두 번째 성공 → 커밋 ✅
    - 세 번째 실패 → 롤백 불가 ❌ (이미 커밋됨)

## 기대 vs 실제

| 기대 | 실제 |
|------|------|
| 전부 성공하거나 전부 실패 (원자성) | 중간에 실패해도 이전 작업은 이미 커밋됨 |
| 하나의 트랜잭션으로 묶임 | 여러 개의 트랜잭션으로 쪼개짐 |

# 왜 이런 문제가 발생하는가?

## JDBC의 자동 커밋

- **JDBC Connection은 기본적으로 `autocommit = true` 상태**
- 각 SQL 문이 실행될 때마다 **자동으로 커밋됨**
- 명시적으로 `connection.setAutoCommit(false)`를 하지 않으면 자동 커밋

```java
// 순수 JDBC
Connection conn = dataSource.getConnection();
// conn.getAutoCommit() == true (기본값)

PreparedStatement stmt = conn.prepareStatement("INSERT INTO orders ...");
stmt.executeUpdate();  // ← 여기서 자동 커밋됨!
// commit() 호출 불필요
```

## Spring Data JPA의 트랜잭션 처리

- Spring Data JPA의 기본 메서드 (`save()`, `findById()`, `deleteById()` 등)는 **내부적으로 `@Transactional`이 적용됨**
- 별도로 `@Transactional`을 붙이지 않아도 각 작업이 하나의 트랜잭션으로 처리됨
- 하지만 **각 메서드 호출마다 별도의 트랜잭션**으로 실행됨

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // save(), findById() 등은 내부적으로 트랜잭션 적용됨
    // 하지만 각 메서드 호출마다 별도 트랜잭션
}
```

## 핵심 정리

- `orderRepository.save(order)` **자체는 자동으로 커밋됨**
- 여러 작업을 하나의 트랜잭션으로 묶으려면 **명시적인 트랜잭션 관리가 필요함**

# 해결 방법

## 방법 1: @Transactional (선언적)

```java
@Transactional
public List<Order> createOrders(List<OrderReq> reqs) {
    return reqs.stream().map(req -> createOrder(req.no(), req.total()))
        .toList();
}
```

### 동작 원리

- 메서드 시작 시 **트랜잭션 시작**
- 모든 `createOrder()` → 모든 `save()`가 **하나의 트랜잭션**에 포함
- 메서드 정상 종료 시 **커밋**
- 예외 발생 시 **전부 롤백**

## 방법 2: TransactionTemplate (명시적)

```java
public List<Order> createOrders(List<OrderReq> reqs) {
    return new TransactionTemplate(transactionManager).execute(status ->
        reqs.stream().map(req -> createOrder(req.no(), req.total()))
        .toList()
    );
}
```

### 동작 원리

1. `TransactionTemplate.execute()` 시작 시 **트랜잭션 시작**
2. `execute()` 내부의 모든 작업이 **하나의 트랜잭션**에 포함
3. 정상 종료 시 **자동 커밋**
4. 예외 발생 시 **자동 롤백**

### 결과

- ✅ 전부 성공 → 전부 커밋
- ✅ 중간 실패 → 전부 롤백 (원자성 보장)

## 비교

| 방법 | 특징 |
|------|------|
| **@Transactional** | 선언적 트랜잭션 관리, AOP 기반, 코드가 간결함 |
| **TransactionTemplate** | 명시적 트랜잭션 관리, 프로그래밍 방식, 코드에 명확히 드러남 |
