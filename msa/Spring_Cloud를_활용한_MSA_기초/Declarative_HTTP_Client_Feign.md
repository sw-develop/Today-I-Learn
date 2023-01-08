# Declarative HTTP Client - Feign

- Interface 선언을 통해 자동으로 Http Client를 생성한다.
- RestTemplate은 concrete 클래스라 테스트하기 어렵다.
    - concrete 클래스?
        - new 키워드를 사용해 인스턴스를 만드는 클래스임
        - 클래스 내의 모든 메서드가 구현되어 있어야 함
- 관심사의 분리
    - 서비스의 관심 - 다른 리소스, 외부 서비스 호출과 리턴값
    - 관심 X - 어떤 URL, 어떻게 파싱할 것인가
- Spring Cloud에서 Open-Feign 기반으로 Wrapping 한 것이 Spring Cloud Feign

<br>

### 실습 ⭐️
- 목적
    - Display --> Product 호출 시 Feign을 사용해 RestTemplate을 대체하도록 한다.

#### (1) Feign Client 사용하기
```java
@FeignClient(name = "product", url = "http://localhost:8082/")  
public interface FeignProductRemoteService {  
  
    @RequestMapping(path = "/products/{productId}")  
    String getProductInfo(@PathVariable("productId") String productId);  
}
```
- 위의 코드를 통해 Feign Client 구현이 가능하다.
- 동작 원리
    - FeignProductRemoteService용 Spring Bean을 자동 생성하고, 해당 Bean을 주입받아 사용함
- 정리하자면,
    - Feign은 interface 선언을 통해 자동으로 HTTP Client를 만들어주는 Declarative Http Client이다.
    - Open-Feign 기반으로 Spring Cloud가 Wrapping한 것이 Spring Cloud Feign이다.
    - 위의 코드처럼 Feign + URL 명시는
        - Ribbon 적용 X
        - Eureka 적용 X
        - Hystrix 적용 X

<br>

#### (2) Feign + Hystrix, Ribbon, Eureka
- 배경
    - Feign의 또 다른 장점은 Ribbon + Eureka + Hystrix와 통합되어 있다는 점이다.

```java
@FeignClient(name = "product")  
public interface FeignProductRemoteService {  
  
    @RequestMapping(path = "/products/{productId}")  
    String getProductInfo(@PathVariable("productId") String productId);  
}
```

```YAML
feign:
	hystrix:
		enabled: true
```
- 기존 코드에서 URL 명시를 제거하면 Ribbon + Eureka가 함께 사용되고, YML 파일에 다음 설정을 추가하면 Hystrix도 함께 사용된다.
- 즉, Eureka에서 @FeignClient(name = {...})에 명시한 서버 목록을 조회한 뒤, ribbon을 통해 load-balancing하면서 HTTP 호출을 수행한다. ⭐️

<br>

#### (3) Factory 패턴을 사용해 기본 Fallback의 에러 원인(Exception) 파악하기
```java
@Component  
public class FeignProductRemoteServiceFallbackFactory implements FallbackFactory<FeignProductRemoteService> {  
  
    @Override  
    public FeignProductRemoteService create(Throwable cause) {  
        System.out.println("t = " + cause);  
        return productId -> "[ this product is sold out !!! ]";  
    }  
}
```
- FallbackFactory< T > 인터페이스를 openfeign에서 제공해준다.

```java
@FeignClient(name = "product", fallbackFactory = FeignProductRemoteServiceFallbackFactory.class)  
public interface FeignProductRemoteService {  
    @RequestMapping(path = "/products/{productId}")  
    String getProductInfo(@PathVariable("productId") String productId);  
}
```
- 'fallbackFactory' 속성에 Hystrix Fallback을 명시한다.

<br>

#### (4) Feign용 Hystrix 프로퍼티 정의하기
```YAML
hystrix:  
  command:  
    productInfo:    # command key. use 'default' for global setting.  
      execution:  
        isolation:  
          thread:  
            timeoutInMilliseconds: 3000  # default 1,000ms
      circuitBreaker:  
        requestVolumeThreshold: 1   # Minimum number of request to calculate circuit breaker's health. default 20  
        errorThresholdPercentage: 50 # Error percentage to open circuit. default 50  
    FeignProductRemoteService#getProductInfo(String):  # command key
      execution:  
        isolation:  
          thread:  
            timeoutInMilliseconds: 3000  
      circuitBreaker:  
        requestVolumeThreshold: 1   
        errorThresholdPercentage: 50
```
- Feign을 사용하는 경우 프로퍼티 명시 시 commandKey 이름을 주의해야 한다.

<br>

### 정리하자면,
- Feign은 인턴페이스 선언 + 설정 으로 다음과 같은 것들이 가능하다.
    - HTTP Client
    - Eureka 타겟 서버 주소 획득
    - Ribbon을 통한 Client-Side Load Balancing
    - Hystrix를 통한 메서드별 Circuit Breaker

<br>

### 장애 유형 별 동작 예시

![IMG_18676584A963-1](https://user-images.githubusercontent.com/69254943/211208773-e485aa01-a83d-43b8-ac5c-e4b37c7dafc2.jpeg)
