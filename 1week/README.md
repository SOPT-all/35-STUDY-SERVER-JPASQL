## 해당 강의 진행시 h2 db 서버를 키고 진행

## 1섹션

## 2섹션

### 회원 등록 api

- 엔티티를 직접 받아오는 것은 최대한 지양해야함! → 추후 api 스펙이 변경되었을때 많이 곤란해짐
    - 엔티티랑 api 랑 1대1로 매핑이 되버림
    - api 스펙을 위한 dto를 만드는것이 좋음(혹여나 스펙이 변경되면 엔티티를 바꿔야하나?) 그건 아님!
    - 엔티티 하나 만으로 ex) 여러 등록 api 대응 가능? 안됨! 힘듦! 엔티티를 외부에 최대한 노출하면 안됨!
- presentation 계층에서의 valid 를 거쳐야 할 것이 이후에
- v1, v2 버전 차이 dto 로 받거나 엔티티로 받거나
    - dto로 받는 것의 단점은 그냥 클래스가 하나 더 생긴다는 점임!
    - dto로 하는것이 조금 더 괜찮은 방식
    - dto로 안 받고 엔티티로 받으면 어떤 값이 넘어오나? 이게 감이 안 잡힘!

### 회원 수정 api

- 수정이 일어날때는 변경감지(dirty checking)을 활용해보면 좋음
    - @transactional 끝나고 flush, commit 진행

### 회원 조회 api

- 밖으로 꺼낼때 회원의 엔티티를 직접 노출시키면 안된다.
- 이렇게 되면 프레젠테이션 계층을 위한 로직이 추가된다.
- 한 엔티티에 각각 api 를 위한 pres 응답 로직 담기 어려움
- 엔티티 변경되면 api 스펙도 다같이 변경이 되어야 함.
- array 로 넘어왔는데 갑자기 스펙 변경 사항에  넣어야 함

```java
"count":4
"data": [

]
```

- 위와 같은 형태로 할 수가 없음 엔티티로 뱉어버리면
- 엔티티가 변경이 되어도 api 스펙이 변하지 않는다는 것이 좋아짐 v2 로 변경되면서

## 3섹션(API 개발 고급)

### 조회용 샘플 데이터 입력

- `postconstruct` 활용 -> 
  - spring 공식문서에서 `InitializaingBean` 인터페이스를 활용해서 하지 말라고 함! -> 불필요하게 Spring 에 결합시키기 때문 
  - Spring 이 해당 어노테이션이 달린 경우 빈 속성 초기화 직후 한 번 호출
  - `predestroy` : 빈이 제거되기 직전에 한 번 호출되는 친구

> JSR-250 @PostConstruct과 @PreDestroy주석은 일반적으로 최신 Spring 애플리케이션에서 라이프사이클 콜백을 수신하기 위한 모범 사례로 간주됩니다. 이러한 주석을 사용하면 빈이 Spring 특정 인터페이스에 결합되지 않습니다. 자세한 내용은 및 사용을@PostConstruct@PreDestroy 참조하세요 .


### 번외 : InitializingBean, DisposableBean 사용법 (Spring에서 제공하는 빈 초기화 시점 활용방법)

```java
public class AnotherExampleBean implements InitializingBean {

	@Override
	public void afterPropertiesSet() {
		// do some initialization work
	}
}


public class AnotherExampleBean implements DisposableBean {

    @Override
    public void destroy() {
        // do some destruction work (like releasing pooled connections)
    }
}
```

- init DB 를 활용하여 만들기