## 애플리케이션 아키텍처

![스크린샷 2024-11-19 184407.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8082c7c0-79c7-4547-ba2b-78142b6ffb42/f1cd7d32-ae01-4f5a-a4da-c2b12ba1d867/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7_2024-11-19_184407.png)

- **controller ,web** : 웹계층
- **service** : 비즈니스 로직, 트랜잭션 처리
- **repository** : JPA를 직접 사용하는 계층, 엔티티 매니저 사용
- **domain** : 엔티티가 모여 있는 계층, 모든 계층에서 사용

### 패키지 구조

- **domain**
- **exception**
- **repository**
- **sevice**
- **web**

---

## 트랜잭션

<aside>
💡

데이터베이스에서 **데이터를 일관성 있게 관리하기 위한 논리적 작업 단위**이다. 하나의 트랜잭션은 여러 작업(쿼리, 연산 등)을 포함할 수 있으며 이 작업들은 **모두 실패하거나 성공**해야한다

</aside>

## ACID원칙

### 1️⃣Atomicity(원자성)

<aside>
💡

**모든 작업이 전부 수행되거나 전혀 수행되지 않아야한다. 일부 작업만 완료되고 나머지가 실패하면 트랜잭션은 롤백되어야 한다.**

</aside>

### 2️⃣Consistency(일관성)

<aside>
💡

**트랜잭션이 완료되면 데이터베이스는 일관된 상태를 유지해야한다**

</aside>

### 3️⃣Isolation(격리성)

<aside>
💡

**동시에 실행되는 트랜잭션이 서로 영향을 미치지 않아야한다**

</aside>

### 4️⃣Durabillity(내구성)

<aside>
💡

**트랜잭션이 커밋된 후에는 데이터가 영구적으로 반영된다.**

</aside>

### 트랜잭션의 동작 흐름

1. 트랜잭션 시작
2. 여러 SQL 쿼리 실행
3. 트랜잭션이 정상적으로 완료되면 변경사항을 영구적으로 반영(**커밋**)
4. 중간에 오류가 발생하면 변경 사항을 모두 취소하고 이전 상태로 되돌림(**롤백**)

**@Transactional**

`Transactional`은 Spring Framework에서 제공하는 애너테이션으로, **트랜잭션을 선언적으로 관리**할 수 있도록 지원한다

데이터베이스 연산(예: INSERT, UPDATE, DELETE 등)이 모두 성공하거나, 중간에 문제가 발생할 경우 변경사항을 되돌릴 수 있게 해준다

### 1️⃣**주요 기능**

1. **트랜잭션 시작**: 메서드 실행 시 트랜잭션을 자동으로 시작한다
2. **성공 시 커밋**: 메서드가 정상적으로 완료되면 트랜잭션을 커밋(Commit)하여 변경 사항을 반영한다
3. **실패 시 롤백**: 메서드 실행 중 예외가 발생하면 트랜잭션을 롤백(Rollback)하여 변경 사항을 되돌린다.

### 2️⃣**적용 위치**

- **클래스 레벨**: 해당 클래스의 모든 메서드에 트랜잭션 적용.

    ```java
    @Transactional
    public class UserService {
    
    }
    
    ```

- **메서드 레벨**: 특정 메서드에만 트랜잭션 적용.

    ```java
    public class UserService {
        @Transactional
        public void createUser(User user) {
        }
    }
    
    ```


> **메서드 레벨 선언이 클래스 레벨 선언보다 우선 적용된다**
>

### 3️⃣**@Transactional 주요 속성**

1. **`propagation` (전파 방식)**

   →트랜잭션을 다른 트랜잭션과 어떻게 연계할지 정의.

   기본값: `Propagation.REQUIRED`

    - **`REQUIRED` (기본값)**: 이미 트랜잭션이 있으면 참여하고, 없으면 새로 시작.
    - **`REQUIRES_NEW`**: 항상 새 트랜잭션을 시작하고 기존 트랜잭션은 일시

    ```java
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveData() {
        // 항상 새로운 트랜잭션 시작
    }
    
    ```

2. **`isolation` (격리 수준)**

   트랜잭션 간 데이터 읽기/쓰기를 격리하는 방법 설정.

   기본값: `Isolation.DEFAULT` (DB 기본값 사용

    ```java
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void processData() {
        // 가장 높은 격리 수준 적용
    }
    
    ```

3. **`readOnly`**

   읽기 전용 트랜잭션 여부를 설정.

   `true`로 설정하면 데이터 변경 시 오류를 발생시키거나 최적화를 적용.

    ```java
    
    @Transactional(readOnly = true)
    public List<User> getUsers() {
        // 읽기 전용 트랜잭션
    }
    
    ```

4. **`timeout`**

   트랜잭션 실행 제한 시간(초)을 설정. 초과하면 롤백.

    ```java
    @Transactional(timeout = 5)
    public void saveLargeData() {
        // 5초 초과 시 롤백
    }
    
    ```


---

## 의존관계 주입

<aside>
💡

**의존관계주입(DI)란 의존관계를 외부에서 결정(주입)하여 객체를 생성하고 사용하는 것이다**

</aside>

💭객체가 스스로 의존하는 객체를 생성하거나 관리하지 않고 외부에서 필요한 객체를 전달받음으로써 객체 간 결합도를 낮추고 코드의 재사용성과 테스트 용이성을 높인다.

### 의존관계 주입이 필요한이유?

1. **결합도 감소**
    1. 객체가 스스로 의존 객체를 생성하면 해당 객체에 강하게 결합되므로 유지보수가 어렵다
    2. 외부에서 객체를 주입받으면 유연하게 의존성을 변경할 수 있다.
2. **테스트 용이성**
    1. 외부에서 객체를 주입받기 때문에 테스트 시에 Mock 객체를 쉽게 주입할 수 있어 테스트 작성이 편하다

### 의존관계 주입방식

### 1️⃣생성자주입

- 의존 객체를 생성자를 통해 주입하는 방식이다
- 객체가 생성될 때 모든 의존성이 주입되므로 불변성 보장에 유리하다

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public void saveUser(User user) {
        userRepository.save(user);
    }
}
```

### 2️⃣필드 주입

- 의존 객체를 필드에 직접 주입하는 방식
- 간결하지만 테스트 용이성과 캡슐화 측면에서 단점이 많다

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public void saveUser(User user) {
        userRepository.save(user);
    }
}
```

### 예시-피자가게

<aside>
❓

피자 가게에서 요리사는 피자 레시피에 의존을 하고 있다. 만약 피자 레시피가 변경되면다면 요리사는 피자를 새로운 방법으로 만들게 된다.

</aside>

```java
public class PizzaChef{
	
	private PizzaRecipe pizzaRecipe;
	
	public PizzaChef() {
		this.pizzaRecipe = new PizzaRecipe();
	}
}
```

💭위의 코드인 경우 `PizzaChef` 클래스와 `PizzaRecipe`  클래스 간에 강하게 결합되어 있다는 것을 알 수 있다. 만약 새로운 레시피를 도입하게 된다면 `PizzaChef`의  생성자를 매번 바꿔줘야만한다.

💭객체들 간의 관계가 아닌 클래스 간의 관계가 맺어진다. 객체지향 5원칙(SOLID) 중 DIP원칙, 구체화에 의존하는 것이 아닌 추상화에 의존해야한다는 원칙을 어기게 된다.

```java
public interface PizzaRecipe{
	
}

public class CheesePizzaRecipe implements PizzaRecipe{
	
}
```

```java
public class PizzaChef{
	
	private PizzaRecipe pizzaRecipe;
	
	public PizzaChef(PizzaRecipe pizzaRecipe) {
		this.pizzaRecipe = pizzaRecipe;
	}
	
}
```

💡이렇게 된다면 피자 셰프는 피자 레시피가 바뀌더라도 생성자를 변경하지 않아도 된다. 그저 레시피가 바뀐다면 외부에서 바뀐 레시피를 주입해주기만 하면 된다.

---

## 2주차 어노테이션정리

- `@Repository` : 스프링 빈으로 등록, JPA 예외를 스프링 기반 예외로 예외 변환
- `@PersistenceContext` : 엔티티매니저 주입
- `PersistenceUnit` : 앤티티 매니저 팩토리 주입
- `@Service` : 이 클래스가 서비스 레이어임을 명시한다. 스프링의 컴포넌트 스캔대상이 되어 빈으로 등록됨.
- `@RequiredArgsConstructor` : 클래스 내에서 **final**로 선언된 필드나 @NonNull로 표시된 필드를 포함하는 생성자를 자동으로 생성
- `@Autowired` : 해당하는 빈을 찾아 클래스의 필드, 생성자에 의존성을 주입한다
    - **필드주입**

    ```java
    @Autowired
    private MyService myService;
    ```

    - **생성자 주입 → 생성자가 하나일 경우 `@Autowired` 생략가능**

    ```java
    private final MyService myService;
    
    @Autowired
    public MyClass(MyService myService) {
        this.myService = myService;
    }
    ```

    - 생성자 주입 vs 필드 주입

    <aside>
    💡

  생성자 주입으로 초기화된 필드는 `final`로 선언할 수 있어 **객체가 생성된 후 상태 변경을 막으**며 테스트에서 **의존성 주입을 명시적**으로 할 수 있다.

    </aside>