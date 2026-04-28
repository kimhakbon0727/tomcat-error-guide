# 🔥 Tomcat 오류 대처법 가이드

> 톰캣 운영 중 자주 마주치는 오류 상황과 실전 해결 방법을 정리한 가이드입니다.
> 로그 확인부터 재기동까지, 실무에서 바로 쓸 수 있는 명령어 위주로 작성했습니다.

---

## 📋 목차

- [📂 로그 파일 종류 및 역할](#-로그-파일-종류-및-역할)
- [🚨 서버가 아예 안 뜰 때](#-서버가-아예-안-뜰-때)
- [📦 배포 후 앱이 안 뜰 때](#-배포-후-앱이-안-뜰-때)
- [⚡ 포트 충돌 오류](#-포트-충돌-오류)
- [💾 OutOfMemoryError (메모리 부족)](#-outofmemoryerror-메모리-부족)
- [🌐 404 / 500 에러 대처](#-404--500-에러-대처)
- [🛠️ 자주 쓰는 명령어 모음](#️-자주-쓰는-명령어-모음)
- [📌 환경 정보](#-환경-정보)

---

## 📂 로그 파일 종류 및 역할

톰캣 로그는 `$CATALINA_HOME/logs/` 경로에 있습니다.

| 로그 파일 | 대상 | 언제 보는가 |
|---|---|---|
| `catalina.out` | 톰캣 서버 자체 | 서버 기동 실패, JVM 오류 |
| `localhost.yyyy-mm-dd.log` | 웹 애플리케이션 | 앱 배포 실패, 앱 내부 오류 |
| `localhost_access_log.yyyy-mm-dd.txt` | HTTP 요청/응답 | 404, 500 추적, 트래픽 확인 |

```bash
# 실시간 로그 확인
tail -f /usr/local/tomcat/logs/catalina.out

# 에러만 추려보기
grep -i "SEVERE\|ERROR\|Exception" /usr/local/tomcat/logs/catalina.out

# 오늘 날짜 로그 확인
tail -f /usr/local/tomcat/logs/localhost.$(date +%Y-%m-%d).log
```

> ⚠️ **팁:** 오류 발생 시 `catalina.out` 먼저, 앱 관련 문제면 `localhost.log` 순서로 확인하세요.

---

## 🚨 서버가 아예 안 뜰 때

### 원인
- JVM 버전 불일치
- `server.xml` 설정 오류
- 포트 이미 사용 중
- 권한 문제

### 확인 방법

```bash
# 톰캣 프로세스 확인
ps -ef -ww | grep '[c]atalina'

# 로그에서 SEVERE 찾기
grep "SEVERE" /usr/local/tomcat/logs/catalina.out | tail -20

# Java 버전 확인
java -version
```

### 해결 방법

```bash
# 1. 기존 PID 파일 제거 후 재기동
rm -f /usr/local/tomcat/work/Catalina/
/usr/local/tomcat/bin/shutdown.sh
sleep 3
/usr/local/tomcat/bin/startup.sh

# 2. server.xml 문법 오류 체크 (XML 파서로 확인)
xmllint --noout /usr/local/tomcat/conf/server.xml && echo "OK" || echo "오류 있음"
```

---

## 📦 배포 후 앱이 안 뜰 때

### 원인
- WAR 파일 손상
- 라이브러리 충돌 (jar 버전 중복)
- `web.xml` 설정 오류
- DB 연결 실패로 초기화 오류

### 확인 방법

```bash
# 앱 배포 로그 확인
tail -100 /usr/local/tomcat/logs/localhost.$(date +%Y-%m-%d).log

# work 디렉토리에 캐시 파일 확인
ls -la /usr/local/tomcat/work/Catalina/localhost/

# WAR 파일 정상 여부 확인
jar tf /usr/local/tomcat/webapps/myapp.war | head -20
```

### 해결 방법

```bash
# 1. work 디렉토리 캐시 초기화 (가장 먼저 시도)
/usr/local/tomcat/bin/shutdown.sh
rm -rf /usr/local/tomcat/work/Catalina/*
/usr/local/tomcat/bin/startup.sh

# 2. 기존 배포 파일 제거 후 재배포
rm -rf /usr/local/tomcat/webapps/myapp
rm -f /usr/local/tomcat/webapps/myapp.war
# WAR 파일 다시 복사 후 startup
```

> ⚠️ **주의:** work 디렉토리 삭제 시 반드시 톰캣을 내린 후 진행하세요.

---

## ⚡ 포트 충돌 오류

### 증상

```
java.net.BindException: Address already in use: 8080
```

### 확인 방법

```bash
# 8080 포트 점유 프로세스 확인
ss -tlnp | grep 8080
# 또는
lsof -i :8080

# 톰캣 기본 포트 목록 (모두 확인)
ss -tlnp | grep -E '8080|8443|8005|8009'
```

### 해결 방법

```bash
# 방법 1: 점유 중인 프로세스 종료
kill -9 $(lsof -ti :8080)

# 방법 2: server.xml에서 포트 변경
vi /usr/local/tomcat/conf/server.xml
#  ⚠️ **팁:** `Xmx`는 서버 전체 RAM의 70% 이내로 설정하는 걸 권장합니다.

---

## 🌐 404 / 500 에러 대처

### 404 Not Found

```bash
# 요청 URL 경로 확인
grep "404" /usr/local/tomcat/logs/localhost_access_log.$(date +%Y-%m-%d).txt | tail -20

# webapps 디렉토리 구조 확인
ls -la /usr/local/tomcat/webapps/

# 컨텍스트 루트 확인 (server.xml 또는 context.xml)
grep -i "Context" /usr/local/tomcat/conf/server.xml
```

### 500 Internal Server Error

```bash
# 앱 로그에서 Exception 추적
grep -A 20 "Exception" /usr/local/tomcat/logs/localhost.$(date +%Y-%m-%d).log | tail -50

# catalina.out에서 스택트레이스 확인
grep -A 30 "500\|Exception\|SEVERE" /usr/local/tomcat/logs/catalina.out | tail -60
```

---

## 🛠️ 자주 쓰는 명령어 모음

```bash
# ── 프로세스 확인 ──────────────────────────────
ps -ef -ww | grep '[c]atalina'
pgrep -a java | grep tomcat

# ── 톰캣 경로 확인 ──────────────────────────────
cat /proc/$(pgrep -f catalina)/environ | tr '\0' '\n' | grep -i catalina

# ── 기동 / 종료 / 재기동 ───────────────────────
/usr/local/tomcat/bin/startup.sh
/usr/local/tomcat/bin/shutdown.sh
/usr/local/tomcat/bin/shutdown.sh && sleep 3 && /usr/local/tomcat/bin/startup.sh

# ── 로그 실시간 확인 ────────────────────────────
tail -f /usr/local/tomcat/logs/catalina.out
tail -f /usr/local/tomcat/logs/localhost.$(date +%Y-%m-%d).log

# ── 에러 필터링 ─────────────────────────────────
grep -i "SEVERE\|ERROR\|Exception" /usr/local/tomcat/logs/catalina.out

# ── 포트 확인 ───────────────────────────────────
ss -tlnp | grep -E '8080|8443|8005'
lsof -i :8080

# ── 메모리 확인 ─────────────────────────────────
free -h
jmap -heap $(pgrep -f catalina)
```

---

## 📌 환경 정보

| 항목 | 내용 |
|---|---|
| OS | CentOS 7 / Ubuntu 22.04 |
| Java | OpenJDK 11 |
| Tomcat | 9.x |
| 작성자 | - |
| 최종 수정 | 2026-04-28 |

---

> 💡 **기여 방법:** 새로운 오류 케이스나 해결법은 Issue 또는 PR로 남겨주세요.
