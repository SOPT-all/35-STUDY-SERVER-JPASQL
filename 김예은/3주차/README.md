# 📌 상품 도메인 개발

### 상품 엔티티 코드

```java
package jpabook.jpashop.domain.item;
import jpabook.jpashop.exception.NotEnoughStockException;
import lombok.Getter;
import lombok.Setter;
import jpabook.jpashop.domain.Category;
import javax.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "dtype")
@Getter @Setter
public abstract class Item {
   @Id @GeneratedValue
   @Column(name = "item_id")
   private Long id;

   private String name;
   private int price;
   private int stockQuantity;

   @ManyToMany(mappedBy = "items")
   private List<Category> categories = new ArrayList<Category>();

   //==비즈니스 로직==//
   public void addStock(int quantity) {
       this.stockQuantity += quantity;
   }
   /* 메서드는 파라미터로 넘어온 수만큼 재고를 늘린다. 이 메서드는 재고가 증가하거나 상품 주문을 취소해서 재고를 다시 늘려야 할 때 사용한다. */

   public void removeStock(int quantity) {
       int restStock = this.stockQuantity - quantity;
       if (restStock < 0) {
           throw new NotEnoughStockException("need more stock");
       }
       this.stockQuantity = restStock;
   }
}

/* 메서드는 파라미터로 넘어온 수만큼 재고를 줄인다. 만약 재고가 부족하면 예외가 발생한다. 주로 상품을 주문할 때 사용한다.*/

```

### 예외 추가

```java
package jpabook.jpashop.exception;

public class NotEnoughStockException extends RuntimeException {
    public NotEnoughStockException() {
    }

    public NotEnoughStockException(String message) {
        super(message);
    }

    public NotEnoughStockException(String message, Throwable cause) {
        super(message, cause);
    }

    public NotEnoughStockException(Throwable cause) {
        super(cause);
    }
}

```

**예외 처리** : 재고가 부족하면 `NotEnoughStockException`을 발생시켜 문제를 명확히 전달합니다.

# 📌 주문 도메인 개발

## ✅  **디자인 패턴: 비즈니스 로직 구현 방식에 따라**

비즈니스 로직 구현은 크게 **트랜잭션 스크립트 패턴**과 **도메인 모델 패턴**으로 나뉘며, 두 접근 방식은 로직의 위치와 처리 방식에서 차이가 납니다. 각 패턴은 서로 상반된 특성을 가지며, 장단점과 적합한 상황이 달라 프로젝트 요구사항에 따라 적절히 선택해야 합니다.

---

### 1. **트랜잭션 스크립트 패턴**

- **로직 위치**: 서비스 계층이 로직을 중심적으로 처리하고, 엔티티는 데이터를 전달하고 저장하는 역할만 담당합니다.
- **특징**: 엔티티는 단순한 데이터 구조로 동작하며, 서비스 계층에 모든 로직이 집중되어 비즈니스 로직 흐름이 명확합니다.
- **장점**: 빠르고 직관적인 개발이 가능하며, 설계와 구현이 단순해 초기 진입 장벽이 낮습니다.
- **단점**: 코드 중복이 발생하기 쉬우며, 로직이 복잡해질수록 서비스 계층이 비대해지고 유지 보수가 어려워질 수 있습니다. 객체 지향 설계 활용이 제한적입니다.
- **적합한 상황**: 간단한 애플리케이션, 빠른 개발이 필요한 초기 단계, 엔티티가 단순히 데이터를 전달하는 역할을 하는 경우.

---

### 2. **도메인 모델 패턴**

- **로직 위치**: 엔티티 내부에 로직을 포함시켜 데이터 관리와 비즈니스 로직 처리를 함께 수행합니다.
- **특징**: 엔티티가 능동적으로 로직을 처리하며, 서비스 계층은 단순히 엔티티 메서드를 호출하거나 조율하는 역할만 수행합니다.
- **장점**: 로직의 재사용성과 응집도가 높아 유지 보수가 용이하며, 객체 지향 설계를 활용해 확장성과 유연성이 뛰어납니다.
- **단점**: 구현이 복잡하며, 설계 및 데이터베이스 매핑에 시간과 노력이 필요하고, 객체 지향 설계를 이해하지 못하면 구현 난이도가 높습니다.
- **적합한 상황**: 복잡한 비즈니스 로직을 처리해야 하거나, 장기적 유지 보수가 중요한 대규모 애플리케이션, 객체 지향 설계를 최대한 활용하려는 경우.

---

### **트랜잭션 스크립트 vs 도메인 모델: 선택 기준**

- **트랜잭션 스크립트 패턴**은 간단한 로직과 빠른 개발이 필요한 경우 적합하며, 초기 개발이 쉽고 직관적입니다.
- **도메인 모델 패턴**은 복잡한 로직과 높은 유지 보수성이 요구되는 프로젝트에 적합하며, 객체 지향 설계를 기반으로 데이터와 로직의 응집도를 높입니다.
- 두 패턴은 혼합 사용도 가능하며, 단순한 엔티티는 트랜잭션 스크립트 패턴을, 복잡한 도메인 로직이 필요한 경우에는 도메인 모델 패턴을 사용하는 방식이 유연한 설계를 도와줍니다.

## ✅ **영속성 전이(Persistence Cascade)**

### **1. 개념**

- 영속성 전이는 **특정 엔티티를 영속화할 때**, 연관된 엔티티도 함께 **영속 상태로 전이**시키는 기능입니다. 즉, 영속성을 전이한다. (영속성 상태를 전염시킨다는 느낌으로 받아들여보자 !)
- 이는 **JPA의 편의 기능**으로, 부모 엔티티를 저장하거나 삭제할 때, 자식 엔티티도 자동으로 처리됩니다.
- **중요**: 영속성 전이는 연관관계 매핑(association mapping)과는 관련이 없습니다. 이는 단지 엔티티를 저장하거나 삭제할 때 **편리함**을 제공할 뿐입니다.

---

### **2. 영속성 전이가 필요한 경우**

1. **부모-자식 관계**:
    - 부모 엔티티가 자식 엔티티의 생명주기를 완전히 관리하는 경우.
    - 예: `Order`와 `Delivery`, `OrderItem`처럼 부모가 생성되거나 삭제되면 자식도 같이 처리되어야 하는 관계.
2. **소유자가 하나뿐인 관계**:
    - 연관된 엔티티가 다른 곳에서 참조되지 않는 경우.
    - 예: `Delivery`가 다른 엔티티에서 참조되지 않고 `Order`만 소유할 때.

---

### **3. 영속성 전이 설정 (CascadeType)**

JPA에서 제공하는 **CascadeType** 옵션은 다음과 같습니다:

1. **`PERSIST`**: 부모를 저장하면 연관된 자식도 함께 저장됩니다.
2. **`MERGE`**: 부모 엔티티를 병합하면 연관된 자식도 병합됩니다.
3. **`REMOVE`**: 부모 엔티티를 삭제하면 연관된 자식도 삭제됩니다.
4. **`REFRESH`**: 부모 엔티티를 새로고침하면 연관된 자식도 새로고침됩니다.
5. **`DETACH`**: 부모 엔티티를 준영속 상태로 만들면 연관된 자식도 준영속 상태가 됩니다.
6. **`ALL`**: 위의 모든 동작을 포함합니다 (`PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`).

---

### **4. 사용 예**

`Order` 엔티티

```java
@Entity
@Table(name = "orders")
@Getter @Setter
@NoArgsConstructor
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();

    @OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    @JoinColumn(name = "delivery_id")
    private Delivery delivery;
}
```

- `Order`를 저장하면: `OrderItem`과 `Delivery`도 함께 저장됩니다.
- `Order`를 삭제하면: `OrderItem`과 `Delivery`도 함께 삭제됩니다.

---

### **5. 적용 시 주의사항**

1. **연관된 엔티티의 소유 여부**:
    - 연관된 엔티티가 다른 곳에서 참조되지 않는 경우에만 영속성 전이를 사용하는 것이 안전합니다.
    - 예: `Delivery`가 `Order`에만 속하는 경우.
2. **CascadeType.ALL 사용 주의**:
    - 모든 전이 동작이 포함되므로, 삭제(`REMOVE`) 시에도 자식 엔티티가 삭제됩니다. 이는 의도치 않은 데이터 손실로 이어질 수 있습니다.
3. **대규모 연관 데이터**:
    - 연관된 자식 엔티티가 매우 많을 경우, 성능 이슈가 발생할 수 있으므로 사용을 신중히 검토해야 합니다.

---

### **6. 영속성 전이의 이점**

- **코드 간결화**: 부모 엔티티만 처리하면 연관된 자식 엔티티도 자동으로 처리되므로 코드가 단순해집니다.
- **생명주기 동기화**: 부모와 자식 엔티티의 생명주기를 일치시킬 수 있습니다.
- **유지 보수성 향상**: 수동으로 자식 엔티티를 처리할 필요가 없어 관리가 편리합니다.

---

### **7. 영속성 전이가 없는 경우**

영속성 전이를 설정하지 않으면 부모 엔티티를 저장하거나 삭제할 때, 연관된 자식 엔티티는 별도로 처리해야 합니다. 이 경우, 자식 엔티티를 개별적으로 `persist`, `merge`, `remove`해야 하므로 코드가 복잡해질 수 있습니다.

# 📌 주문 검색 기능 개발

### **동적 쿼리 처리**

주문 검색 기능은 동적으로 검색 조건을 받아 JPA를 사용하여 주문 데이터를 조회하는 작업입니다. 이를 구현하기 위해 다양한 방법을 사용할 수 있으며, 각 방법의 장단점과 구현 방식을 정리했습니다.

---

### **1. 검색 조건 파라미터**

검색 조건을 담는 **OrderSearch** 클래스: 검색 조건을 캡슐화하여 **동적 검색**을 쉽게 구현합니다.

```java
public class OrderSearch {
    private String memberName;      // 회원 이름 검색조건
    private OrderStatus orderStatus; // 주문 상태 [ORDER, CANCEL] 검색조건
    // Getter, Setter
}
```

### **2. JPQL(Java Persistence Query Language)을 사용한 동적 쿼리**

☑️ **개념**

- **JPQL**은 JPA 표준 쿼리 언어로, SQL과 유사하지만 **객체 중심**으로 작성됩니다.
- 데이터베이스 테이블 대신 **엔티티**와 그 **필드**를 기준으로 쿼리를 작성합니다.
- SQL과 비슷한 문법을 사용하지만, JPA에서 지원하는 추상화된 쿼리입니다.

☑️ **알아둘 점 !**

1. **객체 기반**:
    - 엔티티 이름과 필드를 기준으로 작성합니다.
    - 데이터베이스 독립적이며, 테이블 대신 엔티티를 참조합니다.
2. **정적 쿼리**:
    - 컴파일 시점에 쿼리 문자열의 문법 오류를 확인할 수 없습니다.
    - 쿼리는 문자열로 작성되며, 실행 시점에 파싱됩니다.

**JPQL 기반 검색 메서드**:

``` java
public List<Order> findAllByString(OrderSearch orderSearch) {
    String jpql = "select o From Order o join o.member m";
    boolean isFirstCondition = true;

    // 주문 상태 검색 조건 추가
    if (orderSearch.getOrderStatus() != null) {
        jpql += isFirstCondition ? " where" : " and";
        jpql += " o.status = :status";
        isFirstCondition = false;
    }

    // 회원 이름 검색 조건 추가
    if (StringUtils.hasText(orderSearch.getMemberName())) {
        jpql += isFirstCondition ? " where" : " and";
        jpql += " m.name like :name";
    }

    TypedQuery<Order> query = em.createQuery(jpql, Order.class).setMaxResults(1000);

    // 파라미터 바인딩
    if (orderSearch.getOrderStatus() != null) {
        query.setParameter("status", orderSearch.getOrderStatus());
    }
    if (StringUtils.hasText(orderSearch.getMemberName())) {
        query.setParameter("name", orderSearch.getMemberName());
    }

    return query.getResultList();
}
```

☑️ **특징**:

1. **장점**:
    - **직관적**: JPQL 문법은 SQL과 유사하여 이해하기 쉽습니다.
    - **유연성**: 문자열로 동적으로 쿼리를 구성할 수 있습니다.
2. **단점**:
    - **복잡성**: 조건이 많아질수록 문자열로 쿼리를 생성하는 작업이 번거롭고, 실수로 인한 버그 발생 가능.
    - **가독성 저하**: 조건이 많아질수록 쿼리 생성 코드가 길어지고 관리하기 어렵습니다.

### **3. JPA Criteria를 사용한 동적 쿼리**

☑️ **개념**

- JPA 표준 스펙으로 제공되는 **타입 안전** 쿼리 작성 방식입니다.
- 문자열 대신 메서드 체인을 사용하여 쿼리를 작성하며, 동적 쿼리 작성에 적합합니다.
- JPQL의 단점을 보완하기 위해 등장했지만, 코드가 복잡해 실무에서는 잘 사용되지 않습니다.

**Criteria 기반 검색 메서드**:

``` java
public List<Order> findAllByCriteria(OrderSearch orderSearch) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Order> cq = cb.createQuery(Order.class);
    Root<Order> o = cq.from(Order.class);
    Join<Order, Member> m = o.join("member", JoinType.INNER);

    List<Predicate> criteria = new ArrayList<>();

    // 주문 상태 검색 조건 추가
    if (orderSearch.getOrderStatus() != null) {
        Predicate status = cb.equal(o.get("status"), orderSearch.getOrderStatus());
        criteria.add(status);
    }

    // 회원 이름 검색 조건 추가
    if (StringUtils.hasText(orderSearch.getMemberName())) {
        Predicate name = cb.like(m.<String>get("name"), "%" + orderSearch.getMemberName() + "%");
        criteria.add(name);
    }

    cq.where(cb.and(criteria.toArray(new Predicate[0])));
    TypedQuery<Order> query = em.createQuery(cq).setMaxResults(1000);

    return query.getResultList();
}
```

☑️ **특징**:

1. **장점**:
    - **표준 스펙**: JPA 표준으로 제공되므로 라이브러리 의존 없이 사용할 수 있습니다.
    - **타입 안전성**: SQL처럼 문자열이 아닌 메서드 체인을 사용하므로 컴파일 시점에 오류를 감지할 수 있습니다.
2. **단점**:
    - **복잡성**: 코드가 길어지고 가독성이 떨어져 실무에서 사용하기 어렵습니다.
    - **유연성 부족**: 동적 쿼리를 작성하기 번거롭고, 수정 시 수정 범위가 넓어질 수 있습니다.

### **4. QueryDSL로의 전환**

☑️ **개념**

- QueryDSL은 JPA를 위한 **타입 안전** SQL/JPQL 대체 라이브러리입니다.
- JPQL 및 Criteria의 단점을 보완하여, **동적 쿼리 작성**과 **코드 가독성**을 개선합니다.
- 쿼리 생성기를 통해 엔티티 기반의 **Q타입 클래스**를 생성하여 사용합니다.

☑️ **특징**

1. **타입 안전성**:
    - Q타입 클래스를 사용하므로, 컴파일 시점에 쿼리 오류를 감지할 수 있습니다.
2. **간결성**:
    - JPQL과 Criteria보다 간결하고 직관적인 코드를 작성할 수 있습니다.
3. **동적 쿼리 지원**:
    - BooleanBuilder를 사용하여 조건을 동적으로 추가할 수 있습니다.

☑️ 따라서 !!!

- JPQL과 Criteria의 단점을 보완하기 위해 **QueryDSL**이 대안으로 제시됩니다.
- QueryDSL은 **타입 안전성**과 **동적 쿼리 작성의 간결함**을 제공합니다.
- 실무에서는 QueryDSL을 선호하며, 복잡한 동적 쿼리를 쉽게 작성할 수 있습니다.

### **정리**

1. **JPQL 기반**: 간단한 동적 쿼리에는 적합하지만 조건이 많아지면 관리가 어렵습니다.
2. **Criteria 기반**: 표준 스펙이지만 코드 작성이 번거롭고 가독성이 낮아 실무에서 사용이 드뭅니다.
3. **QueryDSL 사용 권장**: 실무에서 동적 쿼리를 작성할 때 가장 효율적이고 간결한 방법입니다.