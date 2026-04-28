# 🪵 Tomcat 로그 파일 완전 가이드

> 언제 어떤 로그를 봐야 하는지 빠르게 찾기 위한 실무 레퍼런스

---

## 📁 Tomcat 로그 파일 종류

| 로그 파일 | 기본 위치 | 한 줄 요약 |
|---|---|---|
| `catalina.{date}.log` | `logs/` | Tomcat **서버 자체**의 시작/종료/JVM 오류 |
| `localhost.{date}.log` | `logs/` | **웹 애플리케이션(WAR)** 내부에서 발생한 예외 |
| `localhost_access_log.{date}.txt` | `logs/` | HTTP **요청/응답** 기록 (access log) |
| `host-manager.{date}.log` | `logs/` | Host Manager 앱 관련 로그 |
| `manager.{date}.log` | `logs/` | Manager 앱 관련 로그 |

---

## 🔍 로그별 상세 설명

### 1. `catalina.{date}.log` — Tomcat 서버 로그

**= Tomcat 프로세스 그 자체의 로그**

#### 이럴 때 확인
- Tomcat이 **시작/재시작/종료** 될 때
- `java.lang.OutOfMemoryError` 같은 **JVM 레벨 오류**가 의심될 때
- 서버 자체가 **뜨지 않을 때** (포트 충돌, 설정 오류 등)
- `server.xml`, `context.xml` 등 **설정 파일 파싱 오류**가 의심될 때
- Connector, Valve 등 **Tomcat 컴포넌트 초기화 실패** 시

#### 주요 내용
```
INFO: Server startup in 3456 ms
SEVERE: Error deploying web application directory [/webapps/myapp]
WARNING: Failed to create SessionFactory
```

#### 특징
- `System.out`, `System.err`로 출력한 내용도 여기에 기록됨
- Tomcat 내장 구성 요소의 라이프사이클 로그

---

### 2. `localhost.{date}.log` — 애플리케이션 예외 로그

**= 배포된 WAR/앱 내부에서 터진 예외의 로그**

#### 이럴 때 확인
- **특정 페이지에서 500 에러**가 날 때 ← ✅ 오늘 상황이 바로 이것!
- **등록, 수정, 조회 등 기능 동작 중** 오류가 났을 때
- `catalina` 로그에는 안 보이는데 오류가 날 때
- Spring, Hibernate, MyBatis 등 **프레임워크 레벨 예외**를 추적할 때
- `NullPointerException`, `SQLException` 등 **런타임 예외** 발생 시

#### 주요 내용
```
SEVERE: Servlet.service() for servlet [dispatcher] threw exception
java.lang.NullPointerException
    at com.example.service.BoardService.register(BoardService.java:42)
    ...
```

#### 특징
- `catalina`에서 안 잡히는 **앱 레벨 예외**가 여기에 기록됨
- 스택 트레이스(stack trace)가 상세하게 남아 디버깅에 최적
- `log4j`, `logback` 등 앱 자체 로거와는 **별개**임 (Tomcat이 직접 기록)

#### 실무 grep 명령어
```bash
# 오늘 날짜의 오류 마지막 50줄 확인
grep -nE "Exception|ERROR|SEVERE" \
  /tomcat/tomcat-instance1/logs/localhost.$(date +%Y-%m-%d).log \
  /tomcat/tomcat-instance2/logs/localhost.$(date +%Y-%m-%d).log | tail -50

# 특정 날짜 확인
grep -nE "Exception|ERROR|SEVERE" \
  /tomcat/logs/localhost.2025-07-01.log | tail -100

# 스택 트레이스 포함해서 보기 (전후 10줄)
grep -A 10 "SEVERE" /tomcat/logs/localhost.$(date +%Y-%m-%d).log
```

---

### 3. `localhost_access_log.{date}.txt` — HTTP 접근 로그

**= 누가 어떤 URL을 호출했는지의 기록**

#### 이럴 때 확인
- 특정 요청이 **실제로 서버에 도달했는지** 확인할 때
- **응답 코드**(200, 404, 500)와 **응답 시간** 확인이 필요할 때
- **트래픽 분석** 또는 **특정 IP의 요청 추적** 시
- 성능 이슈에서 **느린 요청**을 찾을 때
- 보안 감사, **비정상적인 접근 패턴** 감지 시

#### 주요 내용
```
192.168.1.10 - - [01/Jul/2025:14:32:01 +0900] "POST /board/register HTTP/1.1" 500 1234 5678
```
→ IP / 날짜·시간 / 메서드+URL / **응답코드** / 바이트 수 / **응답시간(ms)**

#### 실무 grep 명령어
```bash
# 500 에러 요청만 추출
grep " 500 " /tomcat/logs/localhost_access_log.$(date +%Y-%m-%d).txt

# 특정 URL 요청 확인
grep "/board/register" /tomcat/logs/localhost_access_log.$(date +%Y-%m-%d).txt

# 응답시간 느린 요청 찾기 (마지막 컬럼이 ms인 경우)
awk '{if ($NF > 3000) print}' /tomcat/logs/localhost_access_log.$(date +%Y-%m-%d).txt
```

---

## 🚦 상황별 로그 선택 가이드

```
오류 발생!
    │
    ├─ Tomcat 자체가 안 뜬다 / 배포 실패
    │       └─→ catalina.log
    │
    ├─ 특정 페이지/기능에서 500 에러
    │       └─→ localhost.log  ← 가장 먼저 볼 것!
    │
    ├─ 요청이 서버에 도달은 했나? 응답코드 확인
    │       └─→ localhost_access_log.txt
    │
    └─ 셋 다 이상 없는데 오류? 
            └─→ 앱 자체 로그 확인 (log4j / logback 설정 경로)
```

---

## 💡 실무 팁

### catalina vs localhost 헷갈릴 때
| 구분 | catalina | localhost |
|---|---|---|
| 기록 주체 | Tomcat 서버 | 웹 애플리케이션 |
| 주로 보는 상황 | 서버 기동 문제 | 기능 오류 디버깅 |
| 예외 종류 | JVM, 설정, 배포 오류 | NPE, SQL, 비즈니스 로직 오류 |

### 멀티 인스턴스 환경에서 한 번에 보기
```bash
# 여러 인스턴스의 로그를 동시에 grep
grep -nE "Exception|ERROR|SEVERE" \
  /tomcat/tomcat-instance{1,2}/logs/localhost.$(date +%Y-%m-%d).log \
  | sort -t: -k2 | tail -50

# 실시간으로 여러 인스턴스 동시 모니터링
tail -f \
  /tomcat/tomcat-instance1/logs/localhost.$(date +%Y-%m-%d).log \
  /tomcat/tomcat-instance2/logs/localhost.$(date +%Y-%m-%d).log
```

### 스택 트레이스 전체 보기
```bash
# SEVERE 이후 20줄까지 같이 출력
grep -A 20 "SEVERE" /tomcat/logs/localhost.$(date +%Y-%m-%d).log | less
```

---

## 📌 요약 한 줄 정리

- **catalina** → Tomcat이 못 뜨거나 배포 자체가 실패할 때
- **localhost** → 기능 쓰다가 오류 날 때 (500, NPE, SQL 등) ← **실무에서 제일 자주 봄**
- **access_log** → 요청/응답 흔적 추적, 트래픽 분석할 때

---

*마지막 업데이트: 2025-07*
