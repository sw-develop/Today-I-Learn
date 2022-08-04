# JpaPagingItemReader의 createQuery() 사용 방안

<br>

## 📌 상황
- JpaPagingItemReader 클래스의 createQuery() 정의 부분을 보면, 아래와 같이 2가지 방식으로 수행이 된다.

```java
//JpaPagingItemReader 클래스

/**
* Create a query using an appropriate query provider (entityManager OR
* queryProvider).
*/
private Query createQuery() {
    if (queryProvider == null) {
		return entityManager.createQuery(queryString);
    }
    else {
        return queryProvider.createQuery();
    }
}
```

<br>

## 📌 적용

<br>

### ▶️ 방식1 - queryString으로 바로 전달

```java
@Bean
@StepScope
public JpaPagingItemReader<Object[]> bizRegFileDeleteReader(
        @Value("#{jobParameters[currentDate]}") String currentDate
) {

    HashMap<String, Object> parameterValues = new HashMap<>();
    parameterValues.put("expiredDate", LocalDateTime.now().minusMonths(1));
    
    return new JpaPagingItemReaderBuilder<Object[]>()
        .name("bizRegFileDeleteReader")
        .entityManagerFactory(entityManagerFactory)
        .pageSize(chunkSize)
        .queryString(
            "SELECT p.id, p.bizRegFilePath FROM Pharmacy p WHERE p.createdDate <= :expiredDate and p.bizRegFilePath is not NULL ORDER BY p.id"
        ) //queryString 지정
        .parameterValues(parameterValues)
        .build();
}
```

<br>

### ▶️ 방식2 - AbstractJpaQueryProvider를 상속한 CustomQueryProvider 클래스 생성

```java
public class CustomQueryProvider extends AbstractJpaQueryProvider {
	
	@Override
	public Query createQuery() {
		EntityManager entityManager = getEntityManager();
		Query query = entityManager.createQuery("SELECT p.id, p.bizRegFilePath FROM Pharmacy p WHERE p.createdDate <= :expiredDate and p.bizRegFilePath is not NULL ORDER BY p.id")
		query.setParameter("expiredDate", LocalDateTime.now().minusMonths(1));
		return query;
	}
}

//설정
@Bean
@StepScope
public JpaPagingItemReader<Object[]> bizRegFileDeleteReader(
        @Value("#{jobParameters[currentDate]}") String currentDate
) {

    HashMap<String, Object> parameterValues = new HashMap<>();
    parameterValues.put("expiredDate", LocalDateTime.now().minusMonths(1));

    return new JpaPagingItemReaderBuilder<Object[]>()
            .name("bizRegFileDeleteReader")
            .entityManagerFactory(entityManagerFactory)
            .pageSize(chunkSize)
            .queryProvider(new CustomQueryProvider()) //QueryProvider 지정
            .parameterValues(parameterValues)
            .build();
}
```

<br>

### 🌈 참고
- [https://jooy-p.tistory.com/25](https://jooy-p.tistory.com/25)
- [JpaPagingItemReader 코드](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/JpaPagingItemReader.java)

