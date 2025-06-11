# 리눅스 내 Spring 서비스들의 systemd 화

## 문제 상황

기존 배포 방식은 `startup.sh`와 `shutdown.sh`를 작성하고, 각 서비스 폴더들에 이동해 해당 스크립트를 실행
이를 각 위치 별로 해당 서비스들의 시작/종료를 실행하는 것이 번거롭다고 생각되어 수정을 진행
또한, haproxy를 통해 각 서비스를 이중화하여 관리하고 있었고, 이를 재시작 할때 포트를 열고 닫는 것도 자동화 진행 시도

## systemd 서비스 등록

1. `systemd` 서비스 파일 구성 - 위치는 /etc/systemd/system/에 위치
    예시로 `myapp@.service` 파일을 작성

    ```ini
    # 간단한 설명
    [Unit]
    Description=Generic Service Instance %i
    After=network.target

    [Service]
    User=ubuntu # 서비스 실행 사용자(현재 배당 받은 id 활용)
    EnvironmentFile=/srv/service-env/%i.env # 이후 각 서비스 별 환경 변수 파일

    WorkingDirectory=/  # 서비스 실행 디렉토리로 기본 root 설정

    # APP_HOME/ BACKEND_NAME 을 환경변수로 활용하여 서비스를 동적으로 등록 가능하도록 실행
    ExecStart=/usr/bin/java -Dlogging.config=${APP_HOME}/config/logback-spring.xml -jar ${APP_HOME}/${APP_NAME}.jar --spring.config.location=file:${APP_HOME}/config/application.yml

    # haproxy를 종료시에는 disable/ 실행 시에는 10초의 딜레이 이후 enable 처리
    ExecStop=/bin/bash -c '[ -n "${BACKEND_NAME}" ] && /srv/haproxy/haproxy_control.sh disable %i ${BACKEND_NAME} || echo "Skipping HAProxy disable (BACKEND_NAME not set)"'

    ExecStartPost=/bin/bash -c 'sleep 10; [ -n "${BACKEND_NAME}" ] && /srv/haproxy/haproxy_control.sh enable %i ${BACKEND_NAME} && sleep 3 || echo "Skipping HAProxy enable (BACKEND_NAME not set)"'
    
    SuccessExitStatus=143
    Restart=always
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    ```

2. 환경 변수 파일 작성 - `/srv/service-env/` 디렉터리에 각 서비스별 환경변수 파일 작성

    예시로 `myapp.env` 파일을 작성

    ```bash
    APP_HOME=/srv/myapp/backend_1
    APP_NAME=myapp
    BACKEND_NAME=myapp_pool
    ```

3. 서비스 파일 등록
  
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable myapp@myapp.service
    sudo systemctl start myapp@myapp.service
    ```

## 개선 사항

- 위의 동작을 통해 각 서비스들의 시작/종료를 `systemd`를 통해 관리
- 그로 인해, 각 서비스들이 서버의 재시작과 함께 자동으로 재시작이 되도록 설정
- 또한, `haproxy`의 포트 관리를 자동화 하여 이중화된 서비스들의 관리도 용이
- 서비스 추가 시에는 `env` 파일 작성 후, `systemd` 서비스 파일을 추가하는 과정을 통해 처리가 가능

## 느낀 점

> 기존에 전달 받은 방식은 `sh` 파일 작성을 통한 java 서비스의 시작/종료의 수동 처리였지만, `systemd`라는 항목을 통해서 서비스로 관리하는 방안을 알게 되었습니다.
> 이 방안이 효율적이라고 생각하게 된 것은 `cd`를 통해 이동하는 많은 명령어보단, `systemd`의 서비스 시작 종료를 하는 것이 효율적이라 판단하게 되었기 때문이었고,
> 하나의 서버 내에 여러 서비스를 관리하는 데 있어 효율적이라고 판단이 되었기 때문이었습니다.
> 기존의 방식이 나을 수 있지만, 새로운 관리 방안을 통해서 효율성이 높아진 것 같아 이후 리눅스 상에서는 `systemd`를 활용을 잘 하는 것 뿐 아니라, 다른 관리 방안을 채택할때 좀더 틀을 깨고 시도할 수 있을 것 같습니다.
