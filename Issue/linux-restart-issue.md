# 리눅스 배포 서비스가 정전 이후 동작하지 않은 문제

## 문제 상황

건물 자체 정전 이후 리눅스 서버는 재시작 되었으나, 서비스 재시작이 정상적으로 되지 않은 상황이 발생

## 원인

내부적으로 apache는 재시작이 되었으나, 설정한 `haproxy`, 제공중인 서비스들에 대한 재시작을 설정하지 않음

## 해결 방안

해결 방안으로는 3가지가 존재

1. **crontab**의 `@reboot`을 활용한 재시작 동작
2. **systemd**를 통한 재시작 스크립트의 서비스 등록
3. **/etc/rc.local** 파일에 스크립트 추가

이 중에서 이후 관리가 제일 편하다고 생각되는 2번 방법 채택

### systemd 서비스 등록

1. 서비스 재시작 할 스크립트 작성

    ```bash
    #!/bin/bash

    echo "[INFO] Starting full backlog stack: $(date)" >> /var/log/service-restart.log

    echo "[INFO] Restarting HAProxy..."
    sudo /usr/bin/systemctl restart haproxy

    echo "[INFO] Restarting backend_1"
    bash /srv/myapp/backend_1/run/shutdown.sh
    bash /srv/myapp/backend_1/run/startup.sh

    echo "[INFO] Restarting backend_2"
    bash /srv/myapp/backend_2/run/shutdown.sh
    bash /srv/myapp/backend_2/run/startup.sh
    ```

2. 해당 스크립트를 실행할 서비스 파일 작성

    ```ini
    [Unit]
    Description=spdto service
    After=network.target

    [Service]
    Type=simple
    ExecStart=/srv/haproxy/restart-service.sh
    Restart=always
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    ```

3. 서비스 파일을 `/etc/systemd/system/`에 저장

    ```bash
    sudo cp /srv/haproxy/restart-service.service /etc/systemd/system/
    ```

4. 서비스 등록 및 활성화

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable restart-service.service
    sudo systemctl start restart-service.service
    ```

## 요약

- 기존 리눅스 서비스에 대해서 단순 등록만 하고 후처리를 하지 않아 서버의 재시작 시 발생할 문제 확인
- 이 부분을 `systemd`를 등록하는 걸 기반으로 서비스 수정 및 재시작 자동화 처리
