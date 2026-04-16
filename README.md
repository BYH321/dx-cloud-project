# E-Commerce REST API

Flask 기반 E-Commerce REST API (Product / Category)

---

## 기술 스택

| 라이브러리 | 버전 | 역할 |
|---|---|---|
| Flask | ≥ 3.0 | 웹 프레임워크 |
| psycopg2-binary | ≥ 2.9 | PostgreSQL 어댑터 |
| paramiko | ≥ 3.4 | SFTP 이미지 업로드/삭제 |
| python-dotenv | ≥ 1.0 | 환경 변수 관리 |

---

## Flask 외 기술 스택 상세

### psycopg2-binary

PostgreSQL 공식 Python 어댑터입니다. `psycopg2-binary`는 C 확장 라이브러리가 미리 컴파일된 바이너리 패키지로, 별도의 PostgreSQL 클라이언트 라이브러리 설치 없이 바로 사용할 수 있습니다.

**사용 위치:** `database.py`

**주요 특징:**
- `ThreadedConnectionPool`로 커넥션 풀을 관리해 요청마다 새 연결을 맺지 않습니다.
- `RealDictCursor`를 사용해 쿼리 결과를 `dict` 형태로 반환합니다.
- `psycopg2.extras`의 커서를 사용하면 컬럼명으로 바로 값에 접근할 수 있습니다.

**커넥션 풀 환경 변수:**

| 변수 | 기본값 | 설명 |
|---|---|---|
| `DB_HOST` | `localhost` | PostgreSQL 호스트 |
| `DB_PORT` | `5432` | PostgreSQL 포트 |
| `DB_NAME` | `ecommerce` | 데이터베이스 이름 |
| `DB_USER` | `postgres` | 접속 유저 |
| `DB_PASSWORD` | (없음) | 패스워드 |
| `DB_POOL_MIN` | `1` | 풀 최소 커넥션 수 |
| `DB_POOL_MAX` | `10` | 풀 최대 커넥션 수 |

---

### paramiko

SSH2 프로토콜 구현 라이브러리로, SFTP 클라이언트 기능을 포함합니다. 이미지 파일을 원격 SFTP 서버에 업로드하고 삭제할 때 사용합니다.

**사용 위치:** `services/sftp_service.py`

**주요 특징:**
- `SSHClient.connect()`로 SSH 터널을 열고, `open_sftp()`로 SFTP 세션을 획득합니다.
- `sftp.putfo(BytesIO, remote_path)`를 사용해 파일을 메모리에서 바로 업로드합니다 (임시 파일 불필요).
- 원격 디렉터리가 없으면 자동으로 생성합니다.
- 업로드 완료 후 SSH/SFTP 연결을 즉시 종료합니다.

**SFTP 환경 변수:**

| 변수 | 기본값 | 설명 |
|---|---|---|
| `SFTP_HOST` | `localhost` | SFTP 서버 호스트 |
| `SFTP_PORT` | `22` | SFTP 포트 |
| `SFTP_USER` | (없음) | SFTP 접속 유저 |
| `SFTP_PASSWORD` | (없음) | SFTP 패스워드 |
| `SFTP_BASE_DIR` | `/uploads/products` | 이미지 저장 원격 디렉터리 |
| `SFTP_BASE_URL` | (없음) | 이미지 공개 URL 접두사 (CDN 주소 등) |

> `SFTP_BASE_URL`을 설정하면 업로드된 이미지의 `image_url`이 `https://cdn.example.com/products/abc123.jpg` 형태로 저장됩니다. 비워두면 원격 경로(`/uploads/products/abc123.jpg`)가 저장됩니다.

**허용 이미지 형식:** `jpg`, `jpeg`, `png`, `webp`, `gif`

---

### python-dotenv

`.env` 파일에서 환경 변수를 읽어 `os.environ`에 주입하는 라이브러리입니다. 코드에 민감한 정보(DB 패스워드, SFTP 자격증명)를 하드코딩하지 않고 별도 파일로 관리할 수 있습니다.

**사용 위치:** `config/db.py`, `config/sftp.py`

**설정 방법:**
```bash
cp .env.example .env
# .env 파일을 열어 실제 값으로 수정
```

> `.env` 파일은 절대 git에 커밋하지 마세요. `.gitignore`에 추가하세요.

---

## 프로젝트 구조

```
.
├── app.py                  # Flask 앱 팩토리 & 진입점
├── database.py             # psycopg2 커넥션 풀 관리
├── config/
│   ├── db.py               # PostgreSQL 환경 변수
│   └── sftp.py             # SFTP 환경 변수
├── routes/
│   ├── category.py         # Category API Blueprint
│   └── product.py          # Product API Blueprint
├── services/
│   └── sftp_service.py     # SFTP 업로드/삭제 서비스
├── schema.sql              # PostgreSQL 스키마 정의
├── .env.example            # 환경 변수 예시
├── requirements.txt
└── README.md
```

---

## 설치 및 실행

```bash
# 의존성 설치
pip install -r requirements.txt

# 환경 변수 설정
cp .env.example .env

# DB 스키마 적용
psql -U postgres -d ecommerce -f schema.sql

# 서버 실행
python app.py
```

---

## API 엔드포인트

### Category API

| Method | URL | 설명 |
|---|---|---|
| GET | `/api/categories` | 카테고리 목록 (`?tree=true`로 트리 반환, `?parent_id=<id>`로 필터) |
| GET | `/api/categories/<id>` | 카테고리 단건 조회 |
| GET | `/api/categories/<id>/children` | 하위 카테고리 목록 |
| POST | `/api/categories` | 카테고리 생성 |
| PUT | `/api/categories/<id>` | 카테고리 수정 |
| DELETE | `/api/categories/<id>` | 카테고리 삭제 |

**POST/PUT 요청 바디 (JSON):**
```json
{
  "name": "아우터",
  "slug": "outer",
  "parent_id": null,
  "description": "자켓, 코트 등",
  "sort_order": 1,
  "is_active": true
}
```

---

### Product API

| Method | URL | 설명 |
|---|---|---|
| GET | `/api/products` | 상품 목록 (`?category_id=`, `?search=`, `?page=`, `?limit=`) |
| GET | `/api/products/<id>` | 상품 단건 조회 (이미지, 옵션, SKU 포함) |
| POST | `/api/products` | 상품 생성 |
| PUT | `/api/products/<id>` | 상품 수정 |
| DELETE | `/api/products/<id>` | 상품 소프트 삭제 |
| POST | `/api/products/<id>/images` | 이미지 업로드 (multipart/form-data) |
| DELETE | `/api/products/<id>/images/<image_id>` | 이미지 삭제 |

**POST /api/products 요청 바디 (JSON):**
```json
{
  "category_id": "uuid",
  "name": "오버핏 코튼 자켓",
  "slug": "overfit-cotton-jacket",
  "description": "상품 설명",
  "base_price": 89000,
  "discount_price": 75000,
  "is_active": true
}
```

**POST /api/products/<id>/images 요청 (multipart/form-data):**
```
image        = <파일>
alt_text     = "상품 대표 이미지"
sort_order   = 0
is_thumbnail = true
```
