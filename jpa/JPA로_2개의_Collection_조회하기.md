# JPA로 대상 엔티티와 연관된 2개의 Collection 조회하는 방법

<br>

### 간단 패턴화
- @EntityGraph or Fetch Join + Hibernate의 default_batch_fetch_size 옵션 사용

<br>

## 📌 상황
- 회사에서 조회 기능을 구현하며 특정 엔티티 조회 시 해당 엔티티와 @OneToMany로 연관되어 있는 2개의 Collection을 함께 가져오기 위해 추가 쿼리가 2번이나 발생하는 상황을 마주하게 되었다. 
- @OneToMany의 FetchType은 기본이 LAZY이므로, 해당 엔티티를 조회할 때 연관된 Collection은 한번에 가져오지 않기 때문이었다.
- 현재는 조회 대상 엔티티를 1개만 가져오는 것이므로 N + 1 상황은 아니지만, 추후 조회 대상 엔티티가 1개 이상이 되면 N + 1 상황이 되는 것이고, 현재 상황에서도 쿼리 실행 횟수를 줄일 방안이 있을 것이라고 생각하여 개선하게 되었다.

```java
//예시 코드
@Entity
public class A {
	...
	@OneToMany(mappedBy = "a")
	private List<B> b = new ArrayList<>();
	@OneToMany(mappedBy = "a")
	private List<C> c = new ArrayList<>();
	...
}
```

<br>

## 📌 개선
- 개선 방안은 '현재 상황(조회 대상 엔티티가 1개일 때)에서의 개선 방안'과 'N + 1 상황에서의 개선 방안'으로 나누어 정리할 수 있다.

<br>

### ▶️ 현재 상황(조회 대상 엔티티가 1개일 때) 에서의 개선 방안

<br>

### @EntityGraph or Fetch Join 사용하기
- JPA에서는 Collection 조회시 @EntityGraph와 Fetch Join을 통해 연관 엔티티들을 한 번의 쿼리로 가져올 수 있다.
- 회사에서는 @EntityGraph를 사용하여 기존 3번 → 2번으로 쿼리 실행 횟수를 줄였다. 

<br>

적용 예시 코드는 아래와 같고, 엔티티 간의 관계는 PharmacyRole 대상 엔티티가 PharmacyRoleMenu와 PharmacyEmployeeRole Collection을 가지고 있는 상태이다.

1. 첫 번째 실행 쿼리
```java
// Repository 코드 예시
@EntityGraph(attributePaths = {"pharmacyRoleMenus"})
@Query("select distinct pr " +
       "from PharmacyRole pr " +
       "where pr.id = :pharmacyRoleId")
Optional<PharmacyRole> findRoleDetailByIdWithEntity(@Param("pharmacyRoleId") Long pharmacyRoleId);
```
- @EntityGraph를 사용해 연관된 Collection 엔티티를 즉시 로딩하여 한번의 쿼리로 같이 가져온다.
- 이때, 연관 컬렉션의 원소 개수만큼 레코드 수가 늘어나므로, 대상 엔티티 중복 제거를 위해 distinct를 사용해야 한다.

```sql
// 실행 쿼리
select
        pharmacyro0_.id as id1_124_0_,
        ...
from
    pharmacy_role pharmacyro0_ 
left outer join
    pharmacy_role_menu pharmacyro1_ 
        on pharmacyro0_.id=pharmacyro1_.pharmacy_role_id 
where
    pharmacyro0_.id=?
```

- left outer join을 사용한다.
- fetch join의 경우에는 inner join을 사용한다.

<br>

2. 두 번째 실행 쿼리
- @EntityGraph와 Fetch Join은 2개의 Collection에 대해서는 적용할 수 없다. 만약 적용한다면, Multi-Bag-Exception이 발생한다.
- 따라서 남은 1개의 Collection에 대해서는 추가 쿼리가 실행된다.

<br>

### ▶️ N + 1 상황에서의 개선 방안

<br>

### @EntityGraph or Fetch Join + Hibernate의 default_batch_fetch_size 옵션 사용하기
- 앞서 설명한 현재 나의 개발 상황에서는 하나의 대상 엔티티를 조회하는 것이므로, 사실 N = 1이 되어 1(@EntityGraph를 사용해 대상 엔티티 + 연관 Collection 한번에 조회) + 1(연관 Collection 조회) 쿼리가 수행된다.
- 하지만 여러 엔티티를 조회하는 상황에서는 위와 같이 개선하여도 나머지 Collection 조회를 위한 추가 쿼리가 실행되므로, N + 1이 유지된다.
- 해당 상황에서는 ```Hibernate의 default_batch_fetch_size 옵션```을 설정해 추가 쿼리 실행 횟수를 줄일 수 있다.
  - 옵션으로 설정한 개수만큼 WHERE IN절로 한 번에 조회해온다.
