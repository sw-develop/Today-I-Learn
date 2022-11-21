# 2.4 Docker의 작동 구조
- Docker는 Linux 커널 기술이 베이스로 되어 있다.

<br>

## 📌 컨테이너를 구획화하는 장치 (namespace) ⇒ 컨테이너별 독립된 값 및 공간 제공

- Docker는 컨테이너라는 독립된 환경을 만들고, 그 컨테이너를 구획화하여 애플리케이션의 실행 환경을 만든다.
- 이 컨테이너를 구획하는 기술은 Linux 커널의 namespace 기능을 사용한다. 이를 통해 호스트 상에서 컨테이너를 가상적으로 격리시킨다.
    - namespace : 한 덩어리의 데이터에 이름을 붙여 분할하여 충돌 가능성을 줄이고, 쉽게 참조할 수 있게 해는 개념임
    - 이름과 연결된 실체는 그 이름이 어떤 이름공간에 속해 있는지 고유하게 정해짐
    - 이름공간이 다르면 동일한 이름이더라도 다른 실체로 처리됨

Linux 커널의 namespace 기능은 Linux의 오브젝트에 이름을 붙여 다음과 같은 6개의 독립된 환경을 구축한다.

아래의 {value} namespace는 ⇒ namespace별로 value를 가진다는 의미이다.

1) PID namespace

- PID란 Linux에서 각 프로세스에 할당된 고유한 ID이다.
- PID와 프로세스를 격리시킨다. namespace가 다른 프로세스끼리는 서로 액세스할 수 없다.

2) Network namespace

- 네트워크 디바이스, IP 주소, 포트 번호, 라우팅 테이블, 필터링 테이블 등과 같은 네트워크 리소스를 격리된 namespace마다 독립적으로 가질 수 있다.
- 이 기능을 사용하면 호스트 OS 상에서 사용 중인 포트가 있더라도 컨테이너 안에서 동일한 번호의 포트를 사용할 수 있다.

3) UID namespace

- UID(사용자 ID), GID(그룹 ID)를 namespace별로 독립적으로 가질 수 있다.
- namespace 안과 호스트 OS상의 UID/GID가 서로 연결되어 namespace 안과 밖에서 서로 다른 UID/GID를 가질 수 있다.
    - ex. namespace 안에서는 UID/GID가 0인 root 사용자를, 호스트 OS 상에서는 일반 사용자로서 취급할 수 있음
        - 이는 namespace 안의 관리자 계정은 호스트 OS에 대해서는 관리 권한을 일절 갖지 않는다는 것을 의미하므로 보안이 뛰어난 격리 환경 구성 가능

4) MOUNT namespace

- Linux에서 파일 시스템을 사용하기 위해서는 마운트가 필요하다.
    - 마운트 : 컴퓨터에 연결된 기기나 기억장치를 OS에 인식시켜 이용 가능한 상태로 만드는 것
- 마운트 조작을 하면 namespace 안에 격리된 파일 시스템 트리를 만든다.
    - namespace 안에서 수행한 마운트는 호스트 OS나 다른 namespace에서는 액세스 불가함

5) UTS namespace

- namespace별로 호스트명이나 도메인명을 독자적으로 가질 수 있다.

6) IPC namespace

- IPC(프로스세간 통신) 오브젝트를 namespace별로 독립적으로 가질 수 있다.
    - IPC는 공유 메모리나 세마포어/메시지 큐를 말함


## 📌 릴리스 관리 장치 (cgroups) ⇒ 그룹별 자원 할당

- Docker에서는 물리 머신 상의 자원을 여러 컨테이너가 공유하여 작동한다. 이때 Linux 커널의 기능인 ‘control groups (cgroups)’ 기능을 사용해 자원의 할당 등을 관리한다.
- cgroups는 프로세스와 스레드를 그룹화하여, 그 그룹 안에 존재하는 프로세스와 스레드에 대한 관리를 수행하기 위한 기능이다.
    - Linux에서는 프로그램을 프로세스로서 실행하고, 프로세스는 하나 이상의 스레드 모음임
    - 호스트 OS의 CPU나 메모리와 같은 자원에 대해 그룹별로 제한을 둘 수 있음
    - 컨테이너 안의 프로세스에 대해 자원을 제한하여 어떤 컨테이너가 호스트 OS의 자원을 모두 사용해 버려 동일한 호스트 OS 상에서 가동되는 다른 컨테이너에 영향을 주는 일을 막을 수 있음
- cgroups의 주요 서브 시스템

    | 항목 | 설명 |
    | --- | --- |
    | cpu | CPU 사용량 제한 |
    | cpuacct | CPU 사용량 통계 정보 제공 |
    | cpuset | CPU나 메모리 배치 제어 |
    | memory | 메모리나 스왑 사용량 제한 |
    | devices | 디바이스에 대한 액세스 허가/거부 |
    | freezer | 그룹에 속한 프로세스 정지/재개 |
    | net_cls | 네트워크 제어 태그 부가 |
    | blkio | 블록 디바이스 입출력량 제어 |

- cgroups는 계층 구조를 사용해 프로세스를 그룹화하여 관리할 수 있다.

  ![image](https://user-images.githubusercontent.com/69254943/202955862-ed846a90-cefb-4534-94ce-aa74b919128a.png)

    - 위와 같이 사용자 애플리케이션과 데몬 프로세스를 나눠, 각각의 그룹에 CPU 사용량을 할당함
    - 부모자식 관계에서 자식이 부모의 제한을 물려받음
    - 자식이 부모의 제한을 초과하는 설정을 하더라도 부모 cgroups의 제한에 걸림 ⇒ 자식은 부모 그룹의 제한을 초과하여 할당할 수 없으므로 중요한 프로세스라도 영향을 받지 않음

<br>

## 📌 네트워크 구성 (가상 bridge / 가상 NIC)

- Linux는 Docker를 설치하면 서버의 물리 NIC가 docker0이라는 가상 브리지 네트워크로 연결된다.
    - docker0은 Docker 실행시킨 후에 디폴트로 만들어짐
- Docker 컨테이너가 실행되면 컨테이너에 172.17.0.0/16이라는 서브넷 마스크를 가진 private ip 주소가 eth0으로 자동으로 할당된다.
    - 이 가상 NIC는 OSI 참조 모델의 Layer 2인 가상 네트워크 인터페이스로, 페어인 NIC와 터널링 통신을 함

![image](https://user-images.githubusercontent.com/69254943/202955907-f203792e-3453-4d24-b734-8e770b2ee78c.png)

⇒ 가상 NIC(vethxxx)는 컨테이너에서는 eth0으로 보임

<br>

### **Docker 컨테이너와 외부 네트워크간 통신은 어떻게?!**

- Docker 컨테이너와 외부 네트워크가 통신을 할 때는 가상 브리지인 docker0과 호스트 OS의 물리 NIC에서 패킷을 전송하는 장치가 필요하다.
- Docker에서는 NAPT 기능을 사용해 연결한다.
- NAPT(Network Address Port Translation)
    - 하나의 IP 주소를 여러 컴퓨터가 공유하는 기술로, IP 주소와 포트 번호를 변환하는 기능임
    - private IP 주소와 global IP 주소를 투과적으로 상호변환하는 기술로, TCP/IP의 포트 번호까지 동적으로 변화하기 때문에 하나의 글로벌 IP 주소로 여러 대의 머신이 동시에 연결할 수 있음
    - Docker에서는 NAPT에 Linux의 iptables를 사용하고 있음
- Docker에서 해당 기능을 사용할 때는 컨테이너 시작 시 컨테이너 안에서 사용하고 있는 포트를 가상 브리지인 docker0에 대해 개방한다.
    - ex. 컨테이너 시작 시에 컨테이너 안의 웹 서버가 사용하는 80번 포트를 호스트 OS의 8080번 포트로 전송하도록 설정함
        - 외부 네트워크에서 호스트 OS의 8080번 포트에 액세스하면, 컨테이너 안의 80번 포트로 연결됨

![image](https://user-images.githubusercontent.com/69254943/202955945-301d0bc6-00b8-4c82-8542-26843aaa1b78.png)

<br>

### 💡 NAT와 IP 마스커레이드(NAPT)의 차이

- private IP 주소와 global IP 주소를 변환하여 private IP 주소가 할당된 컴퓨터에 대해 인터넷 액세스를 가능하게 할 때 사용하는 기술로는 NAT와 NAPT가 있다.

**NAT(Network Address Translation)**

- private IP 주소가 할당된 클라이언트가 인터넷상에 있는 서버에 액세스할 때 NAT 라우터는 클라이언트의 private IP 주소를 NAT가 갖고 있는 public IP 주소로 변환하여 요청을 송신한다. 응답은 NAT 라우터가 수신처를 클라이언트의 private IP 주소로 변환하여 수신한다.
    - 이러한 주소 변환에 의해 private 네트워크상의 컴퓨터와 외부 인터넷상의 서버 간의 통신이 성립된다.
- NAT의 경우 public IP 주소와 private IP 주소를 1:1로 변환하기 때문에 동시에 여러 클라이언트가 동일한 외부 인터넷상의 서버와 통신할 수 없다.

![image](https://user-images.githubusercontent.com/69254943/202956002-553edd9f-8d79-4de6-a613-7dadea8b1ce1.png)

**NAPT (Network Address Port Translation)**

- private IP 주소와 함께 포트 번호도 같이 변환하는 기술이다. ⇒ 동일한 내부 네트워크상의 클라이언트를 식별하기 위함
- private IP 주소를 public IP 주소로 변환할 때 private IP 주소별로 서로 다른 포트 번호로 변환한다.

![image](https://user-images.githubusercontent.com/69254943/202956069-87bfb79d-0dad-4760-8ff7-df7885d1777e.png)

→ 위와같이 클라이언트 A가 보낸 요청은 포트 번호 1500으로 하고, 클라이언트 B가 보낸 요청은 포트 번호 1600으로 함

→ 동시에 여러 클라이언트가 동일한 외부 인터넷상의 서버와 통신할 수 있음