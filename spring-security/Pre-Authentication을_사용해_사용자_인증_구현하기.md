# 📌 구현을 위해 필요한 Spring Security 사전 지식 정리

<br>

## ▶️ 공통 내용

Spring 기반의 어플리케이션의 보안(인증, 권한, 인가 등)을 담당하는 스프링 하위 프레임워크이다.
인증과 권한에 대한 부분은 Filter 흐름에 따라 처리한다.
- Filter는 Dispatcher Servlet으로 가기 전에 적용되므로 가장 먼저 URL 요청을 받는다.
- Interceptor는 Dispatcher와 Controller 사이에 위치한다.
  - Servlet Authentication 아키텍쳐
  
    ![image](https://user-images.githubusercontent.com/69254943/190582727-07acec3b-9a49-48e0-863f-f75486396796.png)
    
    - Authentication
        - Principal - username/password로 인증을 할 경우 UserDetails의 객체가 들어간다.
    - AuthenticationManager
        - Spring Security의 Filters가 어떻게 인증을 수행할 것인지에 대해 정의해놓는 주체이다. 인터페이스이므로 구현체가 필요하고, 일반적으로 ProviderManager 클래스를 구현체로 사용한다.
        - AuthenticationManager(인터페이스) > ProviderManager(구현되어 있는 구현체) > AuthenticationProviders(직접 등록 가능)
    - 참고
        - [https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontext](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontext) → 자세하게 나와있는 공식문서

<br>

## ▶️ 인증 방식
Spring Security에서 제공하는 인증 방식은 여러 개가 존재하고 그 중 아래 2개에 대해 살펴보면 다음과 같다.

<br>

### Username/Password Authentication
- 인증과 인가를 위해 Principal을 아이디로, Credential을 비밀번호로 사용하는 Credential 기반의 인증 방식을 사용한다.
- 관련 아키텍쳐

  ![image](https://user-images.githubusercontent.com/69254943/190583173-2ae6029d-8cd2-4974-8f15-370b5cb0ae5c.png)

<br>

### Pre-Authentication
- 외부에 의해 이미 인증을 거친 사용자에 대한 권한 체크 구현을 위해 사용된다. 예를 들어, X.509나 Siteminder와 같이 인증을 거친 사용자를 말한다. ([https://roytuts.com/spring-security-pre-authentication-example/](https://roytuts.com/spring-security-pre-authentication-example/))

![image](https://user-images.githubusercontent.com/69254943/190583304-5aee60c9-4946-4b12-b0da-3957badbbed2.png)

- 공식문서를 바탕으로 아키텍쳐를 그려보면 위와 같다. 흔히 JWT 로그인 방식 구현에 사용되는 위의 Username/Password의 인증 아키텍쳐와 구성은 동일하고, 각각이 사용하는 클래스나 인터페이스가 용도에 따라 다른 것 뿐이다.

필요한 클래스들을 자세히 살펴보면 다음과 같다.

**AbstractPreAuthenticatedProcessingFilter**

- security context의 현재 내용을 체크하고, 비어있다면, HTTP 요청으로부터 사용자 정보를 추출하여 AuthenticationManager에게 넘긴다.
        - 해당 클래스를 거쳐 반환되는 데이터(Principal, Credentials 등)를 포함한 PreAuthenticatedAuthenticationToken을 생성하고, 이를 인증을 위해 넘긴다.
- 해당 클래스의 하위 클래스는 아래 2가지 메서드를 오버라이딩해야 한다.

    ```java
    protected abstract Object getPreAuthenticatedPrincipal(HttpServletRequest request);
                
    protected abstract Object getPreAuthenticatedCredentials(HttpServletRequest request);
    ```

**PreAuthenticatedAuthenticationProvider**

- UserDetails 객체를 반환하는 것을 AuthenticationUserDetailsService 인터페이스에 위임한다. (해당 인터페이스의 구현체에서 해당하는 객체를 찾고, 없으면 예외 처리하는 것임)

    ```java
    //AuthenticationUserDetailsService
    public interface AuthenticationUserDetailsService {
        UserDetails loadUserDetails(Authentication token) throws UsernameNotFoundException;
    }

    //UserDetailsServce
    public interface UserDetailsServce {
        UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
    }
    ```

- UserDetailsService 인터페이스와 달리 메서드의 인자로 Authentication 객체를 전달 받는다.
- Pre-Authentication에서는 AuthenticationUserDetailsService 인터페이스의 구현체로 PreAuthenticatedGrantedAuthoritiesUserDetailsService 클래스를 주로 사용한다. → 필요에 맞게 따로 구현체 클래스 정의해줘도 된다. (나는 새로 정의해서 사용하였다)

<br>

# 📌 Spring Security를 사용한 외부 사용자 인증 구현 상세 내용

<br>

## ▶️ 어떤 인증 방식을 사용했는가?

- Spring Security의 Pre-Authentication 인증 방식을 사용했다.
- Spring Security에서 공식으로 제공해주는 인증 방식으로, 기존에 회사 서비스에 적용되어 있던 Spring Security의 Username/Password 인증 방식과 유사한 구조였다.
- 또한 아이디와 비밀번호로 인증하는 방식 대신에 Header로 전달된 AppId와 API Key 값으로 사용자 인증을 해야했던 개발 상황에 가장 적합하여 사용하였다.

<br>

## ▶️ 전체 구조 - 다중 보안 설정

<br>

### 다중 보안 구조를 선택한 이유

- 기존에 Spring Security의 Username/Password 인증 방식이 적용되어 있던 회사 서버에 새로운 인증을 추가해야 했으므로, Spring Security 기반의 기존 구현 구조를 최대한 사용하기 위해 다중 보안 구조를 선택하였다.

<br>

### 전반적인 구조

![image](https://user-images.githubusercontent.com/69254943/190584528-bb3cf2ce-c072-4eb1-acb0-b3e9eb19c147.png)

- 위의 그림에서 왼쪽이 새롭게 만든 ApiKey 관련 SecurityFilterChain이 되고, 오른쪽이 기존의 JWT 관련 SecurityFilterChain이다.
- 어플리케이션을 실행하면, FilterChainProxy의 SecurityFilterChains 리스트에 SecurityFilterChain이 등록되게 된다.
- 요청이 들어오면 FilterChainProxy에서 SecurityFilterChains 리스트에 있는 SecurityFilterChain 객체들의 RequestMatcher 멤버변수를 통해 매칭되는 SecurityFilterChain 객체를 우선적으로 찾고, 해당 객체에 저장된 Filter들을 가지고와 인증 처리를 진행한다.

<br>

## ▶️ 상세 구조 - Pre-Authentication 적용

<br>

### Pre-Authentication 아키텍쳐

![image](https://user-images.githubusercontent.com/69254943/190584997-24d341e6-e640-4246-90df-f7224b4fff2e.png)

- 위의 아키텍쳐를 기반으로 구현하였다.

<br>

### 구현 코드

`ApiKeySecurityConfiguration` 클래스

```java
@Slf4j
@Configuration
@EnableWebSecurity
@Order(1)
public class ApiKeySecurityConfiguration extends WebSecurityConfigurerAdapter {

    private final ApiKeyProvider apiKeyProvider;
    private final ApiKeyAuthFailureHandler apiKeyAuthFailureHandler;

    @Builder
    public ApiKeySecurityConfiguration(ApiKeyProvider apiKeyProvider, ApiKeyAuthFailureHandler apiKeyAuthFailureHandler) {
        this.apiKeyProvider = apiKeyProvider;
        this.apiKeyAuthFailureHandler = apiKeyAuthFailureHandler;
    }

    @Bean
    public AuthenticationManager getAuthenticationManager() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder authBuilder) throws Exception {
        //ProviderManager(AuthenticationManager의 구현체)의 Authentication Providers에 ApiKeyProvider 등록
        authBuilder.authenticationProvider(apiKeyProvider);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                .antMatcher("/*/api/pharm/**")
                .httpBasic().disable()
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .addFilterBefore(apiKeyAuthFilter(), UsernamePasswordAuthenticationFilter.class);
    }

    /** ApiKeyAuthFilter 객체 생성 */
    protected ApiKeyAuthFilter apiKeyAuthFilter() throws Exception {
        ApiKeyAuthFilter apiKeyAuthFilter = new ApiKeyAuthFilter(getAuthenticationManager());
        apiKeyAuthFilter.setAuthenticationFailureHandler(apiKeyAuthFailureHandler);
        apiKeyAuthFilter.afterPropertiesSet();
        return apiKeyAuthFilter;
    }
}
```

- WebSecurityConfigurerAdapter 클래스를 상속받은 클래스를 새로 정의
- @Order(1)로 지정해 SecurityFilterChains 리스트에 기존의 JWT 인증 관련 SecurityFilterChain보다 앞에 등록하여 해당 SecurityFilterChain을 먼저 거치도록 하였다.
  - 기존의 JWT 관련 인증의 RequestMatcher 범위가 더 크므로, 해당 범위가 더 작은 ApiKey 관련 인증을 먼저 등록해서 RequestMatcher를 판별하도록 해야하기 때문이다.
- 상세 설명
  - AuthenticationManager를 @Bean으로 생성하고, 이때 AuthenticationManager 인터페이스의 구현체는 기본으로 구현되어 있는 ProviderManager 클래스이다.
  - ApiKeyProvider 클래스를 ProviderManager 클래스의 AuthenticationProviders에 직접 등록한다.
  - 생성자를 사용해 ApiKeyAuthFilter 객체를 생성하고, 이때 AuthenticationManager 객체와 ApiKeyAuthFailureHandler 객체를 함께 등록한다.
  - 전체 구조 그림에서의 SecurityFilterChain의 Filters에 ApiKeyAuthFilter가 등록된다.
    - 이때 ApiKeyAuthFilter를 @Bean으로 생성하면 안된다. ApplicationFilterChain에 등록되기 때문이다. 이것 때문에 에러가 발생했다. (Spring Security CORS 에러 처리 문서화에 자세히 있음^^)


<br>  

`ApiKeyAuthFilter` 클래스

```java
@Slf4j
public class ApiKeyAuthFilter extends AbstractPreAuthenticatedProcessingFilter {

    //AuthenticationManager 등록
    public ApiKeyAuthFilter(@Qualifier("getAuthenticationManager") AuthenticationManager authenticationManager) {
        setAuthenticationManager(authenticationManager);
    }

    /** Authentication의 Principal 반환 */
    @Override
    protected Object getPreAuthenticatedPrincipal(HttpServletRequest request) {
        log.debug("ApiKeyAuthFilter의 getPreAuthenticatedPrincipal() 실행");

        return ApiKeyAuthReqDto.builder()
                .uuid(request.getHeader("Authorization"))
                .appId(request.getHeader("appId")).build();
    }

    /** Authentication의 Credentials 반환 */
    @Override
    protected Object getPreAuthenticatedCredentials(HttpServletRequest request) {
        log.debug("ApiKeyAuthFilter의 getPreAuthenticatedCredentials() 실행");

        return "N/A";
    }
}
```

- AbstractPreAuthenticatedProcessingFilter 클래스를 상속받는다.
- SecurityContext의 현재 내용을 체크하고, 비어있다면, HTTP 요청으로부터 사용자 정보를 추출하여 AuthenticationManager에게 넘긴다.
  - 해당 클래스를 거쳐 반환되는 데이터(Principal, Credentials 등)를 포함한 PreAuthenticatedAuthenticationToken을 생성하고, 이를 인증을 위해 AuthenticationManager에게 넘긴다.
- getPreAuthenticatedPrincipal()에서 Authentication의 Principal 부분을 반환한다.
  - Header로 들어오는 Authorization과 appId 값을 DTO 객체로 저장한다.
- getPreAuthenticatedCredentials()에서 Authentication의 Credentials 부분을 반환한다.
  - 사용되지 않을 값이기 때문에 의미없는 값을 반환한다.

<br>  

`ApiKeyProvider` 클래스

```java
@Component
public class ApiKeyProvider extends PreAuthenticatedAuthenticationProvider {

    //setPreAuthenticatedUserDetailsService()로 구현체 명시
    public ApiKeyProvider(ApiKeyAuthService apiKeyAuthService) {
        setPreAuthenticatedUserDetailsService(apiKeyAuthService);
    }

}
```

- UserDetails 객체를 반환하는 것을 AuthenticationUserDetailsService 인터페이스에 위임한다.
- PreAuthenticatedAuthenticationProvider 클래스를 상속받는다.
  - 인증을 위해 해당 클래스의 *public* Authentication authenticate(Authentication authentication) 메서드가 실행된다.
  - 해당 메서드 내에서 ‘DB나 Redis에서 조회하여 실제 존재하는 객체인지 확인하는 작업을 수행하기 위해 AuthenticationUserDetailsService<PreAuthenticatedAuthenticationToken> 인터페이스를 구현한 구현체’의 loadUserDetails() 메서드를 호출한다. 따라서 해당 인터페이스의 구현체를 setPreAuthenticatedUserDetailsService()를 사용해 명시해줘야 한다.

<br>  

`ApiKeyAuthService` 클래스

```java
@Slf4j
@Service
public class ApiKeyAuthService implements AuthenticationUserDetailsService<PreAuthenticatedAuthenticationToken> {

    private final ApiKeyCacheRepository apiKeyCacheRepository;

    public ApiKeyAuthService(ApiKeyCacheRepository apiKeyCacheRepository) {
        this.apiKeyCacheRepository = apiKeyCacheRepository;
    }

    /** UserDetails 반환 */
    @Override
    public UserDetails loadUserDetails(PreAuthenticatedAuthenticationToken token) throws UsernameNotFoundException {
        //전달된 Header 값(Authorization, appId)에 해당하는 객체 존재 여부 확인
        ApiKeyAuthReqDto reqDto = (ApiKeyAuthReqDto) token.getPrincipal();

        log.debug("Authorization : {}, appId : {}", reqDto.getUuid(), reqDto.getAppId());

        if (StringUtils.isBlank(reqDto.getUuid()) || StringUtils.isBlank(reqDto.getAppId())) {
            throw new UsernameNotFoundException("uuid, appId");
        }
        
        //Redis 조회
        ApiKeyCache apiKeyCache = apiKeyCacheRepository.findById(reqDto.getUuid())
                .orElseThrow(() -> new UsernameNotFoundException("uuid"));

        if (!apiKeyCache.getAppId().equals(reqDto.getAppId())) {
            throw new UsernameNotFoundException("appId");
        }

        return new User(reqDto.getUuid(), "N/A", true, true, true, true, token.getAuthorities());
    }
}
```

- DB나 Redis로부터 존재하는 사용자인지 확인하는 실질적인 인증 작업이 수행된다.
  - 필요한 Header 2개의 값 중 하나라도 null or blank 일 때 UsernameNotFoundException 예외를 반환하도록 먼저 체크하였다.
  - 통과하면, Redis에서 조회하여 체크한다.
    - Redis에 uuid와 appId를 HashTable 형태로 저장하였고, uuid(ApiKey테이블의 필드)를 @Id로 설정하였다.
- SpringSecurity에서 제공해주는 UserDetails 인터페이스를 구현한 User 객체를 반환한다. (PreAuthenticatedGrantedAuthoritiesUserDetailsService 클래스를 참고함)
  - UserDetails 객체를 여기서 반환하게 되는 것이다.
  - username은 uuid 값을 넣었다.
    - 해당 값이 SecurityContext에 저장되는 Authentication의 Principal의 name으로 저장된다.
    - Controller에서 API 호출 횟수 체크를 위해 해당 값만 username에 넣었다.

<br>  

`ApiKeyAuthFailureHandler` 클래스

```java
@Component
public class ApiKeyAuthFailureHandler implements AuthenticationFailureHandler {

    @Value("${hdmedi.server.Domain}")
    private String hdmediServerDomain;

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {

        response.sendRedirect(hdmediServerDomain + "/exception/accessdenied");
    }
}
```
- AuthenticationFailureHandler 인터페이스를 구현하여 예외 발생에 대한 처리로 기존 설정인 /exception/accessdenied로 redirection하도록 하였다.
  
