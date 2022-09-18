# Spring의 Custom Annotation을 만들어 코드의 중복 제거하기

<br>

# 📌 상황

- 인턴으로 근무했던 스타트업에서 사내 OpenAPI 구현 프로젝트에서 사용자별 API 호출 횟수를 체크해 제한하는 기능이 있었다. [관련 내용](https://github.com/sw-develop/Today-I-Learn/blob/main/spring-security/%EC%99%B8%EB%B6%80_%EC%82%AC%EC%9A%A9%EC%9E%90_%EC%9D%B8%EC%A6%9D%EC%9D%84_%EC%9C%84%ED%95%9C_%EC%9D%B8%EC%A6%9D%ED%82%A4_%EB%B0%9C%EA%B8%89_%EC%84%A4%EA%B3%84.md)
- 기존에는 OpenAPI로 사용되는 모든 API에 대해 API 호출 횟수를 체크하는 코드를 포함되게 하였다.
- 하지만, 앞으로 추가될 API가 많아질수록 코드의 중복이 증가하고, 관심사가 흩어지는 단점이 존재하여 이를 개선하기로 하였다.

<br>

# 📌 개선

<br>

### ▶️ Custom Annotation을 생성해 AOP로 처리하기

- API 호출 횟수 체크에 대한 커스텀 어노테이션을 생성해 코드의 중복을 제거하고 관심사를 분리시켰다.
- 해당 어노테이션이 여러 곳에 쓰이고, 코드가 간결해지기 때문에 해당 방안으로 구성하기로 하였다.

- 적용한 코드는 다음과 같다.

```java
// ApiCallAvailable 어노테이션 인터페이스
@Target(ElementType.METHOD)
@Retention(value = RetentionPolicy.RUNTIME)
public @interface ApiCallAvailable {
}

// ApiCallAspect
@Aspect
@Component
@Slf4j
public class ApiCallAspect {

    private final HdmediConfigUtil hdmediConfigUtil;
    private final ApiKeyService apiKeyService;

    public ApiCallAspect(HdmediConfigUtil hdmediConfigUtil, ApiKeyService apiKeyService) {
        this.hdmediConfigUtil = hdmediConfigUtil;
        this.apiKeyService = apiKeyService;
    }

    @Before(value = "@annotation(ApiCallAvailable)")    //해당 어노테이션을 사용한 곳에서 실행
    public void apiCallAvailableCheck() {
        log.debug("apiCallAvailableCheck 실행");
        
        //요청 사용자의 uuid 값 추출
        String uuid = hdmediConfigUtil.getAuthenticatedExternalUserUuid();
        //요청 가능한지 체크
        apiKeyService.apiCallAvailableCheck(uuid);
    }

}

// 어노테이션 사용 예시
public class Controller {
	
	@Get("/get")
	@ApiCallAvailable //어노테이션 사용
	public ResponseEntity<?> void get() {
		return ResponseEntity.ok(); 
	}
}

```

<br>

### 참고

- [https://techblog.woowahan.com/2684/](https://techblog.woowahan.com/2684/)
- [https://linkeverything.github.io/springboot/spring-aop/](https://linkeverything.github.io/springboot/spring-aop/)




