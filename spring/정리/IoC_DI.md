# IoC(Inversion of Control) - 제어의 역전

<br>

## IoC란?

- 스프링 프레임워크에서는 객체(빈)의 생성, 연관관계 설정 등 객체의 관리를 스프링 컨테이너가 담당한다.
- 스프링 컨테이너가 코드 대신 객체에 대한 제어권을 가지고 있다고 하여 IoC라고 부른다.
- 따라서, 스프링 컨테이너를 IoC 컨테이너라고도 부른다.

<br>

## IoC 컨테이너란?

- 스프링 프레임워크에서 IoC를 담당하는 컨테이너를 BeanFactory, DI 컨테이너, ApplicationContext라고 부르기도 한다.
- DI를 위한 BeanFactory의 기능을 상속받아 여러 컨테이너 기능을 추가한 것을 ApplicationContext라고 부른다.
- 즉, ApplicationContext는 그 자체로 DI와 IoC를 포함한 그 이상의 기능을 제공한다.

<br>

## BeanFactory와 ApplicationContext

- 관계

  ![image](https://user-images.githubusercontent.com/69254943/193457542-5d0b86d5-d4c6-405d-b3f8-786e49a92137.png)

<br>

### BeanFactory

- 스프링 컨테이너의 최상위 인터페이스이다.
- 스프링 빈을 관리하고 조회하는 역할을 담당한다.
- 대표적으로 `getBean()` 메서드를 제공한다.

<br>

### ApplicationContext

- BeanFactory 기능을 모두 상속받아 제공한다.
- BeanFactory 이외의 다른 인터페이스들도 구현하고 있다. 따라서 빈 관리 기능 뿐 아니라 추가적인 기능을 제공한다.
    - 메시지 소스를 활용한 국제화 기능 (MessageSource)
    - 환경변수 (EnvironmentCapable)
        - 로컬, 개발, 운영 등을 구분해서 처리
    - 애플리케이션 이벤트 (ApplicationEventPublisher)
        - 이벤트를 발행하고 구독하는 모델을 편리하게 지원
    - 편리한 리소스 조회 (ResourcePatternResolver)
        - 파일, classpath, 외부 등에서 리소스를 편하게 조회

<br>

## 설정 메타 정보

- IoC 컨테이너의 가장 기초적인 역할은 객체 생성하고 관리하는 것이다.
- 스프링 컨테이너가 관리하는 객체를 ‘Bean’이라고 부른다.
- 설정 메타 정보는 빈을 어떻게 만들고, 어떻게 동작하게 할 것인가에 대한 정보이다.

- 스프링 컨테이너는 자바 코드, XML, Groovy 등 다양한 형식의 설정 정보를 받아들일 수 있도록 유연하게 설계되어 있다.

  ![image](https://user-images.githubusercontent.com/69254943/193457549-d0bc23a8-8404-4cd7-a52d-81b80ccaa3b8.png)

    - Annotation 기반 자바 코드 설정

        ```java
        @Configuration
        public class AppConfig {
        	
        	@Bean
        	public MemberService memberService() {
        		return new MemberServiceImpl(memberRepository());
        	}
        }
        ```

        - @Configuration : 1개 이상의 빈을 제공하는 클래스의 경우 반드시 명시해야 한다.
        - @Bean : 빈으로 등록한다.

<br>

# DI(Dependency Injection) - 의존관계 주입

<br>

## Dependency(의존관계)란?

- “A가 B를 의존한다”는 것은 ‘의존 대상 B가 변하면, 그것이 A에 영향을 미친다’고 설명할 수 있다.
    - 토비의 스프링 참고
- 예시

    ```java
    class BurgerChef {
    	private HamburgerRecipe hr;
    
    	public BurgerChef() {
    		this.hr = new HamburgerRecipe();
    	}
    }
    ```

    - 위의 코드의 경우, 햄버거 레시피가 변화되었을 때, BurgerChef 클래스를 수정해야 한다.
    - 즉, BurgerChef가 HamburgerRecipe에 의존하는 상황이다.

<br>

### 의존성을 인터페이스로 추상화

- 위의 예시에서는 BurgerChef가 HamburgerRecipe에만 의존할 수 있는 구조이다.
- 더 다양한 객체를 의존할 수 있게 하려면, 인터페이스로 추상화해야 한다.
- 개선 예시

    ```java
    class BurgerChef {
        private BurgerRecipe burgerRecipe;
    
        public BurgerChef() {
            burgerRecipe = new HamBurgerRecipe();
            //burgerRecipe = new CheeseBurgerRecipe();
            //burgerRecipe = new ChickenBurgerRecipe();
        }
    }
    
    interface BugerRecipe {
        newBurger();
    } 
    
    class HamBurgerRecipe implements BurgerRecipe {
        public Burger newBurger() {
            return new HamBerger();
        }
    }
    ```

    - 의존관계를 인터페이스로 추상화하여,
        - 더 다양한 의존관계를 맺을 수 있다.
        - 실제 구현 클래스와의 관계가 느슨해지고, 결합도가  낮아진다.

<br>

## DI(의존관계 주입)란?

- 객체간의 의존관계를 외부(스프링 컨테이너)에서 결정하고 주입하는 것을 DI라고 한다.

![image](https://user-images.githubusercontent.com/69254943/193457577-cc6143eb-7bfb-418e-88ff-78732fa38c46.png)

→ 객체를 외부에서 생성하고, 외부에서 생성한 객체를 주입한다.

- 토비의 스프링에서는 다음의 3가지 조건을 충족하는 작업을 ‘DI’라고 정의한다.
    1. 클래스 모델이나 코드에는 런타임 시점의 의존관계가 드러나지 않는다.
        - 이를 위해 인터페이스만 의존하고 있어야 함
    2. 런타임 시점의 의존관계는 컨테이너나 팩토리 같은 제3자의 존재가 결정한다.
    3. 의존관계는 사용할 객체에 대한 레퍼런스를 외부에서 주입해줘서 만들어진다.
        - 즉, 외부에서 객체가 생성되는 것임

<br>

## DI(의존관계 주입) 구현 방법

- 스프링에서 제공하는 3가지 방법
    - 필드 주입
    - 수정자 주입
    - 생성자 주입

<br>

### 필드 주입

- 변수 선언부에 @Autowired 를 붙인다.
- 장점
    - 사용하기 편리하다.
- 단점
    - 단일 책임의 원칙(SRP) 위반
        - 하나의 클래스가 많은 책임을 갖게된다.
    - 의존성이 숨는다.
        - 의존관계 한눈에 파악하기 어렵다.
    - 스프링 컨테이너에 대한 의존성이 높아진다.
        - 스프링 컨테이너가 없는 환경(테스트)에서는 의존관계 주입이 불가능하다.
    - 불변성을 보장하지 않는다.
        - final 선언 불가하다.
    - 순환 참조가 발생할 수 있다.

<br>

### 수정자 주입

- setter 메서드로 주입한다.
- 선택적인 의존성을 사용할 때 유용하다.
- 단점
    - 선택적인 의존성 사용이라 해당 구현체를 주입하지 않아도 클래스 객체 생성이 가능하다는 것이므로, 주입받지 않은 구현체 사용시 NullPointerException이 발생한다.

<br>

### 생성자 주입

- 생성자에 @Autowired 어노테이션을 붙여 의존성을 주입받는다.
    - 생성자가 1개인 경우 해당 어노테이션 생략 가능
- 장점
    - 의존관계를 모두 주입하지 않으면 객체 생성이 불가능하다.
        - NPE 발생 X
    - 불변성을 보장할 수 있다.
        - final 키워드 사용
    - 순환 참조 에러를 방지할 수 있다.
        - 스프링 컨테이너가 빈을 생성하는 시점에 순환 참조를 확인하기 때문에 컴파일 단계에서 순환 참조를 잡아낼 수 있다. 서로 다른 여러 객체들이 서로를 참조하고 있는 것이다.
        - 스프링부트 2.6 부터는 순환 참조가 기본적으로 허용되지 않도록 변경되어 필드 주입을 받아도 순환 참조를 컴파일 시점에 알 수 있게 되었다.
    - 의존성을 주입하기 번거롭고 생성자 인자가 많아지면 코드가 길어져 위기감을 느끼게 된다.
        - 이를 바탕으로 SRP(단일책임원칙)에 대해 생각하게 되고, 리팩토링을 하게 된다.

<br>

## @Autowired

<br>

### @Autowired란?

- DI(의존관계 주입)를 위해 사용하는 어노테이션이다.
- 의존관계 타입에 해당하는 빈을 찾아서 주입해주는 역할을 한다.

- 스프링 서버가 올라갈 때 ApplicationContext가
    - @Bean, @Service, @Controller 등의 어노테이션을 이용하여 등록한 스프링 빈을 생성하고, @Autowired 어노테이션이 붙은 위치에 의존성 주입을 수행한다.

- 자바 리플렉션을 통해 수행한다.


