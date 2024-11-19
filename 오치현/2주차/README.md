# API 개발 고급

## 지연 로딩과 조회 성능 최적화

JOIN 쿼리가 발생하는 조회 API 작성 시, `엔티티를 직접 반환`할 경우 문제점!

```sql
// Order
public class Order {
	@ManyToOne(fetch = LAZY)
	@JoinColumn(name = "member_id")
	private Member member;
}

// Member
public class Member {
	@OneToMany(mappedBy = "member")
	private List<Order> orders = new ArrayList<>();
}
```

1. 무한 루프에 빠지게 된다.

   양방향 연관관계의 경우 서로를 참조하면서 무한 루프에 빠질 수 있다.

   이는 둘 중 하나의 관계를 @JsonIgnore 을 사용함으로써 해결할 수 있다.

   → @JsonIgnore 한 연관 관계가 지연로딩일 경우, 값을 읽어올 수 없어 문제가 발생한다!

2. 값을 읽어올 수 없음

   Spring 은 프록시 객체로 ByteBuddyInterface 클래스를 사용한다. @JsonIgnore 을 사용할 경우 지연 로딩으로 값을 읽지 않기 때문에 조회 쿼리가 발생하지 않아 값이 없는 상태이다. 이때, 해당 정보를 화면에 출력할 경우 서버 에러(500) 이 발생한다.

   → [해결방안]

    ```sql
    @Bean
    Hibernate5Module hibernate5Module() {
    	Hibernate5Module hibernate5Module = new Hibernate5Module();
    	hibernate5Module.configure(Hibernate5Module.Feature.FORCE_LAZY_LOADING, true);
    	return hibernate5Module;
    }
    ```

   Hibernate5Module을 사용하면 된다. (스프링 부트 3.0 버전 이상인 경우 Hibernate5JakarataModule)

   이는 지연 로딩된 객체에 대해 null 값을 채워주는 역할을 한다.

   데이터가 조회되어 들어있을 경우 무한 루프가 될 수 있으므로 해당 모듈을 사용했더라도 @JsonIgnore을 사용해야한다.

   → 하지만, 이것도 많은 서비스를 처리하게 되면 엔티티에 각각 다른 설정이 필요하게 되고, 부가적인 설정이 많이 필요해 좋은 방법이 아니다.

   ‼️ 엔티티를 반환하지 말자


❌ 주의

지연 로딩(LAZY)을 피하기 위해 즉시 로딩(EAGER)로 설정하면 안된다!

1. 연관관계가 필요 없는 경우에도 데이터를 항상 조회하게 된다.
2. 성능 튜닝이 매우 어려워진다. (N + 1 문제)

    ```sql
    private final EntityManager em;
    ~~
    em.find(Order.class, id);
    ```

   위의 경우 `Order` 클래스를 인자로 전달해주기 때문에 연관된 클래스 정보를 같이 조회할 때 `JOIN` 을 통해 최적화를 한다.

    ```sql
    orderRepository.findAllByString(new OrderSearch());
    ```

   위의 경우 `JPQL` 이 날라가고 그대로 쿼리로 변환되기 때문에 최적화가 되지 않는다. 이때 `EAGER` 로 설정되어있다고 하더라도 `LAZY` 처럼 단건 조회 쿼리가 날라간다.

   → N + 1 문제가 터짐!


→ 항상 `지연 로딩` 을 기본으로 하고, 성능 최적화가 필요한 경우에는 `패치 조인` 을 사용해야 한다.

리스트 데이터 반환 시 Result 클래스로 한 번더 감싸야 함.

DTO를 이용해서 해결해도 LAZY 가 설정되어 있으면, 추가적인 쿼리가 발생한다. (1 + N) 문제 → fetch join으로 해결

🔊 참고 : 지연로딩은 영속성 컨텍스트에서 조회하므로, 이미 조회된 경우 쿼리를 생략한다. (트렌젝션 안에서만 존재하는게 아닌듯..?)

최종 해결 코드

[V3] Fetch Join

```java
public List<Order> findAllWithMemberDelivery() {
        return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d", Order.class
        ).getResultList();
    }
```

→ 엔티티를 반환하기 때문에 재사용이 가능하고 이후 수정도 간편하다.

→ V4 보다는 최적화 성능이 좋진 않다.

[V4] Fetch Join & DTO

JPA 는 Value Object (Entity, Embed) 만 반환할 수 있다. DTO X

그래서 쿼리 내에서 DTO로 변환할 경우 조금 지저분해 진다.

```java
select Packages/.../DTO(o.value1, o.value2, ...) from Order o ....
```

```java
public List<OrderSimpleQueryDto> findOrderDtos() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderSimpleQueryDto.class
        ).getResultList();
    }
```

→ 요구 사항에 핏하게 제작되어 재사용이 어렵다. 코드가 더 지저분 하다

값을 풀어서 넣는 이유!

```sql
"select new jpabook.jpashop.repository.OrderSimpleQueryDto(o) from Order o"
```

→ 엔티티를 인자로 넣으면 Order의 식별자 값이 들어간다.

[V4] 에서 Address는 Value Object 이기 때문에 인자로 넣을 수 있다!

→ `new` 명령어를 사용해서 `JPQL`의 결과를 `DTO`로 즉시 변환

→ V3 보다는 최적화 성능이 좋다. (생각보다 미비)

성능이 좋다? → 원하는 값만 조회한다. 네트워크 IO에 전달하는 데이터 양이 적어진다.

→ Repository 의 본래 엔티티 조회라는 개념이 망가진다.

API 스펙에 맞춘 조회 코드가 Repository 안에 들어온다.

Repository 폴더 안에 해당 쿼리 폴더를 만들고, 쿼리에 대한 Repository를 하나 더 생성해서 분리한다.

[정리 : 쿼리 방식 선택 권장 순서]

1. 우선 엔티티를 DTO로 변환하는 방법을 선택한다.
2. 패치 조인으로 성능을 최적화 한다. → 대부분의 문제가 해결됨
3. DTO를 직접 조회한다.
4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용한다.

## API 개발 고급 - 컬렉션 조회 최적화

엔티티 안에 엔티티가 있을 때 DTO 로 변환한다면 안에 있는 엔티티도 DTO로 변환해야 한다. → fetch join 으로 최적화

[잘못된 DTO - DTO 안에 엔티티]

```sql
static class OrderDto {
	private List<OrderItem> orderItems;
	
	public OrderDto(Order order) {
		~~
	}
}
```

[개선된 DTO - 전부 DTO로 변환]

```java
static class OrderDto {
	private List<OrderItemDto> orderItems;
	
	public OrderDto(Order order) {
		orderItems = order.getOrderItems().stream()
						.map(orderItem -> new OrderItemDto(orderItem))
						~~
	}
}

static class OrderItemDto {
	
	private String itemName;
	private int orderPrice;
	private int count;
	
	public OrderItemDto(OrderItem orderItem) {
		~~
	}
}
```

강의의 [V1] → [V2] 는 DTO, [V2] → [V3] 는 Fetch Join의 내용을 다룬다.

컬렉션에서의 Fetch Join

```java
em.createQuery(
	"select o from Order o" +
	" join fetch o.member m" +
	" join fetch o.delivery d" +
	" join fetch o.orderItems oi" +
	" join fetch oi.item i", Order.class)
```

위와 같은 쿼리의 문제점 : 데이터가 뻥튀기 된다!

데이터베이스 특성 상 JOIN으로 생성된 테이블은 기존 테이블의 중복된 값을 가질 수 있다.

| order_id | order_item_id | item_id |
| --- | --- | --- |
| 1 | 1 | 1 |
| 1 | 1 | 2 |

JPA의 구현체인 하이버네이트는 테이블 내에 중복이 있으면 그대로 처리할지, 제거하고 처리할지 모른다!

→ 결국 결과 row 수 만큼 객체를 가져옴 (이때, 영속성 컨텍스트의 pk 값이 같으면 같은 엔티티로 보는 특성으로 인해 실제 객체의 수는 원래의 수가 된다. → 중복될 경우 같은 레퍼런스를 참조함)

데이터가 뻥튀기 되는 문제는 같은 레퍼런스를 가지는 객체가 여러 개라는 것인데, 같은 레퍼런스의 객체는 무시할 수 없을까?

→ JPQL 쿼리에 `distinct` 넣기!

```java
em.createQuery(
	"select distinct o from Order o" +
            " join fetch o.member m" +
            " join fetch o.delivery d" +
            " join fetch o.orderItems oi" +
            " join fetch oi.item i", Order.class)
```

‼️ DB의 DISTINCT 키워드와 다르다.

DB의 `DISTINCT` 는 각 row의 모든 attribute 값이 같아야 `중복` 으로 판단하고 행을 제거해준다.

JPQL의 `distinct` 는 애플리케이션 상에서 PK 값이 같을 경우 제거해준다.

→ ‼️ 주의 : 일대다 Fetch Join 시, 페이징이 불가능하다. (실행 시 하이버네이트가 경고를 날리 고, 쿼리 조회 후 메모리에서 페이징 처리를 함)

→ ‼️ 주의 : 일대다 Fetch Join은 최대 1개만 가능하다! 2개 이상 존재하면 1 x N x M 이 되어서 데이터 정합성을 맞추기 어려워진다. (부정합한 데이터 조회, 보통 컬렉션 Fetch Join이 2개 이상이면 경고를 냄)