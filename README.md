## 📂 로그 파일 종류 및 언제 봐야 하는가

> 🔑 **유지보수 핵심 원칙**
> 오류가 발생했을 때 세 가지 로그를 무작정 다 열지 말고,
> **"지금 어떤 상황인가"** 를 먼저 판단한 뒤 해당 로그부터 확인하세요.

---

### 1️⃣ catalina.out — 톰캣 서버 자체 로그

```
경로: $CATALINA_HOME/logs/catalina.out
```

#### ✅ 이 로그를 봐야 하는 상황

| 상황 | 구체적인 증상 |
|---|---|
| 톰캣이 아예 안 뜰 때 | startup.sh 실행해도 프로세스가 올라오지 않음 |
| 서버가 갑자기 죽었을 때 | 운영 중 톰캣 프로세스가 사라짐 |
| JVM 관련 오류 | OutOfMemoryError, StackOverflowError 발생 |
| 설정 파일 오류 | server.xml, context.xml 수정 후 기동 실패 |
| 톰캣 버전업 / 이전 후 | 환경 변경 뒤 첫 기동 시 문제 발생 |

#### ❌ 이 로그로 해결 안 되는 상황

- 특정 앱만 안 뜰 때 → `localhost.log` 확인
- 특정 URL이 404/500 날 때 → `access_log` 확인

#### 🔍 실전 확인 명령어

```bash
# 실시간 확인
tail -f $CATALINA_HOME/logs/catalina.out

# SEVERE(심각) 오류만 추출
grep "SEVERE" $CATALINA_HOME/logs/catalina.out | tail -30

# Exception 발생 위치 + 스택트레이스 같이 보기
grep -A 20 "Exception" $CATALINA_HOME/logs/catalina.out | tail -60
```

---

### 2️⃣ localhost.yyyy-mm-dd.log — 웹 애플리케이션 로그

```
경로: $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log
```

#### ✅ 이 로그를 봐야 하는 상황

| 상황 | 구체적인 증상 |
|---|---|
| WAR 배포 후 앱이 안 뜰 때 | catalina.out엔 이상 없는데 앱 접속이 안 됨 |
| 배포는 됐는데 특정 기능 오류 | 일부 페이지만 500 에러, 나머지는 정상 |
| 앱 초기화 오류 | DB 연결, Bean 생성 실패 등 기동 시점 오류 |
| 라이브러리 충돌 의심 | jar 버전 변경 후 문제 발생 |
| catalina.out에 단서가 없을 때 | 서버는 정상인데 앱만 이상할 때 2순위로 확인 |

#### ❌ 이 로그로 해결 안 되는 상황

- 톰캣 자체가 안 뜰 때 → `catalina.out` 확인
- 누가 어떤 URL을 요청했는지 → `access_log` 확인

#### 🔍 실전 확인 명령어

```bash
# 오늘 날짜 로그 실시간 확인
tail -f $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log

# 배포 실패 원인 추적
grep -i "SEVERE\|ERROR\|Exception" $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log

# 특정 앱(컨텍스트) 관련 로그만 보기
grep "/myapp" $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log
```

---

### 3️⃣ localhost_access_log.yyyy-mm-dd.txt — HTTP 요청/응답 로그

```
경로: $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt
```

#### ✅ 이 로그를 봐야 하는 상황

| 상황 | 구체적인 증상 |
|---|---|
| 404 에러 발생 | 특정 URL 접근 시 Not Found |
| 500 에러 발생 | 어떤 요청에서 에러가 났는지 특정 필요 |
| 특정 시간대 장애 원인 파악 | "오후 2시에 갑자기 느려졌어요" 같은 제보 |
| 외부 IP 접근 확인 | 특정 IP가 요청을 보냈는지 확인 |
| 응답 속도 저하 의심 | 요청별 처리 시간 확인 (포맷 설정 시) |

#### ❌ 이 로그로 해결 안 되는 상황

- 오류의 원인(코드 레벨) → `catalina.out` 또는 `localhost.log` 확인
- 톰캣/앱 기동 문제 → 위 두 로그 확인

#### 🔍 실전 확인 명령어

```bash
# 오늘 404 요청 전체 보기
grep " 404 " $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt

# 500 에러 발생 시각과 URL 확인
grep " 500 " $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt | tail -20

# 특정 IP 접근 기록 확인
grep "192.168.1.100" $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt

# 특정 시간대 요청 확인 (예: 14시)
grep "\[28/Apr/2026:14:" $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt
```

---

### 🗺️ 상황별 로그 선택 흐름

```
오류 발생
    │
    ├─ 톰캣 프로세스 자체가 없거나 기동 실패?
    │       └─→ catalina.out ✅
    │
    ├─ 톰캣은 떠 있는데 특정 앱/기능이 안 됨?
    │       └─→ localhost.log ✅
    │
    └─ 어떤 요청에서 문제가 생겼는지 모르겠음?
            └─→ access_log ✅
                    │
                    └─ URL/시각 특정 후
                       → catalina.out 또는 localhost.log에서
                         해당 시각 기준으로 원인 추적
```
