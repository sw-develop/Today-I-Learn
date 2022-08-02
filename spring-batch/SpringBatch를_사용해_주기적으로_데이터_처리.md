# Spring Batch를 사용해 주기적으로 데이터 처리하기

<br>

## 📌 상황

- 인턴 업무 중 등록일이 1달 이상 지난 사업자등록증을 주기적으로 삭제하는 업무를 맡게 되었고, Spring Batch를 사용해 효율적으로 처리할 수 있을 것이라고 생각하여 Spring Batch를 사용해 구현하게 되었다.

<br>

### ▶️ 기존 코드의 문제점 발견
- 기존에 작성되어있던 Spring Batch 관련 회사 코드는 ItemReader 인터페이스를 구현한 CustomItemReader인 QueueItemReader를 구현해서 사용하였고, 데이터베이스에서 처리할 데이터를 조회해 올 때 JPA Repository를 사용해 해당하는 데이터를 한 번에 모두 메모리에 올리는 상황이었다.

#### 기존 방식 예시 코드
```java
//ItemReader 인터페이스의 구현체인 QueueItemReader
public class QueueItemReader<T> implements ItemReader<T> {

    private Queue<T> queue;

    public QueueItemReader(List<T> data) {
        this.queue = new LinkedList<>(data);
    }

    @Override
    public T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        return this.queue.poll();
    }
}

//데이터 조회 방식 예시
@Bean
@StepScope 
public QueueItemReader<User> customReader() {
    List<User> users =
            userRepository.findAll(); //해당하는 데이터를 한 번에 모두 조회함
    return new QueueItemReader<>(users);
}
```

<br>

이후, 팀장님께서 ```‘Chunk 단위로 처리되는데도 문제가 있나요?’```라고 질문을 주셨고, 문제 상황을 구체적으로 파악해보았다.

Chunk 단위 처리는 ItemReader로 조건에 해당하는 데이터를 가져온 후 Chunk 단위만큼 ItemWriter에 전달해 작업을 수행하는 것이다.

예시와 함께 알아보면,
- DB에 존재하는 처리 대상 데이터 = 3개, Chunk Size = 2, Page Size = 2
- 방식1 - 기존 회사 코드의 방식
  - 수행 순서 : ItemReader → ItemWriter → ItemWriter
      - ItemReader에서 조건에 해당하는 데이터 3개를 데이터베이스에서 가져와 모두 메모리에 올림 → ItemWriter에서 Chunk Size인 2개 데이터에 대한 write()을 수행 → ItemWriter에서 나머지 1개 데이터에 대한 write()을 수행
- 방식2 - PagingItemReader 사용
  - 수행 순서 : ItemReader → ItemWriter, ItemReader → ItemWriter
      - Page Size만큼만 데이터를 가져와 처리함



즉, Chunk 단위 처리가 요점이 아닌 ItemReader로 데이터를 조회해 메모리에 올릴 때가 주요 요점인 상황으로 설명할 수 있었다.

이는 대규모 데이터 처리시 성능 이슈가 발생할 가능성이 있으므로 해당 내용을 팀장님께 공유드렸고, 기존 코드 대신 PagingItemReader를 사용해 구현하게 되었다.

<br>

## 📌 구현 및 개선

- 구현한 Spring Batch의 동작 과정은 다음과 같다.

![image](https://user-images.githubusercontent.com/69254943/182304737-b6b4aa6b-f615-42ce-8f16-8be0a13a0eed.png)

<br>

### ▶️ Chunk Oriented Processing 방식 적용

![image](https://user-images.githubusercontent.com/69254943/182305009-2d35b5b6-68fb-4e52-b755-5c39e892eae5.png)

- 해당 방식의 처리 과정은 다음과 같다.
    - Item 단위로 한 번에 하나씩 데이터를 읽고 처리 → 가공된 데이터들을 별도의 공간에 모음 → Chunk 단위만큼 쌓이면 ItemWriter에 전달하고 일괄 처리
    - Chunk 단위로 트랜잭션을 수행하기 때문에 실패할 경우 Chunk 만큼만 롤백이 되고, 이전 커밋된 트랜잭션은 반영이 된다.

<br>

### ▶️ JpaPagingItemReader 사용
- [Spring Batch 공식문서](https://docs.spring.io/spring-batch/docs/current/reference/html/readersAndWriters.html#database)를 보면 Database 연결용 ItemReader로 크게 Cursor 기반과 Paging 이 2가지의 ItemReader 구현체를 제공한다.
- JPA에서는 Cursor 기반 데이터베이스 접근을 지원하지 않으므로, JPA를 사용하기 위해서 해당 구현체를 사용하였다.
- Cursor는 하나의 Connection으로 Batch가 끝날 때까지 사용되기 때문에 작업이 끝나기 전에 데이터베이스와 어플리케이션의 연결이 먼저 끊어질 수 있다. 따라서 배치 수행 시간이 오래 걸리는 경우에는 PagingItemReader를 사용하는게 좋다. (Paging의 경우 한 페이지를 읽을 때마다 Connecton을 맺고 끊음)

#### 구현 코드
```java
@Bean
@StepScope
public JpaPagingItemReader<Object[]> bizRegFileDeleteReader(
            @Value("#{jobParameters[currentDate]}") String currentDate
) {
        log.info("bizRegFileDeleteReader execute : {}", currentDate);

        JpaPagingItemReader<Object[]> jpaPagingItemReader = new JpaPagingItemReader<Object[]>() {
            @Override
            public int getPage() {
                return 0;   //Page 값을 0으로 고정
            }
        };

        jpaPagingItemReader.setName("bizRegFileDeleteReader");
        jpaPagingItemReader.setEntityManagerFactory(entityManagerFactory);
        jpaPagingItemReader.setPageSize(chunkSize);
        jpaPagingItemReader.setQueryString(
        "SELECT p.id as id, p.bizRegFilePath as bizRegFilePath FROM Pharmacy p WHERE p.createdDate <= :expiredDate and p.bizRegFilePath is not NULL ORDER BY p.id"
        );    //등록 한 달 이상 지남 & 파일 경로가 null이 아닌 약국

        HashMap<String, Object> parameterValues = new HashMap<>();
        parameterValues.put("expiredDate", LocalDateTime.now().minusMonths(1));
        jpaPagingItemReader.setParameterValues(parameterValues);

        return jpaPagingItemReader;
}
```

- Spring Batch는 Chunk 단위로 처리를 수행하기 때문에 Chunk Size만큼 데이터가 모일 때까지 Paging으로 데이터를 가져오게 된다.
- 이때 JpaPagingItemReader는 페이지를 읽을 때, 이전 트랜잭션을 초기화하게 된다. 따라서 1번의 조회로 Chunk 단위 처리가 수행되도록 하기 위해 Page Size와 Chunk Size를 동일하게 설정하였다.

<br>

### ▶️ CustomItemWriter 사용
- s3와 데이터베이스에서 삭제 처리를 해야하므로, ItemWriter 인터페이스를 구현한 구현체를 새로 정의하여 사용하였다.

#### 구현 코드
```java
@Bean
@StepScope
public ItemWriter<Long> bizRegFileDeleteWriter(
        @Value("#{jobParameters[currentDate]}") String currentDate
) {
        log.info("bizRegFileDeleteWriter execute : {}", currentDate);

        return items -> {
            if (ListUtil.hasElement(items)) {
                List<Long> result = new ArrayList<>();
                for (Long item : items) {
                    if (item != null) {
                        result.add(item);
                    }
                }

                //필드 값 null로 변경
                pharmacyService.deleteBizRegFile(result);
            }
        };
}
```

<br>

## 📌 전체 코드
- ItemReader, ItemProcessor, ItemWriter로 구성하였고, ItemProcessor와 ItemWriter는 람다식으로 익명 클래스를 생성해 사용하였다.

```java
@Slf4j
@Configuration
public class BizRegFileDeleteJobConfiguration {

    //한 번에 처리할 데이터 갯수
    private final int chunkSize = 20;

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final EntityManagerFactory entityManagerFactory;

    private final PharmacyService pharmacyService;
    private final S3Service s3Service;

    public BizRegFileDeleteJobConfiguration(JobBuilderFactory jobBuilderFactory, StepBuilderFactory stepBuilderFactory, EntityManagerFactory entityManagerFactory, PharmacyService pharmacyService, S3Service s3Service) {
        this.jobBuilderFactory = jobBuilderFactory;
        this.stepBuilderFactory = stepBuilderFactory;
        this.entityManagerFactory = entityManagerFactory;
        this.pharmacyService = pharmacyService;
        this.s3Service = s3Service;
    }

    @Bean
    public Job bizRegFileDeleteJob() {
        log.info("bizRegFileDeleteJob execute");

        return jobBuilderFactory.get("bizRegFileDeleteJob")
                .start(bizRegFileDeleteStep(null))
                .build();
    }

    @Bean
    @JobScope
    public Step bizRegFileDeleteStep(
            @Value("#{jobParameters[currentDate]}") String currentDate
    ) {
        log.info("bizRegFileDeleteStep execute : {}", currentDate);

        return stepBuilderFactory.get("bizRegFileDeleteStep")
                .<Object[], Long>chunk(chunkSize)
                .reader(bizRegFileDeleteReader(currentDate))
                .processor(bizRegFileDeleteProcessor(currentDate))
                .writer(bizRegFileDeleteWriter(currentDate))
                .build();
    }

    @Bean
    @StepScope
    public JpaPagingItemReader<Object[]> bizRegFileDeleteReader(
            @Value("#{jobParameters[currentDate]}") String currentDate
    ) {
        log.info("bizRegFileDeleteReader execute : {}", currentDate);

        JpaPagingItemReader<Object[]> jpaPagingItemReader = new JpaPagingItemReader<Object[]>() {
            @Override
            public int getPage() {
                return 0;   //Page 값을 0으로 고정
            }
        };

        jpaPagingItemReader.setName("bizRegFileDeleteReader");
        jpaPagingItemReader.setEntityManagerFactory(entityManagerFactory);
        jpaPagingItemReader.setPageSize(chunkSize);
        jpaPagingItemReader.setQueryString(
                "SELECT p.id as id, p.bizRegFilePath as bizRegFilePath FROM Pharmacy p WHERE p.createdDate <= :expiredDate and p.bizRegFilePath is not NULL ORDER BY p.id"
        );    //등록 한 달 이상 지남 & 파일 경로가 null이 아닌 약국

        HashMap<String, Object> parameterValues = new HashMap<>();
        parameterValues.put("expiredDate", LocalDateTime.now().minusMonths(1));
        jpaPagingItemReader.setParameterValues(parameterValues);

        return jpaPagingItemReader;
    }

    @Bean
    @StepScope
    public ItemProcessor<Object[], Long> bizRegFileDeleteProcessor(
            @Value("#{jobParameters[currentDate]}") String currentDate
    ) {
        log.info("bizRegFileDeleteProcessor execute : {}", currentDate);

        return item -> {

            PharmacyDtoVo pdv = PharmacyDtoVo.builder()
                    .id(Long.parseLong(String.valueOf(item[0])))
                    .bizRegFilePath(String.valueOf(item[1]))
                    .build();

            //s3에서 삭제 - S3Service 참조가 admin에서 가능
            s3Service.deleteS3(pdv.getBizRegFilePath());
            return pdv.getId();
        };
    }

    @Bean
    @StepScope
    public ItemWriter<Long> bizRegFileDeleteWriter(
        @Value("#{jobParameters[currentDate]}") String currentDate
    ) {
        log.info("bizRegFileDeleteWriter execute : {}", currentDate);

        return items -> {
            if (ListUtil.hasElement(items)) {
                List<Long> result = new ArrayList<>();
                for (Long item : items) {
                    if (item != null) {
                        result.add(item);
                    }
                }

                //필드 값 null로 변경
                pharmacyService.deleteBizRegFile(result);
            }
        };
    }

}
```

<br>

### 🌈 참고
- [spring-batch 연습 repo](https://github.com/sw-develop/spring-batch-practice)
- https://jojoldu.tistory.com/336
- https://docs.spring.io/spring-batch/docs/current/reference/html/index.html


