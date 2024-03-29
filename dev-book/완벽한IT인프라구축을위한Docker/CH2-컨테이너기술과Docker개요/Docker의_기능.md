# 2.3 Docker의 기능

Docker에는 크게 3가지 기능이 있다.

- Build : Docker 이미지를 만드는 기능
- Ship : Docker 이미지를 공유하는 기능
- Run : Docker 컨테이너를 작동시키는 기능

<br>

## 📌 Docker 이미지를 만드는 기능 (Build)

- Docker는 애플리케이션의 실행에 필요한 프로그램 본체, 라이브러리, 미들웨어, OS나 네트워크 설정 등을 하나로 모아서 Docker 이미지를 만든다. Docker 이미지는 실행 환경에서 움직이는 컨테이너의 바탕이 된다.
- Docker에서는 하나의 이미지에는 하나의 애플리케이션만 넣어 두고, 여러 개의 컨테이너를 조합하여 서비스를 구축하는 방식을 권장하고 있다.
- Docker 이미지는 애플리케이션의 실행에 필요한 파일들이 저장된 디렉토리이다. Docker 명령을 사용하면 이미지를 tar 파일로 출력할 수 있다.
- Docker 이미지는
    - Docker의 명령을 사용하여 수동으로 만들 수도 있으며,
    - Dockerfile이라는 설정 파일을 만들어 그것을 바탕으로 자동으로 이미지를 만들 수도 있다.
    - CI/CD 관점에서는 코드에 의한 인프라 구성 관리를 생각하면 Dockerfile을 사용하여 관리하는 것이 바람직함
- Docker 이미지는 겹쳐서 사용 가능하다.
    - ex. OS용 이미지에 웹 애플리케이션용 이미지를 겹쳐서 다른 새로운 이미지를 만들 수 있다.

<br>

## 📌 Docker 이미지를 공유하는 기능 (Ship)

- Docker 이미지는 Docker 레지스트리에서 공유할 수 있다.
    - ex. Docker의 공식 레지스트리인 Docker Hub에서는 Ubuntu나 CentOS와 같은 Linux 배포판의 기본 기능을 제공하는 베이스 이미지를 배포하고 있다.
        - 이러한 베이스 이미지에 미들웨어나 라이브러리, 전개할 애플리케이션 등을 넣은 이미지를 겹쳐서 독자적인 Docker 이미지를 만들어 가는 것이다.

<br>

## 📌 Docker 컨테이너를 작동시키는 기능 (Run)

- Docker는 Linux 상에서 컨테이너 단위로 서버 기능을 작동시킨다. 이 컨테이너의 바탕이 되는 것이 Docker 이미지이다.
- 컨테이너의 기동, 정지, 파기는 Docker의 명령을 사용한다.
    - 다른 가상화 기술로 서버 기능을 실행시키려면 OS의 실행부터 시작하기 때문에 시간이 걸림
    - Docker의 경우 작동중인 호스트 OS 상에서 프로세스를 실행시키는 것과 동일한 속도로 빠르게 실행 가능함
- Docker는 하나의 Linux 커널을 여러 개의 컨테이너에서 공유하고 있다.
    - 컨테이너 안에서 작동하는 프로세스를 하나의 그룹으로 관리하고, 그룹마다 각각 파일 시스템이나 호스트명, 네트워크 등을 할당함
    - 그룹이 다르면 프로세스나 파일에 대한 액세스 불가능함
    - 이러한 구조를 사용해 컨테이너를 독립적인 공간으로서 관리한다. 이를 실행하기 위해 Linux 커널 기능(namespace, cgroups 등) 기술이 사용된다.

- 배포 환경에서는 모든 Docker 컨테이너를 한 대의 호스트 머신에서 작동시키는 일은 드물며, 시스템의 트래픽 증감이나 가용성 요건, 신뢰도 요건 등을 고려한 후에 여러 대의 호스트 머신으로 된 분산 환경을 구축한다.
    - 컨테이너 관리는 오케스트레이션 툴을 사용함
    - 오케스트레이션 툴은 분산 환경에서 컨테이너를 가동시키기 위해 필요한 기능을 제공하고 있음

<br>

## 📌 Docker 컴포넌트

![image](https://user-images.githubusercontent.com/69254943/202906805-7b783899-825d-4ed2-bae4-c4147e93bd99.png)

- Docker Engine - 핵심 기능
    - Docker 이미지를 생성하고, 컨테이너를 가동시키기 위한 핵심 기능 제공
    - Docker 명령 실행이나 Dockerfile에 의한 이미지 생성
- Docker Registry - 이미지 공개 및 공유
    - 컨테이너의 바탕이 되는 Docker 이미지 공개 및 공유를 위한 registry 기능
    - Docker의 공식 레지스트리 서비스인 Docker Hub도 Docker Registry를 사용
- Docker Compose - 컨테이너 일원 관리
    - 여러 개의 컨테이너 구성 정보를 코드로 정의하고, 명령을 실행하여 애플리케이션의 실행 환경을 구성하는 컨테이너들을 일원 관리하기 위한 툴