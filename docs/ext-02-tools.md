# 부록 B : 도커로 개발을 지원하는 도구 및 서비스

## 1. 인하우스 도커 레지스트리 구축

- 도커 이미지를 보관하기에는 도커 허브나 Quay.io 같은 퍼블릭 레지스트리 서비스를 이용하는 것이 편리하다.
- 퍼블릭 레지스트리를 사용 불가한 경우
    - 퍼블릭 레지스트리의 지나치게 긴 응답 시간
    - 보안 측면의 문제
        - 운영 환경에 배포하기 위해 개발한 어플리케이션 이미지를 퍼블릭 레지스트리에 저장할 수는 없다.
- 인하우스 도커 레지스트리
    - 레지스트리를 직접 구축하여 어떤 보안 정책이라도 만족시킬 수 있다.
    - 컨테이너를 배포하는 호스트와 가까이 도커 레지스트리가 위치할 것이므로 배포 속도 또한 개선할 수 있다.
- Registry(Docker Distribution)
    - 도커에서 인하우스 도커 레지스트리를 구축할 수 있도록 배포하는 도구
    - Docker Distribution과 함께 제공된다.
    - 오픈소스 소프트웨어이며 도커 컨테이너 형태로 실행할 수 있으므로 도커 호스트 역할을 할 운영 체제만 있으면 레지스트리 운영이 가능하다.
    - 3장의 docker swarm을 구축할 때 사용했다.
- 구축
    - Registry 설치(서버) : ubuntu 16.x
        - /etc/systemd/system/docker-container@registry.service
            ```
            [Unit]
            Descriptioni=Registry
            Requires=docker.service
            After=docker.service

            [Service]
            Restart=always
            ExecStart=/usr/bin/docker container run --rm --name registry -p 5000:5000 library/registry

            [Install]
            WantedBy=multi-user.target
            ```
        - 서비스로 활성화 한다.
            ```
            $ systemctl enable docker-container@registry.service
            $ systemctl start docker-container@registry.service
            ```
        - 곧 registry 컨테이너가 실행된다.
    - Registry에 이미지 등록하기
        - Registry는 HTTP API를 제공한다.
            ```
            $ curl http://localhost:5000/v2/_catalog
            ```
        - 태그를 부여해 기존 알파인 리눅스 이미지를 Registry에 등록한다.
            ```
            $ docker image pull alpine:3.7
            $ docker image tag alpine:3.7 localhost:5000/alpine:3.7
            $ docker image push localhost:5000/alpine:3.7
            ```
    - Registry에서 이미지 내려받기
        - Registry 컨테이너를 실행한 호스트에서 localhost:5000으로 접근하면 된다.
        - 그 외 호스트에서 Registry에서 접근하려면 도메인 부분을 다른 호스트에서 참조 가능한 IP 주소나 호스트명으로 바꿔야 한다.
        - docker image pull 명령은 기본적으로 HTTPS를 통해 레지스트리에 접근하기 때문에 Registry 역시 HTTPS를 활성화 하지 않으면 오류가 발생한다.
        - 외부에서 접근이 불가능한 내부 네트워크 안에서만 이미지가 전송된다면 굳이 HTTPS를 적용할 필요가 없다.
            - docker image pull 명령을 실행하는 쪽의 /etc/docker/daemon.json 파일에 HTTP 통신을 허용한다.
            - `Insecure registies`에 값을 추가한다.

## 2. 도커와 CI/CD 서비스 연동

- 도커를 사용한 어플리케이션 개발 과정
    - 구현 - 단위 테스트 - 어플리케이션 빌드 - 도커 이미지 빌드 - 도커 레지스트리 등록
    - 위 과정의 반복
    - 이 과정을 수작업으로 진행하면 실수가 발생하기 쉽다.
    - CI/CD 서비스에 이 과정을 맡기면 실수를 줄이고 시간도 적야할 수 있으므로 적절한 자동화 도입이 매우 중요하다.
        - CI : Continuous Integrate : 지속적 통합
        - CD : Continuous (Deploy | Delivery) : 지속적 배포
    - CI/CD 서비스의 예시 : Jenkins, CircleCI
- CircleCI
    - INTRO
        - SaaS(Software As A Service) 형태로 제공되는 CI/CD 서비스
        - 빌드 설정 역시 yaml 포맷으로 기술할 수 있다.
        - 이번에 다루는 내용은 CircleCI와 github를 사용한다.
    - github 계정으로 OAuth 인증을 거쳐 가입하고 로그인한다.
    - 프로젝트 추가
        - github에 리포지토리를 새로 추가하고 내용으로 Dockerfile과 main.py를 생성한다.
            - Dockerfile
                ```Dockerfile
                FROM python:3.6-alpine

                WORKDIR /app
                COPY main.py /app/main.py

                CMD ["python", "main.py"]
                ```
            - main.py
                ```python
                # -*- coding: utf-8 -*-
                import socket

                # simple-echo-server
                def serve(host='0.0.0.0', port=8080):
                    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                        s.bind((host, port))
                        s.listen()
                        while True:
                            conn, addr = s.accept()
                            with conn:
                                print('connected by', addr)
                                if conn.recv(1024):
                                    conn.sendall(b'hello python!!')

                if __name__ == '__main__':
                    serve()
                ```
        - 프로젝트가 추가되면 대상 저장소에 푸시가 일어날 때마다 웹훅이 CircleCI의 CI/CD 잡을 시작하게 한다.
        - 깃허브 조직 저장소를 추가하려면 깃 허브 설정에서 CircleCI의 Organization access 권한을 부여해야 한다.
    - 빌드 설정 파일 작성
        - python 저장소를 이용해 도커 이미지 빌드에 이르는 과정을 설정한다.
        - .circleci/config.yml
            ```yml
            version: 2.1
            jobs:
                build:
                    # CircleCI 컨테이너의 작업 디렉토리
                    working_directory: /app
                    # 잡을 실행하는 컨테이너의 도커 이미지
                    docker:
                        -   image: python:3.6-alpine
                    steps:
                        -   checkout    # 저장소에서 소스코드를 체크아웃
                        -   persist_to_workspace: # workspace를 후속 잡에서 사용할 수 있도록 유지
                                root: .
                                paths:
                                    -   .
                docker_build_push:
                    working_directory: /app
                    docker:
                        - image: docker:18.05.0-ce-dind # docker 명령을 사용할 수 있는 컨테이너
                    steps:
                        -   attach_workspace: # 저장해둔 workspace를 배치
                                at: .
                        -   setup_remote_docker: # docker image build를 수행할 데몬 준비
                                version: 18.05.0-ce
                        -   run: # 도커 이미지 빌드
                                name: docker image build
                                command: docker image build -t joonstudydocker/echo-python:latest .
                        -   run: # 빌드된 이미지 확인
                                name : docker image ls
                                command: docker image ls
                        -   run: # 도커 허브 로그인
                                name: docker login
                                command: docker login -u $DOCKER_USER -p $DOCKER_PASS # 프로젝트 설정에서 환경 변수를 저장할 수 있다.
                        -   run: # latest 이미지를 도커 허브에 등록
                                name: release latest
                                command: docker image push joonstudydocker/echo-python:latest
            workflows:
                version: 2
                build_and_push:
                    jobs: # workflows로 실행할 잡을 열거
                        -   build
                        -   docker_build_push:
                                requires: # build 잡 다음에 실행
                                    -   build
                                # master 브랜치인 경우에만 실행
                                # 다른 브랜치에서도 실행하려면 주석 처리하거나 제거
                                filters: 
                                    branches:
                                        only: master
            ```
    - CircleCI와 저장소를 연동하여 어플리케이션 테스트 및 도커 이미지 빌드와 등록을 대신해주는 CI/CD 환경을 구축하였다.
    - 참고
        - docker login 오류 - ubuntu16.x
            ```
            $ sudo apt install gnupg2 pass 
            $ gpg2 --full-generate-key
            $ gpg2 -k
            $ pass init "whatever key id you have"
            ```

## 3. ECS에서 AWS Fargate를 이용한 컨테이너 오케스트레이션