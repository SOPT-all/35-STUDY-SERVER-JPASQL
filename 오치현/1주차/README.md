> 작성 : 오치현<br>
일시 : 2024. 11. 11.<br>
강의 : JPA 활용 2 - 섹션 1 ~ 3
>

# 1. API 개발 기본

API 개발 시, 파라미터로 전송되는 데이터를 엔티티로 받지 말고, 엔티티를 반환하지도 말아야 한다.

→ 엔티티 정보 수정 시 API 스펙이 변해 기존 서비스가 동작하지 않을 수 있음

<br>

@RequestBody : HTTP Body로 전송되는 JSON 데이터를 엔티티에 매핑해줌

@Data : @Getter, @Setter, @RequiredArgConstructor, @EqualsAndHash…, @ToString이 포함되어있음

@Valid : 제약사항을 검사해줌

<br>

[API 안 좋은 예]

```java
@PostMapping("/api/v1/members")
public CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member) {
	Long id = memberService.join(member);
	return new CreateMemberResponse(id);
}
```

→ 넘어온 파라미터를 엔티티에 매핑 시키는 방법

Member가 변하면 API의 스펙이 변경된다. 

<br>

[API 좋은 예]

```java
@PostMapping("/api/v2/members")
public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {
	
	Member member = new Member();
	member.setName(request.getName());
	
	Long id = memberService.join(member);
	return new CreateMemberResponse(id);
}
```

→ 별도의 DTO를 생성해서 받는 방법

장점1. ember가 변해도 엔티티 생성 시 값을 직접 전달하기 때문에 API 스펙이 변하지 않는다.

장점2. API에서 넘어온 파라미터 값이 어느 값으로 매핑되는지 한 눈에 확인할 수 있다.

장점3. 작성된 API에 핏하게 값이 설정된다.

장점4. 엔티티와 프레젠테이션을 명확하게 분리

<br>

PUT 메서드는 멱등하다. → 같은 것을 여러 번 호출해도 변하지 않는다!

<br>

커맨드와 쿼리를 분리!

- 어떤 기준일까?
    
    영상에 직접적으로 나오지는 않았지만, 서비스 계층에서 처리하는 과정에서 데이터베이스 질의의 결과(row)가 외부로 노출될 때 `쿼리` 라고 부르는 것 같다.
    
<br>

엔티티를 그대로 노출할 때, 보여주기 싫은 데이터는 숨기면 되는거 아닐까?

- @JsonIgnore로 해당 데이터가 외부로 노출되는 것을 막을 수 있다.
    
    하지만, 여러 곳에 적용이 불가능해 곤란하다. 어떤 응답에는 값을 포함하고, 어떤 응답에는 값을 포함하지 않아야 되는 경우가 생김.
    그리고 무엇보다 엔티티에 프레젠테이션을 위한 기능이 추가됨.
    
<br>

조회 시, 응답 값으로 엔티티를 외부에 직접 노출한 경우

- 문제점
    1. 엔티티에 프레젠테이션 계층을 위한 로직이 추가됨.
    2. 기본적으로 엔티티의 모든 값이 노출됨.
    3. 응답 스펙을 맞추기 위한 로직이 추가됨. (@JsonIgnore, 뷰 로직 등등)
    4. 하나의 엔티티가 여러 응답 스펙을 맞추기 어려움.
    5. 엔티티 변경 → API 변경
    6. 컬렉션을 반환하면 이후 API 스펙을 변경하기 어려움. (스펙이 굳어버림)
        
        → 반드시 오브젝트로 감싸서 반환해야 함!!
        

→ `DTO` 쓰면 해결~!
