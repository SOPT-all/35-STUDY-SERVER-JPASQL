# Section 2
## ✅ 회원 가입 API 예제

### saveMemberV1 👎🏻

아래 코드는 @RequestBody로 Member 엔티티를 받아 회원 가입을 진행합니다.

```java
    @PostMapping("/api/v1/members")
    public CreateMemberResponse saveMemberV1(
            @RequestBody @Valid Member member
    ){
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }
```

### ▶️ 문제점

💬 **다양한 회원 가입 방식에 대응하기 어려움**

회원 가입은 아래 처럼 여러 방식으로 이루어질 수 있는데, 

- 기본 가입 (이메일과 비밀번호를 입력하는 일반적인 방식)
- 간편 가입 (소셜 미디어 로그인을 통한 가입, 구글 로그인, 페이스북 로그인 등)

→ 하나의 Member 엔티티로는 각기 다른 가입 방식을 모두 수용하기 어렵습니다. 

(ex) 기본 가입의 경우에는 email 과 password 만 있으면 되지만, 구글 로그인에서는 googleId가 필요할 수 있습니다. 즉, 회원 가입 방식에 따라 필요한 데이터가 다르므로, 모든 경우를 하나의 엔티티에 맞추는 것은 한계가 있습니다.

💬 **엔티티를 외부에 직접 노출하는 구조**

V1 에서는 요청으로 들어오는 Member 엔티티를 그대로 받아 사용하고 있습니다. 

- Member 엔티티는 데이터베이스와 직접 매핑된 객체로, 내부의 비즈니스 로직과 데이터베이스 구조가 외부에 노출되는 위험이 있습니다.
- 엔티티의 변경이 외부 API 응답에 그대로 반영되어 클라이언트 쪽 코드 수정이 불가피합니다.

### ▶️ 솔루션 **: DTO 사용하기**

### saveMemberV2 👍🏻

DTO 를 사용하여 API 요청에 필요한 데이터만을 정의할 수 있습니다.

```java
    @PostMapping("/api/v2/members")
    public CreateMemberResponse saveMemberV2(
            @RequestBody @Valid CreateMemberRequest request
    ){
        Member member = new Member();
        member.setName(request.getName());

        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }
    
    ...
    
    @Data
    static class CreateMemberRequest {
        @NotEmpty
        private String name;
    }
    
    ...
    
    @Data
    static class CreateMemberResponse{
        private Long id;

        public CreateMemberResponse(Long id) {
            this.id = id;
        }
    }
```

💬 **API 스펙 독립성**

- 별도의 DTO를 사용하면, 엔티티가 변경되더라도 API 스펙에는 영향을 미치지 않습니다.
    - (ex) Member 엔티티의 필드를 추가하거나 이름을 변경하더라도 DTO의 구조만 그대로 유지하면 API 응답 형식이 바뀌지 않습니다.

💬 **Command와 Query의 분리**

- CreateMemberRequest는 클라이언트 요청에서 받아오는 데이터를 담는 **Command(조회)용 DTO**
- CreateMemberResponse는 API의 응답으로 반환되는 **Query(변경, 수정)용 DTO**

→ Command와 Query의 **책임을 분리**

→ 각각의 클래스가 명확한 목적을 가지게 됨

→ 코드의 가독성 & 유지보수성이 향상됩니다.

---

## ✅ 회원 조회 API 예제

### ▶️  **membersV1()** 👎🏻

그대로 JSON 배열로 반환합니다.

```java
@GetMapping("api/v1/members")
public List<Member> membersV1(){
    return memberService.findMembers();
}
```

- Member 엔티티가 변경될 경우, API 스펙이 의도치 않게 변할 수 있습니다.
- 단순 배열 형태로 반환하면, 추후에 구조를 확장하기 어려워집니다.

### ▶️  memberV2() 👍🏻

**Result 클래스**로 응답 감싸서 반환합니다. 

```java
    @GetMapping("api/v2/members")
    public Result memberV2(){
        List<Member> findMembers = memberService.findMembers();
        List<MemberDto> collect = findMembers.stream()
                .map(m -> new MemberDto(m.getName()))
                .toList();
        return new Result(collect);
    }
```

- 추후에 추가적인 데이터를 추가하는 등 구조를 확장하기 쉽습니다.

&nbsp;

# Section 3

## ✅ 스프링 빈 생명주기 (Bean LifeCycle)

![image](https://github.com/user-attachments/assets/88e20929-49df-43c3-a542-c926a5445c66)


**스프링 컨테이너 생성  → 빈 생성  →  의존 관계 주입  →  초기화 콜백  →  사용  →  소멸 콜백  →  스프링 종료**

## ✅ 스프링 빈으로 등록되는 객체를 사용하기 위해서는..

1. 객체 생성 후 
2. 의존성 주입까지 끝나야 (생성자,필드, setter 주입 등등..)
3. 해당 객체 사용 가능해짐
    
    : 이 시점에서 스프링은 빈의 초기화 메서드가 있다면 이를 호출하여 초기화 작업을 진행합니다. 
    

## ✅ 스프링 빈 생명주기 콜백 사용 방법

- InitializingBean, DisposableBean 인터페이스 사용
- 설정 정보 초기화 메서드, 종료 메서드 지정
- @**PostConstruct**, **@PreDestroy** 사용 👍🏻

3번째가 스프링에서도 권고하는 방법입니다. 그치만 다른 방법들에 대해서도 한 번 간단하게 살펴보도록 합시당.

### **▶️ InitializingBean, DisposableBean 인터페이스 사용**

```java
public class example implements InitializingBean, DisposableBean {
    @Override
    public void destroy() throws Exception {
        /* 
        스프링 컨테이너가 example 을 생성하고 모든 의존성 주입을 마친 후 
        afterPropertiesSet() 메서드를 호출하여 초기화 작업을 수행합니다. 
        */
    }

    @Override
    public void afterPropertiesSet() throws Exception {
	     /*
	     스프링 컨테이너가 종료되거나 example 이 소멸되기 전에 
	     destroy() 메서드를 호출하여 필요한 정리 작업을 수행합니다.
	     */

    }
}
```

- 간단하고 사용법이 직관적입니다.
- But! 스프링 프레임워크에 의존적이고, 메서드의 이름이 고정됩니다.

### **▶️ 설정 정보 초기화 메서드, 종료 메서드 지정**

1. **설정 클래스에서 외부 라이브러리 빈 등록하기**

```java
@Configuration
public class AppConfig {

	  // 외부 라이브러리 빈 등록
    **@Bean(initMethod = "start", destroyMethod = "close")**
    public ExternalLibrary externalLibrary() {
        return new ExternalLibrary();
    }
}
```

1. **외부 라이브러리 초기화와 종료 처리**

```java
public class ExternalLibrary {
    
    // 초기화 메서드
    public void start() {
        System.out.println("ExternalLibrary 초기화");
    }
    
    // 종료 메서드
    public void close() {
        System.out.println("ExternalLibrary 종료");
    }
    
    // 실제 라이브러리 메서드
    public void doSomething() {
        System.out.println("ExternalLibrary 작업 수행 중...");
    }
}
```

- 메서드 이름을 자유롭게 지정할 수 있습니다.
- 코드를 고칠 수 없는 외부 라이브러리에도 초기화, 종료 메서드를 적용할 수 있습니다. (다음에 더 찾아보기!)

### ▶️ `@PostConstruct` 란?

> 단 한 번만 실행해야 하는 메서드라면, 
`@PostConstruct`와 `@PreDestroy` 어노테이션을 사용하여 간결하고 우아하게 처리할 수 있다 !!
> 

- javax.annotation 패키지에 정의되어 있는 어노테이션 
: 스프링에 종속적인 기술이 아님 → 스프링 아닌 다른 컨테이너에서도 동작함!
- 스프링 컨테이너가 빈을 생성한 후에, @PostConstruct 가 붙은 메서드를 호출하여 초기화 작업을 수행합니다.

### @PostConstruct 사용 방법

```java
import javax.annotation.PostConstruct;

**@Component               ----(2)**
@RequiredArgsConstructor
public class InitDb {
    **@PostConstruct              ----(1)**
    **public** void init() {
		    // 초기화 작업 수행
        initService.dbInit1();
        initService.dbInit2();
    }
    
    // ...
 }
```

1️⃣ **초기화 메서드 작성**

- 빈의 초기화 작업을 수행할 메서드를 정의하고, @PostConstruct 를 달아 줍니다.
- 해당 메서드는 반드시 `public` 으로 선언되어야 합니다.

2️⃣ **빈으로 등록**

- @Component, @Service, @Repository 등의 어노테이션을 사용하여 해당 클래스가 스프링 빈으로 등록되도록 합니다.
- 스프링 컨테이너는 이러한 빈을 관리하면서 @PostConstruct가 붙은 초기화 메서드를 자동으로 호출합니다.

3️⃣ 이를 통해 애플리케이션 실행 시 기본 데이터를 데이터베이스에 자동으로 초기화할 수 있습니다.

**🚨주의🚨**

> **`@PostConstruct` 와 `@PreDestroy` 는 스프링이 관리하는 Bean에만 적용 가능**
> 

**즉, 데이터베이스 연결 라이브러리나 외부 API 라이브러리**에는 직접 이 어노테이션들을 적용할 수 없습니다. 

💥 이 경우 위에서 보았던 `@Bean` 어노테이션을 이용해서 초기화, 소멸 메서드를 지정해야 합니다.
