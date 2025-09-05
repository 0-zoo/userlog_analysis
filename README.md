# 👥 팀원 소개

| 황지환 | 이영주 |
|--------|--------|
| [![황지환](https://github.com/jihwan77.png?size=100)](https://github.com/jihwan77) | [![이영주](https://github.com/0-zoo.png?size=100)](https://github.com/0-zoo) |

> 👉 각 이미지 클릭 시 해당 팀원의 GitHub 프로필로 이동합니다.

---


# 🔍 Linux SSH Log Analysis & Automation

---

## 1. 프로젝트 개요

본 프로젝트는 리눅스 서버의 보안 로그(`/var/log/auth.log`)를 기반으로,
**SSH 로그인 성공/실패 기록을 수집·분석**하고 이를 자동화하여 관리하는 실습 프로젝트입니다.

- **분석 대상**: SSH 원격 로그인 (성공 / 실패)
- **활용 도구**: `awk`, `cron`, Bash 스크립트, `fail2ban`
- **핵심 목표**:
    1. 로그 기록 파일로부터 유의미한 데이터 추출
    2. CSV 유사 형식(`.log` 텍스트)으로 변환하여 재활용 가능성 확보
    3. `cron`을 통한 주기적 실행으로 자동화 구축
    4. 보안 관점(비정상 로그인 시도 탐지 및 자동 차단) 학습

---

## 2. 프로젝트 단계

### (1) 주제 선정 및 설계

- **주제**: SSH 로그인 로그 자동 분석
- **설계**:
    - 로그인 실패 / 성공 로그를 각각 별도 파일로 저장
    - 실패 로그: 사용자, IP, 실패 사유 추출
    - 성공 로그: 사용자, IP, 인증 방식 추출
    - 주기적 실행(`cron`) 및 확장 기능으로 보안 강화

---

### (2) 실습 자료 구축

- 로그 원천: `/var/log/auth.log`
- 실습 환경: Ubuntu 24.04 (테스트용 VM)
- 디렉토리 구조:
    
    log_project/
    
    ├── scripts/
    
    │   └── log_analysis.sh
    
    └── logs/
    
    ├── login_failed.log
    
    ├── login_success.log
    

---

### (3) 메인 스크립트 (`log_analysis.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail
BASE="/home/vboxuser/log_project"
LOG_DIR="$BASE/logs"
AUTH="/var/log/auth.log"

mkdir -p "$LOG_DIR"

# 1) 로그인 실패 로그
{
echo "Timestamp,Host,Process,User,IP,Reason"
awk '
{
  ts=$1; host=$2; proc=$3;    # ISO8601, 호스트, sshd[pid]:
  $1=$2=$3=""; msg=$0;        # 나머지는 메시지 부분
  if (match(msg, /Failed password for invalid user ([^ ]+) from ([0-9a-fA-F:\\.]+)/, m)) {
    printf "%s,%s,%s,%s,%s,%s\\n", ts,host,proc,m[1],m[2],"invalid user"
  }
  else if (match(msg, /Failed ([A-Za-z0-9\\-\\/]+) for ([^ ]+) from ([0-9a-fA-F:\\.]+)/, m)) {
    printf "%s,%s,%s,%s,%s,%s\\n", ts,host,proc,m[2],m[3],"failed"
  }
}' "$AUTH"
} > "$LOG_DIR/login_failed.log"

# 2) 로그인 성공 로그
{
echo "Timestamp,Host,Process,User,IP,Method"
awk '
{
  ts=$1; host=$2; proc=$3;
  $1=$2=$3=""; msg=$0;
  if (match(msg, /Accepted ([A-Za-z0-9\\-\\/]+) for ([^ ]+) from ([0-9a-fA-F:\\.]+)/, m)) {
    printf "%s,%s,%s,%s,%s,%s\\n", ts,host,proc,m[2],m[3],m[1]
  }
}' "$AUTH"
} > "$LOG_DIR/login_success.log"

```

---

### (4) 자동화 (cron 적용)

1분 단위 테스트 실행 (실제 운영 시엔 1시간 또는 1일 단위):

```
* * * * * /home/vboxuser/log_project/scripts/log_analysis.sh
```

실행 여부 확인:

```bash
ls -l /home/vboxuser/log_project/logs
tail -n 5 /home/vboxuser/log_project/logs/login_failed.log
```

---

## 3. 실행 예시

### 기존 auth.log 파일 내용
<img width="1592" height="642" alt="image" src="https://github.com/user-attachments/assets/9894e3bf-c545-403c-82fc-e5d277bfe09a" />



### 로그인 실패 로그 (`login_failed.log`)

```
Timestamp,Host,Process,User,IP,Reason
2025-09-04T12:44:41.881219+09:00,myserver00,sshd[866]:,ubuntu,10.0.2.2,invalid user
2025-09-05T09:13:48.938786+09:00,myserver00,sshd[385093]:,ubuntu,10.0.2.2,invalid user

```
<img width="881" height="180" alt="image" src="https://github.com/user-attachments/assets/96a73e7a-cf62-479a-929f-ff908e93ea63" />



### 로그인 성공 로그 (`login_success.log`)

```
Timestamp,Host,Process,User,IP,Method
2025-09-05T11:36:54.343709+09:00,myserver00,sshd[433690]:,ubuntu,::1,password

```
<img width="856" height="140" alt="image" src="https://github.com/user-attachments/assets/f7ba9f5c-b265-4a95-a2c8-f29f3a92a2aa" />

## 4. fail2ban 설치
```
sudo apt update
sudo apt install -y fail2ban
```

### fail2ban 구성 파일 생성 (jail.local)
```
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```
### fail2ban 구성 파일 설정

```
[sshd]
enabled = true          # SSH 감시 활성화
port    = ssh           # SSH 포트 (다른 포트를 쓰면 수정)
filter  = sshd
logpath = /var/log/auth.log
maxretry = 5            # 허용 실패 횟수
bantime  = 600          # 차단 시간 (초) → 600초 = 10분
findtime = 600          # 실패 누적 카운트 유지 시간
```

### 테스트
---
<img width="424" height="280" alt="image" src="https://github.com/user-attachments/assets/0f4ab679-1821-44d3-bf52-6bec8357d4c2" />


### 실제 ban 모습 확인

<img width="645" height="176" alt="image" src="https://github.com/user-attachments/assets/e1fb05a1-2aa8-47ca-8462-db8e5e04ec25" />


---

## 5. 향후 계획

- **일일 리포트 자동 생성**
    - 실패 TOP 10 IP / 지역
    - 성공 사용자 TOP 10
    - 인증 방식 비율

- **시각화**
    - `.log` 파일을 기반으로 Excel, Grafana 등으로 대시보드 구현
- **fail2ban**
    - 현재는 auth.log를 통해 로그를 보고 판단하지만, 직접 생성한 log파일도 conf파일 규격에 맞게 수정하여 사용 가능하도록 구현

---

## 6. 학습 포인트

- 리눅스 보안 로그(`/var/log/auth.log`) 구조 이해
- `awk`를 활용한 텍스트 로그 파싱 및 가공
- `cron`으로 주기적 작업 자동화
- `fail2ban`을 통한 자동 IP 차단 기초
- 보안 로그 모니터링 및 운영 자동화 기초

---
