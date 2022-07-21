# JPQL의 fetch join과 일반 join의 차이가 무엇일까?

<br>

### 한 줄 정리 : SELECT를 통해 영속화하는 엔티티 대상의 차이

<br>

## 📌 fetch join과 일반 join의 차이
- 일반 join
```java
//JPQL 일반 join 예시
select t
from Team t join t.members m
where t.name = '팀A'

//실제 실행 SQL
SELECT T.*
FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'
```

- fetch join
```java
//JPQL 컬렉션 페치 조인 예시
select t
from Team t join fetch t.members
where t.name = '팀A'

//실제 실행 SQL
SELECT T.*, M.*
FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'
```

- 일반 join을 사용하면, 위의 fetch join과 달리 실제 실행 SQL에서도 SELECT T.*로 대상 엔티티만 조회하는 것을 확인할 수 있다.
- fetch join을 사용하지 않은 경우, JPQL은 기본으로 결과를 반환할 때 연관관계까지 고려하지 않고, 단지 SELECT 절에 지정한 엔티티만 조회한다. 따라서 위의 예시에서 팀 엔티티만 조회하고 연관된 회원 컬렉션은 조회하지 않는다.
    - 만약 회원 컬렉션을 Lazy 로딩으로 설정했다면, 위의 일반 join 예시에서 조회했을 때 Team의 members는 프록시나 아직 초기화하지 않은 컬렉션 래퍼를 반환한다.
    - 만약 회원 컬렉션을 Eager 로딩으로 설정했다면, 위의 일반 join 예시 쿼리가 1번 실행되고, 추가적으로 회원 컬렉션을 조회하기 위해 쿼리가 한 번 더 실행한다. → 이때가 바로 N+1이 발생할 수 있는 상황이 되는 것이다.

<br>

## 📌 fetch join의 특징과 한계
### ▶️ 특징
- SQL 한 번으로 연관된 엔티티들을 함께 조회할 수 있어 SQL 호출 횟수를 줄여 성능을 최적화할 수 있다.
- 어플리케이션 전체에 영향을 미치는 글로벌 로딩 전략보다 우선된다.
    - 글로벌 로딩 전략을 Lazy로 설정해도 fetch join을 사용하면, fetch join을 적용해 한 번에 조회한다.
    - 글로벌 로딩 전략은 Lazy로 설정하고, 최적화가 필요하면 fetch join을 적용하는 구성이 효과적이다.
- fetch join은 객체 그래프를 유지할 때 사용하면 효과적이다. 반면에 여러 테이블을 조인해서 기존의 엔티티가 가진 모양이 아닌 전혀 다른 결과를 내야 한다면, 억지로 fetch join을 사용하는 것보다 여러 테이블에서 필요한 필드들만 SELECT 절에 명시해서 DTO로 반환하는 것이 더 효과적이다.

<br>

### ▶️ 한계
- fetch join 대상에 별칭을 줄 수 없다.
    - SELECT, WHERE 절, 서브 쿼리에 fetch join 대상을 사용할 수 없다.
- 둘 이상의 컬렉션을 fetch join 할 수 없다.
- 컬렉션을 fetch join하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다. (엔티티 fetch join은 제외)
    - 데이터베이스에서 페이징 처리해서 가져오는 것이 아니라 일단 메모리에 데이터를 모두 올린 후 메모리에서 페이징 처리를 수행한다.
    - 데이터가 적으면 상관없겠지만, 데이터가 많아지면 성능 이슈와 메모리 초과 예외가 발생할 수 있어 위험하다.

<br>

### 🌈 참고
- 인프런 JPA 기본편 강의 & 책
- [https://cobbybb.tistory.com/18](https://cobbybb.tistory.com/18)