# JPA의 JPQL 사용시 검색 조건의 값이 null인 경우에 대한 처리

<br>

## 📌 상황

```java
@Query("select p.id as id, " +
        "from AffiliatedPharmacy ap " +
        "inner join ap.pharmacy p " +
        "where ap.affiliatedInstitution = :institution " +
        "and p.pharmacyName like %:pharmacyName%")
Page<PharmApiSearchResDtoVo> findByCity(
	@Param("pharmacyName") String pharmacyName, 
	@Param("institution") AffiliatedInstitution institution, 
	Pageable pageable
);
```

- 검색 조건이 Optional인 경우 Request Body에 값을 전달하지 않고 별도의 기본 값 처리를 하지 않는다면 null 값이 전달되고, 위와 같이 JPQL의 Parameter로 매핑될 때 null이 매핑되어 where 절에 포함이 된다.
- 원래의 의도는 값이 전달되지 않았을 때는 해당 값은 검색 조건에 포함시키지 않아야 하므로 추가 처리를 해주지 않으면 의도대로 검색결과가 나오지 않게 된다.

<br>

## 📌 개선
```java
@Query("select p.id as id, " +
        "from AffiliatedPharmacy ap " +
        "inner join ap.pharmacy p " +
        "where ap.affiliatedInstitution = :institution " +
        "and (:pharmacyName is null or p.pharmacyName like %:pharmacyName%")
Page<PharmApiSearchResDtoVo> findByCity(
	@Param("pharmacyName") String pharmacyName, 
	@Param("institution") AffiliatedInstitution institution, 
	Pageable pageable
);
```

- 위와 같이 where절에 `is null` 을 추가해 Parameter의 값이 Null이면 () 안의 문장이 항상 참이 되도록 하여 해당 where 조건이 포함되지 않게 한다.
- Querydsl의 BooleanExpression을 통한 처리와 동일하다.