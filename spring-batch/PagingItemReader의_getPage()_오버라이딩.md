# 하나의 테이블의 컬럼 값을 수정하는 배치 작업시 Spring Batch의 PagingItemReader를 사용할 때 처리되지 않는 값이 존재하는 문제 해결하기

<br>

## 📌 상황

- 인턴 업무에서 약국 데이터 필드 중 등록일이 1달 이상 지난 사업자등록증 필드의 값을 null로 변경하는 배치를 구현하게 되었다.
- 이때, 값을 조회해오는 테이블과 배치로 값이 변경되는 테이블이 동일하기 때문에, PagingItemReader를 사용한다면, 조회할 때마다 반환 결과가 달라지는데, Paging 값은 그대로 커지므로, 처리가 되지 않는 값이 존재하는 문제가 발생하였다.

<br>

## 📌 해결

<br>

### ▶️ 방식1 - Cursor 사용
- Cursor는 한번 데이터베이스와 커넥션을 맺고, Database Cursor를 옮기는 방식이므로 처음 조회했던 결과가 갱신되는 일이 없다.
- JPA는 Cursor를 지원하지 않는다. → Hibernate나 Jdbc를 사용해야 함
- 조회 데이터가 많은 경우 부하 발생 가능성 있다.

<br>

### ▶️ 방식2 - PagingReader의 getPage() 오버라이딩
- Page 번호를 변경시키지 않고 0으로 고정시킨다.

```java
//예시 코드
JpaPagingItemReader<Object[]> jpaPagingItemReader = new JpaPagingItemReader<Object[]>() {
        @Override
        public int getPage() {
            return 0;   //Page 값을 0으로 고정
        }
};
```

<br>

### ▶️ 채택
- JPA를 사용하고, 조회 데이터가 많은 경우에 대비하고자 방식2로 구현하였다.

<br>

### 🌈 참고 자료
- https://jojoldu.tistory.com/337
- https://docs.spring.io/spring-batch/docs/4.1.x/reference/html/readersAndWriters.html#database





