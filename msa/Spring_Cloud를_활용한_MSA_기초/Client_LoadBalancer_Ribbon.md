# Client LoadBalancer - Ribbon

<br>

### 📌Server Side LoadBalancer
- 일반적인 L4 스위치 기반의 로드밸런싱이다.
    - 클라이언트는 해당 로드밸런서의 주소만 알고 요청을 보낸다.
    - L4 스위치는 서버의 목록을 알고 있다.
- H/W Server Side Load Balancer 단점
    - H/W가 필요함 (비용 up, 유연성 down)
    - 서버 목록 추가를 위해서는 설정 필요함 (자동화 어려움)
    - Load Balancing Schema가 한정적임 (Round Robbin, Sticky)
- 12 factors 중 dev/prod 요인을 만족하기 어렵다.

<img width="396" alt="Pasted image 20230108195446" src="https://user-images.githubusercontent.com/69254943/211194495-0bcabae7-a352-43eb-b8ec-1a2ee121459c.png">

<br>

### 📌Client LoadBalancer - Ribbon
- Client (API Caller)에 탑재되는 S/W 모듈
- 주어진 서버 목록에 대해서 로드 밸런싱을 수행한다.
- Ribbon의 장점
    - H/W 필요 없이 S/W로만 가능함 (비용 down, 유연성 up)
    - 서버 목록의 동적 변경이 자유로움
    - Load Balancing Schema를 마음대로 구성 가능함

<img width="369" alt="Pasted image 20230108195703" src="https://user-images.githubusercontent.com/69254943/211194503-c89204aa-c17d-4fe8-83d2-8e4a2fc51fb4.png">
    
<br>

### 📌실습
(1) RestTemplate에 Ribbon 적용하기
- Ribbon 의존성 추가 후 @LoadBalanced 추가
- 설정(application.yml)에 서비스의 주소 설정
- RestTemplate 사용 시 주소를 넣지 않고, 설정에 추가한 서비스 이름 사용

(2) Ribbon의 Retry 기능
- Round Robin 방식을 사용해 Client Load Balancing과 Retry를 수행한다.
- Ribbon은 여러 Component에 내장되어 있으며, 이를 통해 Client Load Balancing 수행이 가능하다.
- Ribbon에는 다양한 설정이 가능하다. (서버선택, 실패 시 skip 시간, Ping 체크)
- Ribbon에는 Retry 기능이 내장 되어있다.
- Eureka와 함께 사용될 때 강력하다.