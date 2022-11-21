# 5.2 Dockerfile의 빌드와 이미지 레이어

- Dockerfile을 빌드하면 Dockerfile에 정의된 구성을 바탕으로 한 Docker 이미지를 작성할 수 있다.

<br>

## 📌 Dockerfile로부터 Docker 이미지 만들기

- docker build 명령어를 사용한다.

    ```bash
    $ docker build -t [생성할 이미지명]:[태그명] [Dockerfile의 위치]
    ```

- 예시

    ```
    # Dockerfile
    FROM centos:centos7
    ```

    ```bash
    $ docker build -t sample:1.0 /home/docker/sample
    $ docker build -t sample:2.0 /home/docker/sample
    ```

    - Dockerfile에 명시한 베이스 이미지가 로컬에 없으면, Docker 리포지토리에서 다운로드함

  ![image](https://user-images.githubusercontent.com/69254943/202996320-66b57388-9486-4a6f-a507-7bff2fc84bc0.png)

  ⇒ IMAGE ID가 동일한 이미지는 그 실체도 똑같음

<br>

### 💡 중간 이미지의 재이용

- Docker는 이미지를 빌드할 때 자동으로 중간 이미지를 생성한다.
- 다른 이미지를 빌드할 때 중간 이미지를 내부적으로 재이용하여 빌드를 고속으로 수행한다.
- 이미지를 재이용하고 있을 때는 빌드 로그에 ‘Using Cache’라고 표시된다.

<br>

## 📌 Docker 이미지의 레이어 구조

- Dockerfile을 빌드하여 Docker 이미지를 작성하면 Dockerfile의 명령별로 이미지를 작성한다. 작성된 이미지는 레이어 구조로 되어 있다.
- 예시
    - 4개의 명령을 갖고 있는 Dockerfile의 예

    ```
    # STEP:1 Ubuntu (베이스 이미지)
    FROM ubuntu:latest
    
    # STEP:2 Nginx 설치
    RUN apt-get update && apt-get install -y -q nginx
    
    # STEP:3 파일 복사
    COPY index.html /usr/share/nginx/html/
    
    # STEP:4 Nginx 시작
    CMD ["nginx", "-g", "daemon off;"]
    ```

    - 해당 Dockerfile을 바탕으로 docker build 명령으로 이미지를 작성하면, Dockerfile의 명령 한 줄마다 이미지가 작성된다.

  ![image](https://user-images.githubusercontent.com/69254943/202996340-198d602d-9666-47b8-b729-a6d084c84ea8.png)

- 작성한 이미지는 다른 이미지와도 공유된다.
    - 공통의 베이스 이미지를 바탕으로 여러 개의 이미지를 작성한 경우, 베이스 이미지의 레이어가 공유됨
    - 이미지 레이어를 공유함으로써 Docker에서는 디스크의 용량을 효율적으로 이용함