# JPA 상속 관계 매핑 Join 전략 사용시 주의할 점 - 컬럼명 동일 여부에 따른 insert와 update 쿼리 실행 확인

<br>

## 📌 상황

- 해당 작업의 DB 테이블 설계는 대략 아래 사진과 같고, JPA 상속 관계 매핑 3가지 방식 중 Join 전략을 사용했다.

![image](https://user-images.githubusercontent.com/69254943/179784511-d29749ff-f258-43e5-abd8-beb501ed6425.png)

- 이때 이력을 남기기 위해 상위 엔티티인 ApiUser에만 생성자, 생성일시, 최종수정자, 최종수정일시 컬럼을 추가했다. (JPA Auditing 기능을 사용했다)
- 그런데 팀장님께서 ‘근데 하위 엔티티의 필드 값만 추가하거나 변경하면 상위 엔티티의 이력 컬럼에 값이 제대로 저장되나요?’ 라는 질문을 하셨고 정확한 근거가 없어 바로 테스트를 해보았다. 만약 하위 엔티티 필드 값만 추가하거나 변경했을 때 상위 엔티티의 위의 4가지 필드(생성자, 생성일시, 최종수정자, 최종수정일시) 값에 반영이 되지 않는다면 이력이 제대로 남지 않는 문제가 발생하기 때문이었다.

<br>

## 📌 확인

### ▶️ 결론부터 말하자면
- JPA 상속 관계 매핑에 Join 전략을 사용하면, 하위클래스 insert와 update할 때 무조건 상위 클래스에도 insert와 update 쿼리가 수행된다. 즉, 하위 엔티티의 필드 값만 추가하거나 변경해도 상위 엔티티의 이력 컬럼에 값이 제대로 저장된다고 팀장님의 질문에 답변을 할 수 있다! 
- 수정 기록을 남기기 위해 하위 클래스에도 해당 컬럼들을 추가하는 것은 중복만 있을 뿐 불필요한 코드가 된다. (하위클래스를 변경하면 무조건 상위 클래스에도 변경 쿼리가 나가니까)
- 그럼에도 상위, 하위 클래스에 동일한 역할의 컬럼을 추가하고자 한다면 컬럼명을 다르게 해야 한다.

### ▶️ 테스트 결과
- 위의 DB 설계를 바탕으로 테스트 결과를 작성하였다. 
- 여기서의 컬럼은 ‘생성자, 생성일시, 최종수정자, 최종수정일시’를 말한다.

### 상황1 - 상위, 하위 클래스의 컬럼명이 동일할 때

<img width="583" alt="image" src="https://user-images.githubusercontent.com/69254943/179786424-9a698501-6fa1-4365-8dc3-c4f95fb652ee.png">

- 상위 클래스 - ApiUser, 하위 클래스 - ApiBizUser
- ApiBizUser 엔티티 save() → insert ApiUser의 컬럼만
    - 하위 엔티티를 저장하면, 상위 테이블의 컬럼에만 값이 들어가고, 하위 테이블의 컬럼에는 값이 저장되지 않는다.
- ApiBizUser 엔티티 update() → update ApiUser의 컬럼만
    - save()와 동일하다.
- ApiUser 엔티티 findById() → ApiBizUser의 컬럼에 ApiUser의 컬럼 값이 매핑되어 반환
    - 이때 ApiUser의 컬럼은 null로 반환된다.
- ApiBizUser 엔티티 findById() → ApiBizUser의 컬럼에 ApiUser의 컬럼 값이 매핑되어 반환
    - 위와 동일하다.

### 상황2 - 상위, 하위 클래스의 컬럼명이 다를 때

<img width="585" alt="image" src="https://user-images.githubusercontent.com/69254943/179786761-4aa84eae-9db6-40b3-beb0-330aad69ec16.png">

- 상위 클래스 - ApiUser, 하위 클래스 - ApiBizUser
- ApiBizUser 엔티티 save() → insert ApiUser, insert ApiBizUser
    - 컬럼명이 동일하지 않아 상위, 하위 클래스의 컬럼 모두에 값이 저장된다.
- ApiBizUser 엔티티 update() → update ApiUser, update ApiBizUser
    - save()와 동일하다.
- ApiUser 엔티티 findById() → ApiUser와 ApiBizUser 컬럼 값 모두 반환
- ApiBIzUser 엔티티 findById() → ApiUser와 ApiBizUser 컬럼 값 모두 반환