# 🔍 Linux SSH Log Analysis & Automation
---

# 👥 팀원 소개

| 황지환 | 이영주 |
|--------|--------|
| [![황지환](https://github.com/jihwan77.png?size=100)](https://github.com/jihwan77) | [![이영주](https://github.com/0-zoo.png?size=100)](https://github.com/0-zoo) |

> 👉 각 이미지 클릭 시 해당 팀원의 GitHub 프로필로 이동합니다.

---

## 1. 프로젝트 개요

이 프로젝트는 리눅스 서버에서 발생하는 보안 로그(`/var/log/auth.log`)를 기반으로,
**SSH 로그인 성공 및 실패 기록을 체계적으로 수집·분석하고 자동화된 보안 관리 체계를 구축**하는 것을 목표로 합니다.

### ✅ 목표

- **보안 관점**: 외부에서의 비정상 로그인 시도를 탐지하고, 일정 횟수 이상 실패 시 자동 차단하는 시스템을 갖추기 위함
- **운영 관점**: 관리자가 직접 로그 파일을 확인하지 않아도, 주기적인 자동 분석 및 리포트 생성으로 관리 효율성을 극대화
- **학습 관점**: `awk`, `cron`, `fail2ban`과 같은 리눅스 도구를 실무적으로 적용하고 자동화/보안 운영 능력 강화

---

## 2. 시스템 구축

- **운영 환경**: Ubuntu 24.04 (테스트 VM)
- **로그 원천**: `/var/log/auth.log`
- **구조**
    
    log_project/
    
    ├── scripts/
    
    │   └── log_analysis.sh    # 로그 수집 및 가공 자동화 스크립트
    
    └── logs/
    
    ├── login_failed.log   # 실패 기록 추출
    
    ├── login_success.log  # 성공 기록 추출
    

---

## 3. 사용 도구

| Tool | Purpose |
| --- | --- |
| **awk** | 리눅스에서 제공하는 텍스트 처리 도구로, `/var/log/auth.log` 안의 비정형 로그 데이터를 정규식 기반으로 파싱합니다. SSH 로그인 성공·실패 로그에서 사용자명, IP, 시간 같은 핵심 정보를 추출하고, 이를 CSV 유사 포맷으로 가공하여 후속 분석이 가능하도록 합니다. |
| **bash script** | 여러 명령어를 조합해 자동화 파이프라인을 구현하는 실행 단위입니다. `awk`로 가공한 데이터를 파일(`login_failed.log`, `login_success.log`)에 저장하고, 크론과 연동해 전체 분석 과정을 자동으로 수행할 수 있도록 합니다. |
| **cron** | 유닉스/리눅스의 스케줄러 도구로, 작성한 스크립트를 주기적으로 실행합니다. 예를 들어 1분, 1시간, 1일 단위로 자동 실행을 설정하여 관리자가 직접 실행하지 않아도 지속적인 로그 분석이 가능하도록 합니다. |
| **fail2ban** | SSH 로그인 실패가 일정 횟수 이상 발생할 경우 해당 IP를 자동 차단하는 보안 툴입니다. `auth.log`를 모니터링하며, 비정상 접근 시도를 탐지하면 `iptables` 또는 `ufw` 규칙을 수정해 일정 시간 동안 차단함으로써 능동적인 보안 방어를 제공합니다. |
| **Ubuntu auth.log** | 분석의 데이터 원천이 되는 보안 로그 파일입니다. SSH 로그인 성공/실패, sudo 명령 실행, PAM 인증 과정 등의 기록이 남는 위치이며, 본 프로젝트는 이 파일을 기반으로 사용자 접속 이력과 공격 시도를 추출합니다. |


---

## 4. 메인 스크립트 (`log_analysis.sh`)

`log_analysis.sh`는 이 프로젝트의 **핵심 엔진**으로, 

 `/var/log/auth.log` 파일을 읽어 SSH 로그인 시도의 **성공**과 **실패**를 각각 분리해 추출하고, 사람이 읽기 쉽고 후속 분석이 가능한 형태(`.log`)로 저장합니다.

### ✅ 주요 역할

1. **로그 파일 읽기**
    - 원천 로그(`/var/log/auth.log`)를 스캔
    - SSH 인증 관련 메시지 중 “Failed password”와 “Accepted”를 정규식으로 필터링
2. **실패 로그 가공 (`login_failed.log`)**
    - `awk`를 이용해 시간, 호스트명, 프로세스, 사용자명, IP, 실패 사유를 추출
    - 예: 비정상 계정(`invalid user`)과 일반 실패(`failed`)를 구분하여 저장
3. **성공 로그 가공 (`login_success.log`)**
    - SSH 접속이 성공한 경우, 시간, 호스트명, 프로세스, 사용자명, IP, 인증 방식(password/publickey 등)을 추출
4. **데이터 구조화**
    - 두 로그 파일 모두 CSV 유사 포맷(`,` 구분자)으로 저장되며, 다른 도구(Excel, Grafana 등)와 쉽게 연동 가능

### ✅ 실행 방식

- 스크립트는 cron 스케줄러를 통해 **자동 실행** 됩니다.
- 실행될 때마다 기존 `login_failed.log`, `login_success.log` 파일이 덮어쓰여 새로운 결과가 기록됩니다.
- 추후 필요시 `>>`를 사용해 누적 저장으로 변경할 수 있습니다.
  
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

## 5. 자동화 (cron 적용)

`cron`은 리눅스에서 가장 널리 쓰이는 작업 스케줄러로, 특정 시간 간격에 따라 명령어나 스크립트를 자동으로 실행합니다.

이를 통해 관리자가 매번 직접 스크립트를 실행하지 않아도, **지속적인 로그 분석과 보안 모니터링이 자동으로 수행**됩니다.

### ✅ cron 설정 예시

테스트 환경에서는 아래 설정을 통해 `log_analysis.sh`를 **매 1분마다 실행**하도록 설정하였습니다.

```bash
* * * * * /home/vboxuser/log_project/scripts/log_analysis.sh
```

실제 운영 환경에서는 1시간(`0 * * * *`) 혹은 1일 단위(`0 0 * * *`)로 설정하는 것이 일반적입니다.

### ✅ 동작 확인

1. 로그 디렉토리 내 파일 변경 시간 확인
    
    ```bash
    ls -l /home/vboxuser/log_project/logs
    ```
    
    → 실행 시각에 맞춰 `login_failed.log`와 `login_success.log`의 수정 시간이 갱신됩니다.
    
2. 로그 내용 확인:
    
    ```bash
    tail -n 5 /home/vboxuser/log_project/logs/login_failed.log
    ```
    
    → 최근 실패 시도가 제대로 기록되었는지 확인할 수 있습니다.

---

## 3. 실행 예시

### 기존 auth.log 파일 내용
<img width="1592" height="642" alt="image" src="https://github.com/user-attachments/assets/9894e3bf-c545-403c-82fc-e5d277bfe09a" />

아래는 `/var/log/auth.log`에서 실제로 발생한 SSH 로그인 시도를 본 프로젝트의 스크립트로 분석한 결과 예시입니다.

---

### 🔴 로그인 실패 로그 (`login_failed.log`)

- **의미**: 잘못된 계정이나 비밀번호를 사용해 SSH 접속을 시도한 흔적을 기록합니다.
- **필드 구성**: `Timestamp`, `Host`, `Process`, `User`, `IP`, `Reason`

```
Timestamp,Host,Process,User,IP,Reason
2025-09-04T12:44:41.881219+09:00,myserver00,sshd[866]:,ubuntu,10.0.2.2,invalid user
2025-09-05T09:13:48.938786+09:00,myserver00,sshd[385093]:,ubuntu,10.0.2.2,invalid user
```
<img width="881" height="180" alt="image" src="https://github.com/user-attachments/assets/96a73e7a-cf62-479a-929f-ff908e93ea63" />

➡️ 위 예시는 `10.0.2.2` IP에서 `ubuntu` 사용자로 로그인 시도가 있었으나, 비밀번호 인증에 실패하여 *invalid user*로 분류된 사례입니다.

---

### 🟢 로그인 성공 로그 (`login_success.log`)

- **의미**: SSH 접속이 성공적으로 인증된 경우의 기록입니다.
- **필드 구성**: `Timestamp`, `Host`, `Process`, `User`, `IP`, `Method`

```
Timestamp,Host,Process,User,IP,Method
2025-09-05T11:36:54.343709+09:00,myserver00,sshd[433690]:,ubuntu,::1,password
```
<img width="856" height="140" alt="image" src="https://github.com/user-attachments/assets/f7ba9f5c-b265-4a95-a2c8-f29f3a92a2aa" />

➡️ 위 예시는 서버 `myserver00`에 `ubuntu` 계정으로 **로컬 IP(::1)** 에서 **비밀번호(password)** 방식으로 SSH 접속이 성공한 사례를 보여줍니다.

---

👉 이렇게 실패/성공 로그를 각각 분리하여 저장하면,

- 보안 위협(IP 기반 공격, 무차별 대입 시도 등)을 빠르게 탐지할 수 있고,
- 정상 사용자 접속 패턴을 별도로 분석하여 운영 관리에 활용할 수 있습니다.

---

## 7. 보안 강화 (fail2ban 적용)

로그 분석만으로는 공격 시도를 "탐지"하는 데에 그치지만, **fail2ban**을 적용하면 공격 IP를 자동으로 차단하여 실시간 방어가 가능합니다.

### 설치

```bash
sudo apt update
sudo apt install -y fail2ban

```

→ `fail2ban`은 기본적으로 `iptables` 또는 `ufw` 방화벽 규칙을 수정하여 공격자를 자동 차단합니다.

---

### SSH 감시 설정 (`/etc/fail2ban/jail.local`)

설정 파일을 복사 후 편집합니다:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

```

SSH 관련 섹션을 아래와 같이 수정합니다:

```
[sshd]
enabled  = true          # SSH 감시 활성화
port     = ssh           # SSH 기본 포트 (22번, 다를 경우 수정)
filter   = sshd          # 기본 제공되는 sshd 필터 사용
logpath  = /var/log/auth.log
maxretry = 5             # 로그인 실패 5회 이상 시
bantime  = 600           # 600초(10분) 동안 IP 차단
findtime = 600           # 600초(10분) 안에서 누적 실패 횟수 계산

```
### 테스트
로그인 실패 IP 차단 태스트
---
<img width="424" height="280" alt="image" src="https://github.com/user-attachments/assets/0f4ab679-1821-44d3-bf52-6bec8357d4c2" />

실제로 해당 IP가 차단된 것을 확인할 수 있습니다.

<img width="645" height="176" alt="image" src="https://github.com/user-attachments/assets/e1fb05a1-2aa8-47ca-8462-db8e5e04ec25" />

---

### 효과

- 특정 IP에서 **5회 이상 로그인 실패** 발생 시, 자동으로 차단됩니다.
- 차단된 IP는 `iptables` 또는 `ufw` 규칙에 추가되어 **10분간 접속 불가** 상태가 됩니다.
- 관리자는 매번 로그를 확인하지 않아도 자동으로 방어가 이루어지므로, **운영 안정성과 보안 수준**을 크게 높일 수 있습니다.
- 예시

```bash
sudo fail2ban-client status sshd
```

→ 현재 차단된 IP 목록과 차단 횟수를 확인할 수 있습니다.

---

👉 이처럼 `fail2ban`은 본 시스템에서 단순 모니터링을 넘어, 탐지(Detection) → 대응(Response)까지 자동화하는 보안 체계의 핵심 도구로 활용됩니다. 이를 통해 관리자는 자동 방어 체계로 서버를 보다 안정적으로 운영이 가능해집니다.

---

## 8. 향후 계획

본 프로젝트는 현재 SSH 로그인 로그를 **수집·분석 → 자동화 → 보안 강화**까지 구축한 상태입니다. 앞으로는 다음과 같은 방향으로 고도화를 진행할 예정입니다.

### 1) 일일 리포트 자동 생성

- **실패 TOP 10 IP**: 무차별 대입 공격(Brute Force) 시도를 한 상위 IP 확인
- **실패 지역 분석**: GeoIP 데이터베이스와 연계하여 공격 발생 지역 시각화
- **성공 사용자 TOP 10**: 실제 서버 접근이 잦은 주요 사용자 모니터링
- **인증 방식 비율**: password / publickey 사용 현황을 비교해 보안 정책 개선 근거 확보

👉 cron 작업을 일 단위로 확장하여 매일 자동 리포트를 생성하고, 이메일 알림 또는 슬랙(Slack) 웹훅 연동이 가능합니다.

---

### 2) 시각화 및 대시보드 구축

- 현재는 텍스트 기반 `.log` 파일이지만, 추후 CSV/JSON 변환을 통해 **Grafana, Kibana, Excel** 등과 연동
- 공격 추세, 사용자 활동 패턴을 **차트·그래프** 형태로 시각화
- 관리자 대시보드를 통해 실시간 보안 모니터링 환경 구축

---

### 3) fail2ban 고도화

- 현재는 `/var/log/auth.log`만 참조하여 IP 차단
- 향후에는 본 프로젝트에서 생성된 `login_failed.log` 파일을 **fail2ban 전용 필터**로 활용할 수 있도록 규격화
- 임계치 초과 시 자동 차단 후, 해당 이벤트를 별도 로그에 기록하여 추적 가능하도록 확장

---

### 4) 운영 최적화 및 확장

- **다중 서버 환경 적용**: 여러 서버에서 로그를 수집해 중앙에서 통합 분석
- **경량화**: 불필요한 로그 제외 및 필드 최적화로 성능 개선
- **보안 정책 개선**: 분석 결과를 토대로 SSH 포트 변경, 공개키 인증 강제화 등 보안 강화 조치 적용

👉 단순 분석을 넘어, **자동화 + 시각화 + 실시간 방어**까지 이어지는 **보안 로그 운영 체계**를 완성하는 것이 최종 목표입니다.

---

## 9. 최적화 서버 운영을 위한 고찰

이번 프로젝트는 단순히 SSH 로그인 로그를 수집하는 수준을 넘어, **자동화와 보안 강화를 결합한 운영 관리 모델**을 실습하는 데 목적이 있었습니다.

첫째, 단순한 로그 확인 작업을 사람이 수동으로 하는 것은 한계가 있음을 깨달았습니다. 보안 위협은 예고 없이 반복적으로 발생하기 때문에, **자동화된 보안 관리 체계**가 반드시 필요합니다.

둘째, `awk`, `cron`, `fail2ban`과 같은 리눅스 네이티브 도구는 가볍고 단순하지만, 적절히 조합하면 충분히 실무적인 보안 운영 환경을 만들 수 있다는 점을 확인했습니다. 이는 별도의 고가 보안 솔루션 없이도 **현업에서 빠르게 적용 가능한 대안**이 될 수 있습니다.

셋째, 반복적인 로그 점검과 차단 작업을 자동화하면서, 관리자의 **운영 효율성**이 크게 개선될 수 있음을 체감했습니다. 이는 운영자가 더 중요한 보안 정책 설계나 최적화 작업에 집중할 수 있게 만듭니다.

마지막으로, 이 구조는 단일 서버뿐 아니라 다수의 서버 환경에서도 확장 가능하며, 경량화된 보안 모니터링 모델로 발전시킬 수 있습니다. 즉, 향후 대규모 인프라 운영 시에도 적용 가능한 **확장성 있는 보안 자동화 프레임워크로 이어질 수 있음을 확인했습니다.**
