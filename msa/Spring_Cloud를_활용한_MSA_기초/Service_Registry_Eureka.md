# Service Registry - Eureka

<br>

### 📌등장 배경
- Ribbon 예제에서 서버 목록을 yml에 직접 넣었는데 자동화할 방법은 없을까?
- 서버가 새롭게 시작되면 그것을 감지하여 목록에 자동으로 추가되고, 서버가 종료되면 자동으로 목록에서 삭제하기 위한 방법은 없을까?

<br>

### 📌Dynamic Service Discovery - Eureka
- Service Registry
    - 서비스 탐색 및 등록을 수행한다.
        - 클라우드의 전화번호부 역할 담당함
    - 침투적 방식의 코드 변경이 단점이긴 하지만, 큰 부분은 아니다.
- Discovery Client
    - spring-cloud에서 서비스 레지스트리 사용 부분을 추상화하여 인터페이스로 제공한다. --> 간단한 사용 가능해짐
    - Eureka, Consul, Zookeeper, etcd 등의 구현체가 존재한다.

![IMG_93ED2756D3BD-1](https://user-images.githubusercontent.com/69254943/211204087-c786dec1-c47c-425e-a1b9-5c2d95b21d9e.jpeg)

- Ribbon은 Eureka와 결합하여 사용할 수 있고, Eureka는 서버 목록을 자동으로 관리한다.

<br>

### 📌Eureka in Spring Cloud
- 서버 시작 시 Eureka Server(Registry)에 자동으로 자신의 상태를 등록한다. (UP)
    - eureka.client.register-with-eureka : true(default)
- 주기적 heartbeat로 Eureka Server에 자신이 살아 있음을 알린다.
    - eureka.instance.lease-renewal-interval-in-seconds : 30(default)
- 서버 종료 시 Eureka Server에 자신의 상태 변경(DOWN) 혹은 자신의 목록을 삭제한다.
- Eureka 상에 등록된 이름은 'spring.application.name'
- 구현
    - @EnableEurekaServer / @EnableEurekaClient를 통해 서버 구축, 클라이언트 Enable이 가능하다.
    - @EnableEurekaClient를 붙인 Application은 Eureka 서버로부터 남의 주소를 가져오는 역할과 자신의 주소를 등록하는 역할 모두 수행 가능하다.
    - EurekaClient가 Eureka 서버에 자신을 등록할 때 'spring.application.name'이 이름으로 사용된다. (application.yml에서 설정)

<br>

### 📌RestTemplate에 Eureka 적용하기
- @LoadBalanced를 사용해 Ribbon + Eureka를 연동한다.
- Eureka Client가 설정되면, 코드상에서 서버 주소 대신 Application 이름을 명시해 호출 가능하다.
- Ribbon의 Load Balancing과 Retry가 함께 동작한다.