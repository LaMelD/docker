# 부록 A : 보안

## 1. 공개된 도커 이미지의 안전성

- INTRO
    - 공개적인 생태계의 특성상 악의적인 이미지가 포함될 가능성을 배제할 수 없다.
    - 안전한 도커 이미지를 판단하는 기준을 파악하고 위험한 이미지를 가려내는 안목이 필요하다.
- 도커 허브
    - 도커의 기본적인 레지스트리이다.
    - docker search 명령으로 도커 허브에 등록된 이미지를 검색할 수 있다.
        - 정렬
            - 리포지토리의 스타 수(인기)
            - pull 횟수
            - OFFICIAL 항목 : `[OK]`라면 도커 허브의 공식 리포지토리
    - OFFICIAL
        - 해당 소프트웨어의 전문가나 보안 전문가와 협조해 이미지의 품질을 담보하기 위한 리뷰를 거친 것
        - 일정 수준의 보안성을 확보했다고 할 수 있다.
- Quay.io
    - CoreOS에서 운영하는 Quay.io
    - 사설 리포지토리 및 자동 빌드(automated build)를 지원한다.
    - 조직 관련 기능으로 팀 구성 및 롤 설정 기능을 제공하므로 도커 허브보다 훨씬 충실하게 팀 개발을 뒷바침할 수 있다.
    - Quay.io는 취약점 정보 데이터베이스이 CVE 정보를 기초로 설치된 패키지를 통해 취약점 포함 여부를 확인한다.
- Outro
    - 도커 허브에서 제공하는 공식 이미지나 Quay.io의 취약점 검사 등은 이미지의 안전성을 평가할 수 있는 중요한 지표이다.
    - 이 지표가 완벽한 것은 아니다.
    - 계획 중인 유즈케이스에 따라 어떤 위험이 따르는지 직접 확인하는 것이 중요하다.

## 2. 안전한 도커 이미지와 도커 운영 체제 꾸리기

- INTRO
    - 각 개발자가 안전한 도커 이미지를 만들고 보안 위험이 발생하지 않도록 운영 체계를 구축할 필요가 있다.
- Docker Bench for Security
    - 운영 환경에서 도커를 적용할 때 반드시 따라야 할 베스트 프랙티스의 준수 여부를 검사하는 도구
    - 도커 컨테이너의 취약점을 발견하는 데 유용하다.
    - docker/docker-bench-security 이미지로 제공된다.
    - 도커 호스트에서 실행
        ```
        $ docker container run -it --net host --pid host --userns host \
        --cap-add audit_control \
        -e DOCKER_CONTENT_TRUST=1 \
        -v /var/lib:/var/lib \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /usr/lib/systemd:/usr/lib/systemd \
        -v /etc:/etc \
        --label docker-bench_security \
        docker/docker-bench-security
        ```
    - 실행 중인 컨테이너뿐만 아니라 도커 환경 전반에 걸친 취약점을 검사한다.
    - Docker daemon configuration : 도커 데몬의 설정을 검사
        - TLS/SSL이 적용되지 않은 비보안 레지스트리를 사용하고 있다면 `2.4 - Ensure insecure registries are not used` 경고를 띄운다.
    - Container Images and Build File : Dockerfile에 대한 검사
        - 컨테이너 안에 어플리케이션 실행만을 위한 별도의 유저가 없다면 `4.1 - Ensure a user for the container has been created` 경고를 띄운다.
    - 운영 중 노출되기 쉬운 취약점을 탐지할 수 있으며 이 취약점을 해결하기 위한 베스트 프랙티스를 제시해 준다.
    - 탐지된 취약점을 충분히 이해하는 것이 중요하다.
- 컨테이너에 파일을 추가하면서 발생하는 위험
    - Dockerfile로 컨테이너에 파일을 추가하는 인스트럭션 : ADD, COPY
        - COPY
            - 호스트에서 파일이나 디렉터리를 복사해오는 기능을 수행
        - ADD
            - 지정한 URL에서 파일을 내려받아 추가할 수 있다.
            - 압축 파일을 자동으로 해제하는 기능이 있다.
        - 두 인스트럭션은 적절히 용도를 나눠 사용하지 않으면 취약점이 생길 수 있다.
    - URL에서 내려받은 파일이 항상 안전하다는 보장이 없다.
        - 파일이 웹 사이트 소유자에 의해 변경될 수 있다.
        - 악의를 품은 제삼자가 파일을 변조했을 수 있다.
        - 이미지가 빌드될 때마다 결과물도 달라질 것이므로 항등성도 담보할 수 없다.
    - ADD 인스트럭션을 사용하려면 추가하려는 파일의 안전성을 먼저 검증해야 한다.
    - Dockerfile에는 ADD 인스트럭션 대신 COPY 인스트럭션을 사용하는 버릇을 들이는 것이 좋다.
    - RUN
        - wget이나 curl을 이용해 추가하는 방법도 있다.
        - 호스트로부터 파일을 추가하는 게 아니라 파일 추가 과정 자체를 컨테이너 안으로 국한 시킬 수 있기 때문에 이식성이 뛰어나다.
        - ADD 인스트럭션과 마찬가지의 취약점을 가질 수 있다는 점을 이해하고 사용해야 한다.
    - 예제 : 프로비저닝 도구 Terraform
        - Terraform은 실행 파일을 포함하는 zip 파일과 함께 SHA256으로 생성한 체크섬 정보를 제공한다.
        - 적합성 검사 Dockerfile
            ```Dockerfile
            FROM alpine:3.7
            
            ENV VERSION=0.11.5

            RUN apk --no-cache add --virtual=build-devs curl gnupg

            # terraform과 배포 파일 체크섬 정보를 받음
            ADD https://releases.hashicorp.com/terraform/$VERSION/terraform_${VERSION}_linux__amd64.zip .
            ADD https://releases.hashicorp.com/terraform/$VERSION/terraform_${VERSION}_SHA256SUMS .
            ADD https://releases.hashicorp.com/terraform/$VERSION/terraform_${VERSION}_SHA256.sig .

            # Keybase에 공개된 Hashicorp사의 PGP 공개키 임포트
            RUN curl https://keybase.io/hashicorp/pgp_keys.asc | gpg --import

            # 체크섬의 전자서명 검증
            RUN gpg --verify terraform_${VERSION}_SHA256SUMS.sig terraform_${VERSION}_SHA256SUMS

            # 압축 파일의 체크섬 검증
            RUN cat terraform_${VERSION}_SHA256SUMS | grep linux_amd64 | sha256sum -cs
            RUN unzip terraform_${VERSION}_linux_amd64.zip
            RUN mv terraform /usr/local/bin/
            RUN terraform -v
            ```
        - Terraform을 개발한 Hashicorp는 Keybase에 PGP 공개키를 공개 중이다.
        - 체크섬 정보의 서명을 검증한다.
        - 서명이 확인되면 이 체크섬 정보를 zip 파일로부터 계산한 체크섬과 비교하는 방법으로 zip 파일의 정합성을 검증할 수 있다.
        - 체크섬 정보가 일치하지 않는다면 빌드는 실패 처리된다.
        - 외부로 공개되는 Dockerfile에 이런 방식으로 검증 로직을 추가하면 이미지의 신뢰성을 향상시킬 수 있다.
- 적절한 접근 제어
    - 컨테이너를 침입으로부터 보호
        - 컨테이너 중심의 개발 및 운영에서 가장 중요한 것은 외부인의 컨테이너 침입을 방지하는 것이다.
        - 대책
            - root로 로그인 금지.
            - AWS의 보안 그룹, GCP의 방화벽을 적절히 설정하고 도커 호스트에 대한 접근 제어.
            - VPN 적용 및 내부망 접근 제한.
    - /var/run/docker.sock의 컨테이너와의 공유 금지
        - 도커는 유닉스 도메인 소켓 /var/run/docker.sock을 -v 옵션으로 마운트하고 이를 이용해 원격지의 도커 데몬에 접근한다.
        - 이 소켓에 접근을 허용하면 컨테이너 생성 등의 작업을 원하는 대로 지시할 수 있다.
        - /var/run/docker.sock을 사용하는 이미지의 사용은 원칙적으로 금해야 한다.
        - 배포 중인 Dockerfile을 살펴보고 보호 대책을 마련해야 한다.
        - 특별한 사정이 없다면 호스트의 /var/run/docker.sock을 컨테이너 마운트하는 일은 하지 않는 것이 좋다.
    - 컨테이너 어플리케이션 실행용 사용자 추가
        - 도커 컨테이너 안에서 어플리케이션을 실행하는 기본 사용자 : root
        - 도커 이미지를 빌드하는 과정이라면 root 권한을 갖는 쪽이 편하다. 하지만 이것이 보안에 큰 구멍이 될 수 있다.
        - 도커는 호스트의 리소스를 컨테이너에서 공유하는 기술적 특징을 갖는다. 사용자 UID도 예외가 아니다.
        - 모종의 경로를 통해 컨테이너에 침입했다면 사실상 호스트의 root 권한을 탈취한 것과 같다.
        - 이런 문제를 방지하기 위해서 useradd 명령으로 어플리케이션 실행용 사용자를 생성하고 USER 인스트럭션에서 해당 사용자를 지정하면 된다.
            ```Dockerfile
            FROM golang:1.9

            RUN mkdir /echo
            COPY main.go /echo/

            RUN useradd gihyo
            USER gihyo

            CMD ["go", "run", "/echo/main.go"]
            ```
            - 컨테이너를 실행해 보면 gihyo 상용자가 어플리케이션을 실행하고 있음을 알 수 있다.
- 기밀정보 취급
    - 환경 변수를 이용하는 방법
        - 컨테이너의 환경 변수 값은 너무 간단히 노출된다.
            - env 명령만 사용할 수 있다면 모든 환경 변수 값을 볼 수 있다.
            - `docker container inspect <컨테이너 ID>`명령도 같다.
        - 환경 변수 방식을 적용하려면 권한 없는 제삼자가 도커 호스트나 도커 데몬을 조작할 수 없도록 적절한 접근 통제 정책을 먼저 실시해야 한다.
        - 제삼자에게 기밀정보가 유출되어도 상관없도록 기밀정보 자체를 암호화하고 어플리케이션 안에서만 복호화할 수 있도록 구현하는 것도 좋은 방법이다.
    - 기밀정보를 외부에서 받아오는 방법
        - 컨테이너 안에 있는 어플리케이션이 직접 외부에서 기밀정보를 받아오는 방법이 있다.
        - 컨테이너에 접근하더라도 사용자는 아무런 기밀정보를 볼 수 없으며 어플리케이션이 원격지에 위치한 기밀정보를 참조해오는 방식이다.
        - 이 기밀정보는 기계만 읽을 수 있는 형태이므로 기밀정보를 받아오는 통신 내용이 암호화되고 적절한 접근 통제만 되면 극히 안전한 방법이다.
        - 예시
            - 아마존 S3에 저장된 기밀정보에 접근하기 위해 IAM을 이용하는 방법이 있다.
            - Hashicorp가 개발한 Vault가 있다.
                - 기밀정보를 저장하는 도구.
                - 어플리케이션을 기밀정보를 전달받는 데이터베이스로 활용할 수 있다.
                - 암호화 기능, 기밀정보 열람 로그 기능, 무효화 기능 등을 제공한다.