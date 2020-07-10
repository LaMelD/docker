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
            - `Insecure registries`에 값을 추가한다.

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

- ECS - Elastic Container Service
    - AWS는 지금까지 독자적인 컨테이너 오케스트레이션 서비스 ECS를 제공해왔다.
    - ESC는 서버 여러대를 이용해 컨테이너 오케스트레이션을 구현하는 방식이다.
    - ALB라는 AWS 서비스와 매끄럽게 연동된다.
        - ALB - Application Load Balancer
    - ECS는 최초 도입이 쿠버네티스보다 쉽기 때문에 프로젝트 규모와 무관하게 널리 사용된다.
        - 넷플릭스에서 대규모로 운영한 실적이 있다.
- AWS Fargate
    - 개발자가 서버나 클러스터를 직접 관리하지 않고 컨테이너를 실행하는 기술
    - ECS나 EKS에서는 서버관리, 클러스터 프로비저닝 등을 담당하는 서비스이다.
    - 컨테이너 오케스트레이션부터 서버 및 클러스터 관리에 이르기까지 가장 손이 많이 가는 작업을 제거했다는 점에서 획기적이라 할 수 있다.
    - AWS Fargate는 AWS Lambda의 뒤를 잇는 차세대 서버리스 기술로서도 주목받고 있다.
        - Lambda는 장시간 실행되는 배치 어플리케이션에는 적합하지 않은데, ECS는 이러한 약점을 보완할 수 있다.
        - AWS Fargate는 서버 인스턴스를 신경 쓰지 않고도 간단히 컨테이너 오케스트레이션을 실현할 수 있으므로 컨테이너 기술 및 서버리스 기술 양쪽에서 새로운 트랜드가 될 것이다.
- Fargate로 ECS 클러스터 구축
    - AWS console에서 ECS를 선택하면 Fargate 마법사 화면이 나타난다.
    - 시작을 눌러서 설정을 시작한다.
    - 1단계 : 컨테이너 및 작업
        - nginx를 선택한다.
    - 2단계 : 서비스
        - 로드 밸런서 유형을 Application Load Balancer를 선택한다.
    - 3단계 : 클러스터
        - Cluster 이름을 설정한다.
        - 필자는 joon으로 하겠다.
    - 4단계 : 검토
        - 지금까지의 내용을 검토한 뒤에 생성을 누른다.
    - 생성 뒤 확인
        - ECS 관련 리소스가 수 분 만에 생성된다.
    - 생성된 nginx에 접속
        - 생성한 뒤 로드 밸러서를 확인한다.
        - ECS 클러스터에 배포된 컨테이너는 모두 ALB라는 로드 밸런서를 통해 접근하므로 지금 만든 ALB의 DNS name을 복사한다.
        - 웹 브라우저에서 'http://복사한도메인'에 접근하면 nginx의 welcome 화면을 볼 수 있다.
- ECS를 조작해 어플리케이션 배포하기
    - ECS 리소스
        - (작음) Container Definition - Task Definition - Service - Cluster (큼)
        - Container Definition : 배포되는 각 컨테이너의 정의
        - Task Definition : 컨테이너의 집합인 Task의 정의
        - 쿠버네티스 파드의 개념이 두 단계로 나뉘어 있다고 생각하면 된다.
    - Task Definition 생성
        - 작업 정의 - 새로운 작업 정의 작성
        - FARGATE를 선택하고 다음을 누른다.
        - 테스크 정의 이름은 echo-task로 한다.
        - 네트워크 모드는 awsvpc를 선택하여 내부 IP(private)만 부여한다.
        - CPU 리소스
            - memory : 0.5GB
            - CPU : 0.25vCPU
        - 컨테이너 정의
            - echo
                - 이름 : echo
                - 이미지 : stormcat24/echo:latest
                - 포트 맵핑 : 8080
            - nginx
                - 이름 : nginx
                - 이미지 : stormcat24/nginx-proxy:latest
                - 포트 맵핑 : 80
                - 환경변수
                    - BACKEND_HOST : value : localhost:8080
    - 서비스 수정
        - 새 배포 적용을 체크한다.
        - 작업 개수는 1로 한다.
        - 이후 화면은 수정 없이 다음을 클릭한다.
        - Service 수정이 끝나면 이미 배포된 태스크가 정지되며 echo-task:1 태스크가 하나 배포된다.
        - 태스크 상세 정보를 확인하면 echo와 nginx 컨테이너를 포함한 태스크가 배포된 것을 알 수 있다.
    - ALB와 대상 그룹
        - Service는 ABL와 연동해 외부로부터 접근할 수 있다.
        - RUNNING 상태가 된 태스크는 ALB가 확인한 헬스 체크 결과가 healthy로 나와야 비로소 트래픽이 전달된다.
- OUTRO
    - 도커로 운영 환경을 꾸린다 해도 서버나 클러스터를 계속 신경써야 할 것이다.
    - 그러나 Fargate를 사용하면 한 번 설정하면 그 이후에는 자동으로 제어된다.
    - 이는 도커를 적용한 운영 환경을 쾌적하게 해주는 기술이다.