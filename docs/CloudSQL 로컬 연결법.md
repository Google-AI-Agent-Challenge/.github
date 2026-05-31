# Cloud SQL 로컬 접속 가이드

> **프로젝트**: TONES (`knudc-zintn11:us-central1:tones-db`)  
> **DB 엔진**: PostgreSQL  
> **접속 방식**: Cloud SQL Auth Proxy

---

## 사전 준비

### 1. 필수 도구 설치 확인

| 도구 | 설명 | 확인 명령어 |
|---|---|---|
| `cloud-sql-proxy.exe` | 프로젝트 루트에 이미 포함됨 | 파일 존재 확인 |
| `psql` | PostgreSQL 클라이언트 | `psql --version` |
| `gcloud` CLI | Google Cloud 인증 | `gcloud --version` |

### 2. Google Cloud 인증

프록시 실행 전 반드시 아래 중 하나로 인증해야 합니다.

```powershell
# 방법 A: 사용자 계정으로 인증 (개발자 본인)
gcloud auth application-default login

# 방법 B: 서비스 계정 키 파일 사용
$env:GOOGLE_APPLICATION_CREDENTIALS = "C:\path\to\service-account-key.json"
```

---

## 접속 절차

### Step 1: Cloud SQL Auth Proxy 실행

프로젝트 루트 폴더(`Project/`)에서 터미널을 열고 아래 명령어를 실행합니다.

```powershell
# 기본 포트(5432)가 이미 사용 중일 경우 다른 포트(예: 5433)를 지정
.\cloud-sql-proxy.exe knudc-zintn11:us-central1:tones-db --port=5433
```

> **참고**: 프록시가 성공적으로 실행되면 아래와 같은 메시지가 출력됩니다.
> ```
> Listening on 127.0.0.1:5433 for knudc-zintn11:us-central1:tones-db
> Ready for new connections
> ```
>
> ⚠️ 프록시는 **별도의 터미널 창을 유지한 채** 실행되어야 합니다.  
> 접속이 끝날 때까지 닫지 마세요.

---

### Step 2: psql로 DB 접속

**새 터미널 창**을 열고 아래 명령어로 접속합니다.

```powershell
psql -h 127.0.0.1 -p 5433 -U postgres -d postgres
```

| 옵션 | 값 | 설명 |
|---|---|---|
| `-h` | `127.0.0.1` | 프록시가 바인딩된 로컬 주소 |
| `-p` | `5433` | 프록시 포트 (`.env`의 `DB_PORT` 기준 조정) |
| `-U` | `postgres` | DB 사용자명 (`.env`의 `DB_USER`) |
| `-d` | `postgres` | 접속할 DB 이름 (`.env`의 `DB_NAME`) |

비밀번호 입력 프롬프트가 나타나면 `.env` 파일의 `DB_PASS` 값을 입력합니다.

```
Password for user postgres: ********
psql (16.x)
Type "help" for help.

postgres=#
```

---

### Step 3: 자주 쓰는 psql 명령어

```sql
-- 현재 DB 확인
SELECT current_database();

-- 테이블 목록 보기
\dt

-- 특정 테이블 구조 보기
\d 테이블명

-- DB 목록 보기
\l

-- 접속 종료
\q
```

---

## 환경 변수 요약 (`.env` 기준)

```dotenv
# GCP Config & Cloud SQL Config
GCP_PROJECT_ID="knudc-zintn11"
GCP_REGION="us-central1"
CLOUD_SQL_CONNECTION_NAME="knudc-zintn11:us-central1:tones-db"
DB_USER="postgres"
DB_PASS="<실제 비밀번호>"
DB_NAME="postgres"
DB_HOST="127.0.0.1"    # 로컬 개발 시 프록시 주소
DB_PORT=5433           # 프록시 포트
```

---

## Python에서 직접 접속하는 경우 (psycopg2)

```python
import psycopg2

conn = psycopg2.connect(
    host="127.0.0.1",
    port=5433,
    user="postgres",
    password="<DB_PASS>",
    database="postgres"
)

cursor = conn.cursor()
cursor.execute("SELECT version();")
print(cursor.fetchone())

cursor.close()
conn.close()
```

---

## 트러블슈팅

### ❌ 인증 오류: `Could not find Application Default Credentials`

```powershell
gcloud auth application-default login
```

### ❌ 포트 충돌: `bind: address already in use`

로컬에 이미 실행 중인 PostgreSQL이 5432 포트를 점유하고 있는 경우,  
`--port` 옵션으로 다른 포트를 지정합니다.

```powershell
.\cloud-sql-proxy.exe knudc-zintn11:us-central1:tones-db --port=5433
```

### ❌ psql 명령어를 찾을 수 없는 경우

PostgreSQL 클라이언트를 설치하거나, 설치 경로를 PATH에 추가합니다.

```powershell
# PostgreSQL 설치 경로 예시
$env:PATH += ";C:\Program Files\PostgreSQL\16\bin"
```

### ❌ 권한 없음 오류: `IAM permission denied`

서비스 계정 또는 사용자 계정에 아래 IAM 역할이 필요합니다.
- `Cloud SQL Client` (`roles/cloudsql.client`)

```powershell
gcloud projects add-iam-policy-binding knudc-zintn11 \
  --member="user:your-email@gmail.com" \
  --role="roles/cloudsql.client"
```

---

## 참고 링크

- [Cloud SQL Auth Proxy 공식 문서](https://cloud.google.com/sql/docs/postgres/connect-auth-proxy)
- [Cloud SQL IAM 인증](https://cloud.google.com/sql/docs/postgres/iam-authentication)
- [psql 명령어 레퍼런스](https://www.postgresql.org/docs/current/app-psql.html)
