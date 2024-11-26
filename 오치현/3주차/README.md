### 페이징과 한계 돌파

1. XToOne 관계를 모두 패치조인 한다. (XToOne 관계는 데이터베이스 row 수를 증가시키지 않음)
2. 나머지 컬렉션은 지연 로딩으로 조회
3. 이때, 지연 로딩 성능 최적화를 위해 Batch 사이즈를 조절한다.

   hibernate.default_batch_fetch_size : 글로벌 설정

   @BatchSize : 개별 최적화

   → 위 옵션 사용 시, 컬렉션이나 프록시 객체를 한꺼번에 설정한 사이즈 만큼 IN Query로 조회

   → 배치 사이즈는 100 ~ 1000으로 잡는 것이 좋다. 데이터베이스는 보통 1 000 개 이상을 In Query로 요청하면 오류를 반환한다. 애플리케이션은 전체 데이터를 로딩해야하기 때문에 결국 같은 양의 메모리를 사용하지만, 데이터베이스는 배치 사이즈에 따라 부하가 달라진다. → 순간 부하를 어디까지 견딜 수 있는지로 결정


[컬렉션 조회 최적화 V3]

```java
public List<Order> findAllWithItem() {
        return em.createQuery(
                "select distinct o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d" +
                        " join fetch o.orderItems oi" +
                        " join fetch oi.item i", Order.class
        ).getResultList();
    }
```

장점

1. 쿼리 한 번으로 모든 데이터를 가져온다.

단점

1. 중복된 데이터가 많아 페이징 처리를 못한다. ( 일대다 관계로 인한 row 수 증가 )
2. 데이터베이스에서 전송되는 데이터 용량이 크다.

[컬렉션 조회 최적화 V3.1]

```java
public List<Order> findAllWithMemberDelivery(int offset, int limit) {
        return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d", Order.class
                )
                .setFirstResult(offset)
                .setMaxResults(limit)
                .getResultList();
    }
```

장점

1. XToOne 관계를 우선적으로 패치 조인 ( 후에 단방 쿼리가 날아가서 성능상 크게 차이 없지만 최적화를 위해 패치 조인 )

   쿼리 호출 수가 1 + N → 1 + 1 로 최적화

2. 배치를 사용하기 때문에 컬렉션 조회도 최적화된다.
3. 페이징이 가능하다.

단점

1. V3 방식보다 쿼리가 많이 나간다. → 사실 테이블 별로 In Query를 날리는 것이기 때문에 정확히 필요한 데이터만 조회한다. 따라서, 성능상 좋을 수도, 좋지 않을 수도 있다. ( 상황마다 잘 판단해서 사용 )

[컬렉션 조회 최적화 V5]

```java
public List<OrderQueryDTO> findAllByDto_optimization() {
        List<OrderQueryDTO> result = findOrders();

        Map<Long, List<OrderItemQueryDTO>> orderItemMap = findOrderItemMap(toOrderIds(result));

        result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

        return result;
}

/* Id를 모아줌 */
private List<Long> toOrderIds(List<OrderQueryDTO> result) {
        List<Long> orderIds = result.stream()
                .map(OrderQueryDTO::getOrderId)
                .toList();
        return orderIds;
}

/* OrderQueryDTO를 일대일 관계 매핑 엔티티와 함께 조회 */
private List<OrderQueryDTO> findOrders() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.query.OrderQueryDTO(o.id, m.name, o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderQueryDTO.class
        ).getResultList();
}

/* Id를 InQuery로 조회 후, Collection을 이용해 Map을 생성 */
private Map<Long, List<OrderItemQueryDTO>> findOrderItemMap(List<Long> orderIds) {
        List<OrderItemQueryDTO> orderItems = em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDTO(oi.order.id, i.name, oi.orderPrice, oi.count)"
                                +
                                " from OrderItem oi" +
                                " join oi.item i" +
                                " where oi.order.id in :orderIds", OrderItemQueryDTO.class
                )
                .setParameter("orderIds", orderIds)
                .getResultList();

        Map<Long, List<OrderItemQueryDTO>> orderItemMap = orderItems.stream()
                .collect(Collectors.groupingBy(OrderItemQueryDTO::getOrderId));
        return orderItemMap;
}
```

→ 루트 쿼리 1번(ToOne 관계 먼저 조회), 서브 쿼리 1번 실행

메모리 상에서 OrderItems를 매핑함 → 매칭 성능 향상 (Map 이용, O(1))

[컬렉션 조회 최적화 V6]

```java
public List<OrderFlatDTO> findAllByDTO_flat() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.query.OrderFlatDTO(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d" +
                        " join o.orderItems oi" +
                        " join oi.item i", OrderFlatDTO.class)
                .getResultList();
}
// 이후 애플리케이션에서 중복 데이터 정리
```

장점

- 쿼리 1번

단점

- 쿼리는 1번 이지만, 한 번에 조인하기 때문에 DB에서 전송되는 데이터 양이 크다.
- 애플리케이션에서 추가 작업이 필요하다.
- 페이징 불가능

# 정리

### 엔티티 조회

V1 → 엔티티 조회, 그대로 반환

V2 → 엔티티 조회, DTO 반환

V3 → 패치 조인으로 쿼리 수 최적화

V3.1 → 컬렉션 페이징과 한계 돌파

컬렉션 패치 조인 시 페이징 불가능

ToOne 관계 → 패치 조인

ToMany 관계 → 배치 사이즈와 함께 Lazy Loading (최적화)

### DTO 직접 조회

V4 → JPA에서 DTO 직접 조회

V5 → 컬렉션 조회 최적화

In 절을 활용, 메모리에 미리 조회해서 최적화

V6 → Join으로 중복을 포함하여 한 번에 조회 후 (Flat), 애플리케이션에서 원하는 모양으로 변환

→ 결국 핵심! 여러 개는 모아서 조회한다

### 권장 순서

1. 엔티티 조회 방식으로 우선 접근
    1. 패치 조인으로 쿼리 수 최적화
    2. 컬렉션 최적화
        1. 페이징 필요 O → 배치 사이즈로 최적화
        2. 페이징 필요 X → 패치 조인 사용
2. DTO 조회 방식 사용
3. NativeSQL or 스프링 JdbcTemplate 사용

→ 엔티티 조회를 우선하는 이유

배치 사이즈와 패치 조인을 이용함으로써 다양한 최적화를 시도할 수 있다.

DTO 사용 시 SQL을 그대로 사용하는 것과 같기 때문에 코드에 변화가 생길 때나 최적화를 진행할 때 많은 수정이 필요하다.

성능에 문제가 생긴 경우 엔티티 조회 + 패치 조회 + 배치 사이즈 설정으로 웬만한 상황은 해결할 수 있다. 이후에도 해결이 되지 않은 경우 DTO 조회를 고민하게 되는데, 사실 캐시 사용 여부에 대해 고민하는 것이 더 효율적일 수 있다.

⚠️ 주의할 점 : 캐시에 엔티티를 적재하면 안된다. 캐시 삭제 시 영속성 컨텍스트에도 존재한다면 삭제되지 않는 현상 등 여러가지 꼬일 수 있다. 따라서 DTO로 변환해서 적재해야한다. (Hibernate 2차 캐시도 있는데 실무에 적용하기 어렵다고 함)

# OSIV와 성능 최적화

Open Session In View : 하이버네이트 (JPA의 Entity Manager 역할을 하이버네이트에서는 Session이 한다.)

Open EntityManager In View : JPA (관례상 OSIV)

![Untitled](ee%2014ab96b19b26801d89e8c467b7b692c0/Untitled.png)

open-in-view 속성이 `True` 값일 경우 트랜잭션을 벗어나도 사용자에게 응답이 전송될 때까지 DB Connection을 잡고 있는다. → 트랜잭션 밖에서 수정 쿼리 불가, 조회 쿼리만 가능

즉, 지연 로딩은 영속성 컨텍스트가 살아있어야 하고, 영속성 컨텍스트는 기본적으로 DB Connection을 유지한다.

장점

1. View 계층에서 지연 로딩 가능

단점

1. DB Connection을 너무 오래 물고 있음
2. 실시간 트래픽 처리 시스템에서 장애가 발생할 확률이 높음

![Untitled](ee%2014ab96b19b26801d89e8c467b7b692c0/Untitled%201.png)

장점

1. DB Connection 리소스를 낭비하지 않는다.

단점

1. 지연 로딩을 트랜잭션 안에서 처리해야 한다. → 별도의 쿼리 서비스 API를 만들어 해결한다. (도메인 별로 분리), 이외에 여러가지 방법이 있음

## Command와 Query 분리

분리하는 방법 중 OSIV를 끈 상태로 복잡성을 관리하는 방법이 있다.

참고 : [링크](https://en.wikipedia.org/wiki/Command-query_separation)

애플리케이션이 크고 복잡해질수록 커맨드와 쿼리를 분리하지 않으면 유지보수성이 급격하게 떨어진다.

따라서, 다음과 같이 분리하는 것이 좋다.

ex)

OrderService

OrderService : 핵심 비즈니스 로직

OrderQueryService : 화면이나 API에 맞춘 서비스 (주로 읽기 전용 트랜잭션 사용)

→ 애플리케이션의 규모가 작다면 OrderService에 한 번에 넣어도 된다.

# Spring Data JPA

JpaRepository라는 인터페이스를 제공하는데, 기본적인 CRUD 기능이 모두 제공된다.

공통적인 기능을 제외하고 함수명을 통해 JPQL 쿼리를 만들 수 있다.

```java
// select m from Member m where m.name = :name
List<Member> findByName(String name);
```