## 다중 컨테이너 웹 애플리케이션 구축 실습 (Flask & MySQL)

본 실습은 **Flask 웹 서버**와 **MySQL 데이터베이스**를 연동하는 전형적인 2-Tier 아키텍처를 도커 컴포즈로 구축합니다. 특히 외부 SQL 파일을 활용한 데이터 초기화와 환경 변수를 이용한 보안 설정을 포함한 중급 수준의 프로젝트입니다.

---

### 1. 프로젝트 설명 및 트리 구조

이 프로젝트는 사용자 정의 Flask 이미지와 공식 MySQL 이미지를 조합합니다. `employees.sql` 데이터를 DB 컨테이너 생성 시 자동으로 주입하며, Flask 서버는 이를 쿼리하여 웹 화면에 부서 정보를 출력합니다.

**프로젝트 트리 구조**

```text
my-db-project/
├── app.py              # Flask 웹 애플리케이션 (DB 연동 로직)
├── requirements.txt    # Python 패키지 의존성 (Flask, PyMySQL)
├── Dockerfile          # Flask 애플리케이션 빌드 정의서
├── compose.yaml        # 전체 서비스 오케스트레이션 설정
└── db/
    └── employees.sql   # 초기 데이터 주입용 SQL 파일

```

---

### 2. 파일별 상세 내용 및 주석

#### ① Flask 애플리케이션 (`app.py`)

```python
from flask import Flask, render_template_string
import pymysql
import os

app = Flask(__name__)

# 환경 변수로부터 DB 접속 정보를 가져옵니다.
DB_HOST = os.environ.get('DB_HOST', 'db')
DB_USER = os.environ.get('DB_USER', 'admin')
DB_PASS = os.environ.get('DB_PASSWORD', 'admin123')
DB_NAME = os.environ.get('DB_NAME', 'employees')

def get_db_connection():
    return pymysql.connect(
        host=DB_HOST,
        user=DB_USER,
        password=DB_PASS,
        db=DB_NAME,
        charset='utf8mb4',
        cursorclass=pymysql.cursors.DictCursor
    )

@app.route('/')
def index():
    try:
        conn = get_db_connection()
        with conn.cursor() as cursor:
            # 부서 정보를 가져오는 SQL 쿼리 실행
            sql = "SELECT dept_no, dept_name FROM departments"
            cursor.execute(sql)
            results = cursor.fetchall()
        conn.close()
        
        # 결과를 HTML 테이블로 렌더링
        html = """
        <h1>부서 목록 (DB Query Result)</h1>
        <table border="1">
            <tr><th>부서번호</th><th>부서명</th></tr>
            {% for row in results %}
            <tr><td>{{ row.dept_no }}</td><td>{{ row.dept_name }}</td></tr>
            {% endfor %}
        </table>
        """
        return render_template_string(html, results=results)
    except Exception as e:
        return f"DB 연결 실패: {str(e)}"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

```

#### ② 의존성 파일 (`requirements.txt`)

```text
flask
pymysql
cryptography

```

#### ③ 웹 서버 빌드 정의서 (`Dockerfile`)

```dockerfile
FROM python:3.9-slim

# 작업 디렉토리 설정
WORKDIR /app

# 필요한 패키지 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 소스 코드 복사
COPY . .

# 5000번 포트 노출
EXPOSE 5000

# Flask 앱 실행
CMD ["python", "app.py"]

```

#### ④ 서비스 구성 (`compose.yaml`)

```yaml
services:
  web:
    build: .
    ports:
      - "8080:5000"
    environment:
      - DB_HOST=db
      - DB_NAME=employees
      - DB_USER=admin
      - DB_PASSWORD=admin123
    depends_on:
      db:
        condition: service_healthy # DB가 완전히 준비된 후 웹 서버 실행

  db:
    image: mysql:8.0
    restart: always
    environment:
      - MYSQL_DATABASE=employees
      - MYSQL_ROOT_PASSWORD=root_pass
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=admin123
    ports:
      - "3306:3306"
    volumes:
      # 초기 SQL 파일을 컨테이너 내 예약된 경로에 마운트 (자동 주입)
      - ./db/employees.sql:/docker-entrypoint-initdb.d/employees.sql
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:

```

---

### 3. 상세 설명 및 핵심 동작 원리

1. **DB 자동 데이터 주입**: MySQL 공식 이미지는 `/docker-entrypoint-initdb.d/` 디렉토리에 있는 `.sql` 파일을 컨테이너 최초 생성 시 자동으로 실행합니다.
2. **어드민 계정 생성**: `MYSQL_USER`와 `MYSQL_PASSWORD` 환경 변수를 통해 루트 외에 일반 관리자 계정을 자동으로 생성합니다.
3. **서비스 의존성 (`healthcheck`)**: 단순 `depends_on`은 컨테이너가 '시작'될 때만 체크합니다. `healthcheck`를 결합하면 DB가 쿼리를 받을 수 있는 '완전 준비' 상태가 되었을 때 Flask가 시작되도록 보장합니다.

---

### 4. 상태 확인 (Status Check)

실행 중인 서비스와 리소스의 상태를 점검합니다. (`docker compose up -d` 실행 후)

**① 서비스 상태 확인 (Compose)**

```bash
docker compose ps

```

```text
NAME                IMAGE               STATUS              PORTS
my-db-project-web-1  my-db-project-web   running             0.0.0.0:8080->5000/tcp
my-db-project-db-1   mysql:8.0           running (healthy)   0.0.0.0:3306->3306/tcp

```

**② 이미지 상태 확인 (Image)**

```bash
docker images

```

```text
REPOSITORY           TAG       IMAGE ID       CREATED          SIZE
my-db-project-web    latest    a1b2c3d4e5f6   1 minute ago     150MB
mysql                8.0       f7e6d5c4b3a2   2 days ago       600MB

```

**③ 전체 시스템 상태 요약 (System)**

```bash
docker system df

```

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          2         2         750MB     0B
Containers      2         2         12.5kB    0B
Local Volumes   1         1         150MB     0B

```

---

### 5. 자원 삭제 및 최종 확인

모든 리소스를 제거하고 깨끗한 상태로 되돌립니다.

```bash
# 1. 모든 서비스 중지 및 삭제 (볼륨, 이미지 포함)
docker compose down -v --rmi all

# 2. 삭제 확인 (목록이 비어있어야 함)
docker compose ps
docker images
docker volume ls

```

---

### AWS 비용 및 유료 전환 주의사항

* **Amazon RDS**: 만약 도커 기반 MySQL 대신 RDS를 사용할 경우, **DB 인스턴스 사양**과 **저장 공간(GP3 등)**에 따라 비용이 실시간으로 청구됩니다. 프리티어 종료 후에는 매우 높은 비용이 발생할 수 있습니다.
* **Data Transfer**: 웹 서버(EC2)와 DB(RDS) 간의 통신이 서로 다른 가용 영역(AZ)에서 발생할 경우 **데이터 전송 비용**이 발생합니다.
* **Public IP**: DB 컨테이너 포트를 외부(3306)로 노출한 채 운영하면 보안에 취약할 뿐만 아니라, EC2 퍼블릭 IP와 관련된 소액의 유지비용이 발생할 수 있습니다.

## Next Step: Docker Compose를 활용한 Nginx 리버스 프록시 설정

*요청하신 대로 초기 데이터 주입과 어드민 계정 생성을 포함한 전체 프로젝트 가이드를 작성하였습니다.*

Next Step: 로그 관리 및 분석 방법
