# SECTION 6, 7


## 디자인 패턴

비즈니스 로직 구현 방식에 따라 **트랜잭션 스크립트 패턴**과 **도메인 모델 패턴**으로 나뉘며, 두 패턴의 차이점은 서비스에 대한 비즈니스 로직의 위치이다.

### 트랜잭션 스크립트 패턴

1. **비즈니스 로직 위치**

   → 엔티티에 비지니스 로직이 거의 없고, 서비스 계층에서 비즈니스 로직을 처리하는 방법이다.

   → Web layer - Service layer - Repository layer로 나뉘는 웹 계층구조에서 서비스 레이어가 모든 비즈니스 로직을 처리하는 모델을 '트랜잭션 스크립트 패턴'이라고 하며, 웹 개발자에게 친숙한 구조다.

2. **특징**

   → 도메인 엔티티는 아무런 비즈니스 로직이 없는 POJO 클래스로 데이터를 전달하는 역할만 수행한다.

   **POJO: 필드와 Getter, Setter와 같은 기본 기능만을 갖는 기본 객체

   → 엔티티는 단순하게 데이터를 전달하는 역할이 되면서 서비스 로직이 커지게 됩니다.

3. 장점

   → 직관적으로 개발할 수 있다.

   → 설계와 구현이 비교적 단순하기 때문에 초기 진입 장벽이 낮다.

4. 단점

   → 코드 중복이 발생하기 쉽다.

   → 로직이 복잡해질수록 서비스 계층이 커지고 유지 보수가 어려워질 수 있다.

5. 적합한 상황

   → 비즈니스 로직이 단순한 경우


### 도메인 모델 패턴

1. 비즈니스 로직

   → 대부분의 비즈니스 로직이 엔티티 안에 구성되어 있어, 데이터 관리와 비즈니스 로직 처리를 함께 수행한다.

   → Web Layered Architecture 에서 도메인 엔티티를 좀 더 객체지향적으로 설계하는 패턴이다.


1. 특징

   → 서비스 계층은 엔티티에 필요한 역할을 위임하는 역할을 한다.

   → 객체지향적 프로그래밍 방식에 더 가깝다.

2. 장점

   → 재사용성이 높다.

   → 비즈니스 로직의 표현이 명확하다.


1. 단점

   → 구현이 복잡하며, 초기 설계 비용이 높다.

2. 적합한 경우

   → 도메인이 복잡한 경우

   → 장기적인 유지보수가 중요한 경우

   > 도메인 엔티티에 생성 메서드를 작성하고, 로직이 변경되는 경우 생성 메서드만 수정하면 된다.


### 결론

객체지향적 관점에서 도메인 모델 패턴이 항상 정답은 아니다.

**트랜잭션 스크립트 패턴**은 간단한 로직과 빠른 개발이 필요한 경우 적합하며, 초기 개발이 쉽고 직관적이다.

**도메인 모델 패턴**은 복잡한 로직과 높은 유지 보수성이 요구되는 프로젝트에 적합하며, 객체 지향 설계를 기반으로 데이터와 로직의 응집도를 높인다.

두 패턴은 서로 상반된 특성과 각각의 장단점이 존재하며 절대적 우위는 없기 때문에, 상황에 맞춰 사용하는 것이 바람직하다.


## JPQL(Java Persistence Query Language)

- 객체지향 쿼리로, JPA가 지원하는 추상화된 쿼리
- SQL과 유사한 문법을 가지지만, 데이터베이스의 **테이블이 아닌 엔티티(Entity)를 대상으로 쿼리를 작성**한다는 점에서 차이가 있다.

> JPQL vs SQL
*SQL: 테이블 대상으로 쿼리
*JPQL: 엔티티 객체를 대상으로 쿼리
>
- JPQL은 결국 SQL로 변환된다.
- JPA에서 제공하는 메소드 호출만으로 섬세한 쿼리 작성이 어렵다는 문제에서 JPQL이 탄생되었다.

**JPQL**은 `객체`를 기준으로 모든 것이 움직이기 때문에(`객체중심적`) 개발할 때, `Table에 매핑되는 객체`가 반드시 존재해야 하며 당연하게도 검색할 때도 `Table`이 아닌 `객체`를 대상으로 검색해야 한다.

### JPQL 특징

```java
String jpql = "select m from Member as m where m.name = 'jpa1'";
```

1. 대소문자 구분
    - 엔티티와 속성은 대소문자를 구분한다.
    - 엔티티 이름인 Member, 그리고 Member의 속성 name은 대소문자를 구분해줘야 한다.
    - SELECT, FROM, AS 같은 JPQL 키워드는 대소문자를 구분하지 않아도 된다.

2. 엔티티 이름
    - Member는 클래스 이름이 아닌 엔티티 이름이다.
        - 엔티티 이름 설정 → @Entity(name=”abcd”)
    - name 속성을 생략하면 기본 값으로 클래스 이름 사용한다.

3. 별칭
    - JPQL에서 엔티티의 별칭은 필수적으로 명시해야 한다.

4. from절에 들어가는 것은 객체이다.

### 조회 결과 반환

- query.getResultList()
    - 결과가 하나 이상인 경우, 리스트 반환
- query.getSingleResult()
    - 결과가 정확히 하나, 단일 객체 반환(하나가 아니면 예외 발생)

### TypedQuery vs Query

두 가지는 crateQuery method 반환 결과를 다룰때 사용하는 타입으로, 쿼리의 반환 타입이 명확한 경우 TypeQuery<type> 형태로 사용하고, 명확하지 않은 경우 Query로 반환 타입을 명시한다.

```java
TypedQuery<Member> t_query = em.createQuery("select m from Member m", Member.class);
Query query = em.createQuery("select m.username from Member m");
```

- TypedQuery - em.createQuery 메소드를 호출할 때 두 번째 인자로 엔티티 클래스를 넘겨준다.
- Query - 데이터 검색 결과의 타입을 명시하지 않는다.

### 파라미터 바인딩

파라미터 바인딩에는 이름 기준 파라미터와 위치 기준 파라미터가 있다. 위치 기준 파라미터 보다 이름 기준 파라미터가 더 명확하다.

1. 이름 기준 파라미터

```java
public static void namedParameter(EntityManager em, String param) {
	String jpql = "select m from Member m where m.name = :name";
	TypedQuery<Member> query = em.createQuery(jpql, Book.class);	
	query.setParameter("name", param);		
	
	List<Member> list = query.getResultList();
	}
```

콜론( : ) 을 사용해 데이터가 추가될 곳을 지정하고,query.setParameter() 메소드를 호출해 데이터를 동적으로 바인딩 한다.

1. 위치 기준 파라미터

```java
public static void locationalParameter(EntityManager em, String param) {
    String jpql = "select m from Member m where m.name = ?1";
    List<Member> members = em.createQuery(jpql, Member.class)
    .setParameter(1, param)
    .getResultList();
    }
```

? 다음에 위치 값을 주면 된다. 위치 값은 1부터 시작한다.

### JPQL의 문제점

1. JPQL은 문자열(String) 형태 → 개발자 의존적 형태
2. Compile 단계에서 Type-Check 불가능
3. Runtime 단계에서 오류 발견(장애 risk 상승)

> JPQL 문자열에 오타 혹은 문법적인 오류가 존재하는 경우, 정적 쿼리라면 어플리케이션 로딩 시점에 이를 발견할 수 있으나 그 외는 런타임 시점에서 에러가 발생합니다.

## query DSL

JPQL에 대한 보완을 위해 나온 것

→ 정적 타입을 이용해서 SQL, JPQL을 코드로 작성할 수 있도록 도와주는 오픈소스 빌더 API

**query DSL의 목적

기존 방식(Mybatis, JPQL, etc..)은 모두 `문자열(=String) 형태`로 쿼리가 작성되었고 이로 인해 `Compile 단계에서 Type-Check 불가`했다.

이러한 risk를 줄이기 위해 **query DSL**이 등장했고 이를 통해 `Compile 단계에서 Type-check가 가능`해진 것이다.

**query DSL**은 모든 쿼리에 대한 내용이 `함수 형태`로 제공된다.

이렇게 `엔티티(객체) + 함수`형태로 구성된 **query DSL**을 통해 구현된 코드는 `오류가 존재할 시, Compile 단계에서 바로 확인 가능`하며 이에 따른 후속 조치가 가능하기 때문에 그만큼 `risk가 줄어들게`되는 것이다.

### query DSL 특징

1. 타입 안정성 (Type Safety)
    - 쿼리를 작성할 때 컴파일 시점에 필드와 조건의 타입을 확인한다.
    - 런타임 에러 대신 컴파일 타임에 오류를 잡을 수 있어 안정성이 높다.
2. 가독성
    - 메서드 체이닝 방식을 사용하여 쿼리가 간결하고 직관적이다.
    - JPQL이나 Criteria API에 비해 코드가 간단하고 읽기 쉽다.
3. 동적 쿼리 구현 가능
    - QueryDSL은 조건에 따라 동적으로 쿼리를 생성하는 데 강력한 기능을 제공한다.
    - 이를 통해 코드 중복 없이 복잡한 조건의 쿼리를 처리할 수 있다.