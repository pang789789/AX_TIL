# 2026-11-02 TIL — FastAPI 회원 등록·부분 수정 API와 PostgreSQL Transaction

## 오늘 학습한 내용

오늘은 FastAPI에서 받은 회원 정보를 PostgreSQL의 `members` 테이블에 실제로 저장하는 과정을 실습했다. 이전 수업에서는 Python과 PostgreSQL의 연결 및 조회를 중심으로 진행했다면, 오늘은 `POST /members` 요청을 받아 유효성을 검사하고, 중복 이메일을 확인한 뒤 `INSERT`와 `commit`까지 수행하는 전체 흐름을 구현했다.

오늘 구현한 처리 흐름은 다음과 같다.

```text
POST /members 요청
→ Pydantic 입력값 검증
→ 이메일 중복 검사
→ parameterized INSERT 실행
→ RETURNING으로 생성된 데이터 받기
→ commit
→ 201 Created 응답
```

오류가 생겼을 때는 오류의 성격에 따라 다음과 같이 구분했다.

- 요청값의 형식이 잘못된 경우: `422 Unprocessable Entity`
- 이미 등록된 이메일인 경우: `409 Conflict`
- DB 처리 중 예상하지 못한 오류가 발생한 경우: `rollback` 후 `500 Internal Server Error`

## 직접 구현한 내용

### 1. 입력 모델과 응답 모델 분리

`MemberCreate`는 사용자가 보내야 하는 `name`, `email`, `age`를 정의하고, `MemberResponse`는 여기에 DB가 생성하는 `id`를 추가하도록 구성했다.

```python
class MemberCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=0, le=150)


class MemberResponse(MemberCreate):
    id: int
```

처음에는 입력과 응답 모델을 왜 나누는지 크게 와닿지 않았지만, 클라이언트가 회원을 등록할 때는 `id`를 보내지 않고 DB에서 생성된 `id`를 응답으로 받는다는 점 때문에 역할을 분리한다는 것을 이해했다.

### 2. DB 연결 코드를 공통 모듈로 분리

매번 endpoint 안에서 연결 문자열과 SQL을 모두 작성하면 코드가 복잡해지기 때문에, `phase02/db.py`에 DB 연결 및 SQL 실행 함수를 모았다. `.env`도 현재 파일 위치를 기준으로 불러오도록 하여 실행 위치가 달라져도 설정을 찾을 수 있게 했다.

```python
PHASE02_DIR = Path(__file__).resolve().parent
load_dotenv(PHASE02_DIR / ".env")
```

이 방법을 사용하면 chapter가 바뀌더라도 공통 DB 연결 함수를 재사용할 수 있다. 특히 `.env`의 DB 비밀번호를 코드에 직접 작성하지 않는 것이 중요하다는 점도 다시 확인했다.

### 3. 중복 이메일 검사

회원 정보를 저장하기 전에 같은 이메일이 이미 존재하는지 조회했다.

```python
cursor.execute(
    "SELECT id FROM members WHERE email = %s",
    (email,),
)
```

조회 결과가 존재하면 `HTTPException`을 발생시켜 `409 Conflict`를 반환했다. 이메일 형식이 잘못된 것은 입력값 검증 문제이므로 `422`, 형식은 올바르지만 이미 DB에 존재하는 것은 현재 데이터 상태와의 충돌이므로 `409`라는 차이를 배웠다.

### 4. Parameterized INSERT와 RETURNING

사용자 입력을 f-string으로 SQL에 직접 붙이지 않고 `%s` placeholder와 값 tuple을 분리했다.

```python
cursor.execute(
    """
    INSERT INTO members (name, email, age)
    VALUES (%s, %s, %s)
    RETURNING id, name, email, age
    """,
    (name, email, age),
)
```

이 방식은 드라이버가 값의 escaping을 처리하므로 SQL Injection 위험을 줄여 준다. 또한 `RETURNING`을 사용하면 INSERT한 뒤 별도의 SELECT를 다시 실행하지 않아도 방금 생성된 회원 정보를 받을 수 있었다.

### 5. Transaction 처리

INSERT가 성공한 뒤 `conn.commit()`을 실행하고, 오류가 발생하면 `conn.rollback()`을 실행하도록 작성했다. 마지막에는 성공 여부와 관계없이 연결이 닫히도록 `finally`에서 `conn.close()`를 호출했다.

```python
try:
    # 중복 검사 및 INSERT
    conn.commit()
except HTTPException:
    conn.rollback()
    raise
except Exception:
    conn.rollback()
    raise HTTPException(status_code=500, detail="회원 등록 중 오류가 발생했습니다.")
finally:
    conn.close()
```

이번 실습을 통해 `INSERT를 실행했다`와 `DB에 저장이 확정되었다`는 같은 의미가 아니라는 것을 알게 되었다. 변경을 실제로 확정하려면 반드시 `commit`이 필요하고, 작업 중 문제가 생기면 `rollback`으로 해당 Transaction의 변경을 취소해야 한다.

## 실습하면서 어려웠던 부분과 해결 과정

### 1. 교수님 화면과 내 터미널 결과가 달랐던 문제

교수님 화면에서는 `(.venv)`가 표시된 상태에서 Python 명령이 정상 실행되었지만, 내 터미널에서는 `python .env`를 입력하거나 `python -c` 뒤에 실행할 코드가 빠져 오류가 발생했다. 처음에는 Python이나 DB 연결 자체의 문제라고 생각했지만, 실제 원인은 가상환경 활성화 상태와 명령어 형식이었다.

다음과 같이 프로젝트 루트로 이동하고 가상환경을 활성화한 후, 완전한 명령을 실행하여 해결했다.

```powershell
cd C:\dev\fullstack-web-course
.\practice\phase02\.venv\Scripts\Activate.ps1
python -c "from practice.phase02.db import check_connection; print(check_connection())"
```

이 경험을 통해 터미널 오류가 발생하면 코드부터 수정하기 전에 현재 경로, 활성화된 가상환경, 입력한 명령의 인자가 완전한지 먼저 확인해야 한다는 것을 배웠다.

### 2. DBeaver에서 테이블이 보이지 않거나 권한 오류가 발생한 문제

처음 DBeaver에서는 다른 원격 연결과 로컬 연결이 함께 표시되어 어떤 DB에서 작업하는지 혼동했다. 또한 `django_session`을 조회했을 때 `permission denied`가 발생했는데, 이 테이블은 이번 실습에서 만든 `members` 테이블이 아니었고 현재 사용자가 접근할 대상도 아니었다.

로컬 PostgreSQL의 `postgres` 연결을 선택한 뒤 다음 순서로 펼쳐 실제 실습 테이블을 확인했다.

```text
postgres
→ Databases
→ postgres
→ Schemas
→ public
→ Tables
→ members
```

결국 연결 이름만 보고 판단하지 않고 host, port, database, schema와 현재 선택된 connection을 함께 확인해야 한다는 점을 알게 되었다.

### 3. `members` 테이블이 처음에는 존재하지 않았던 문제

코드에서 `members`를 조회하려 했지만 테이블 자체가 아직 없어서 먼저 테이블을 생성하고 샘플 데이터를 넣어야 했다. DBeaver에서 다음과 같은 구조로 테이블을 준비했다.

```sql
CREATE TABLE members (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INTEGER NOT NULL
);
```

그 뒤 샘플 데이터를 넣고 SELECT로 확인하면서 Python 코드가 사용할 DB 상태를 맞췄다. API 코드가 맞더라도 대상 테이블과 컬럼이 준비되어 있지 않으면 동작하지 않으므로, 애플리케이션 코드와 DB schema를 항상 함께 확인해야 한다.

### 4. `EmailStr` 사용 시 패키지가 없었던 문제

Pydantic의 `EmailStr`를 사용하자 `email-validator`가 설치되어 있지 않다는 오류가 발생했다. DB 연결이나 SQL 오류가 아니라 이메일 검증에 필요한 선택 패키지가 빠진 것이 원인이었다.

활성화된 가상환경에서 다음 명령으로 설치하여 해결했다.

```powershell
python -m pip install email-validator
```

패키지는 시스템 전체 Python이 아니라 현재 프로젝트가 사용하는 `.venv`에 설치해야 한다는 것도 중요했다.

### 5. 브라우저에서 `127.0.0.1:8000`이 열리지 않았던 문제

코드가 완성된 뒤 브라우저에서 주소를 열었지만 `ERR_CONNECTION_REFUSED`가 나타났다. 처음에는 링크가 잘못된 줄 알았지만, 실제로는 Uvicorn 서버가 실행 중이지 않은 상태였다.

프로젝트 루트에서 다음 명령으로 서버를 실행했다.

```powershell
python -m uvicorn practice.phase02.chapter07.main:app --reload --host 127.0.0.1 --port 8000
```

그 후 기본 주소가 아니라 Swagger 문서 주소인 아래 링크로 접속했다.

```text
http://127.0.0.1:8000/docs
```

서버 실행 터미널은 계속 켜 두어야 하며, 종료하려면 `Ctrl+C`를 누르면 된다. `127.0.0.1`은 외부 웹사이트가 아니라 내 컴퓨터에서 실행 중인 서버이기 때문에 서버 프로세스가 없으면 접속할 수 없다는 것을 확실히 이해했다.

## 테스트 결과

정상 요청, 중복 요청, 잘못된 입력을 각각 테스트하고 API 결과뿐 아니라 실제 DB row 수도 확인했다.

### 정상 회원 등록

- `POST /members` 실행 결과: `201 Created`
- 응답에 DB가 생성한 `id` 포함
- 테스트 응답의 생성된 회원 ID: `4`
- DBeaver에서 같은 회원 row가 실제로 저장된 것을 확인

### 중복 이메일 요청

- 동일한 이메일로 다시 요청한 결과: `409 Conflict`
- 응답 메시지: `이미 등록된 이메일입니다.`
- 요청 전후 DB row 수: `4 → 4`

상태 코드만 확인하지 않고 중복 요청 때문에 데이터가 추가되지 않았는지까지 확인했다.

### 잘못된 이메일 요청

- `not-email`을 입력한 결과: `422 Unprocessable Entity`
- 요청 전후 DB row 수: `4 → 4`

이 경우에는 Pydantic 검증 단계에서 요청이 차단되므로 endpoint 내부의 INSERT 코드까지 도달하지 않는다는 것을 확인했다.

## 느낀 점

처음에는 회원 등록 API를 만드는 일이 단순히 INSERT 문 하나를 실행하는 작업이라고 생각했다. 하지만 실제로는 요청 데이터의 계약, 입력값 검증, 현재 DB 상태에 따른 비즈니스 규칙, 안전한 SQL 작성, Transaction 처리, HTTP 상태 코드, 연결 종료까지 모두 연결되어 있었다.

특히 Swagger에서 `201`이 출력되었다고 끝난 것이 아니라 DBeaver에서 실제 row를 조회하고, 오류 요청 이후 row 수가 증가하지 않았는지 확인하는 과정이 인상적이었다. 화면에 성공 메시지가 보이는 것과 데이터가 올바르게 저장된 것은 별개의 검증 대상이라는 점을 배웠다.

또한 오류가 발생했을 때 무작정 코드를 바꾸기보다 오류가 발생한 층을 먼저 구분하는 것이 중요했다. 터미널 명령 문제인지, Python 패키지 문제인지, DB 연결 또는 권한 문제인지, Pydantic 검증 문제인지, SQL 실행 문제인지 차례대로 좁혀 가니 해결이 훨씬 쉬웠다.

이번 실습에서 가장 크게 이해한 것은 다음 세 가지다.

1. 사용자 입력은 SQL 문자열에 직접 합치지 않고 parameterized query로 전달한다.
2. INSERT 이후 성공하면 `commit`, 실패하면 `rollback`을 실행한다.
3. API 응답뿐 아니라 실제 DB 데이터까지 확인해야 실습이 완료된 것이다.

## 다음 학습에서 보완할 점

- SQL 컬럼명을 일부러 잘못 지정하여 `500`과 `rollback`이 실제로 동작하는지 직접 검증해 보기
- 회원 생성 후 `GET /members/{member_id}`로 다시 조회하여 POST 응답과 비교하기
- `check_duplicate_email`처럼 남아 있는 중복 helper를 정리하고 DB 접근 함수의 역할을 더 명확히 나누기
- 다음 Chapter의 `PUT/PATCH + UPDATE`에서 전체 수정과 부분 수정의 차이 학습하기

오늘은 여러 번 막히기는 했지만, 각각의 오류 원인을 직접 확인하면서 FastAPI와 PostgreSQL이 연결되는 전체 흐름을 이전보다 구체적으로 이해할 수 있었다.

---

## 오후 수업: PATCH 부분 수정과 UPDATE

오후에는 오전에 만든 회원 데이터를 수정하는 API를 실습했다. 처음에는 수정 API도 POST와 비슷하게 요청값을 받아 UPDATE만 실행하면 된다고 생각했다. 하지만 PATCH는 요청에 포함된 필드만 바꾸고 나머지 값은 그대로 보존해야 하므로, 전체 수정과는 다른 처리가 필요했다.

이번 실습의 흐름은 다음과 같았다.

```text
PATCH /members/{member_id} 요청
→ 요청에 실제로 포함된 필드 추출
→ 빈 요청인지 검사
→ 수정할 회원 존재 여부 확인
→ 이메일이 포함된 경우 중복 검사
→ 허용된 컬럼만 UPDATE 문으로 구성
→ parameterized UPDATE 실행
→ RETURNING으로 수정 결과 받기
→ commit
→ GET으로 다시 조회
→ Swagger 응답과 PostgreSQL row 비교
```

### 1. PUT과 PATCH의 차이

PUT은 회원의 이름, 이메일, 나이처럼 정해진 전체 필드를 모두 받아 교체하는 방식이고, PATCH는 변경할 필드만 받는 방식으로 이해했다. 예를 들어 나이만 변경하고 싶다면 PATCH 요청에는 다음 값만 보내면 된다.

```json
{
  "age": 31
}
```

이때 `name`과 `email`은 기존 값이 그대로 남아 있어야 한다. 처음에는 Pydantic 모델의 기본값이 `None`이면 누락된 필드도 모두 `None`으로 바뀌는 것이 아닌지 헷갈렸다. `model_dump(exclude_unset=True)`를 사용하면 사용자가 실제로 보낸 필드만 dictionary에 들어간다는 것을 확인했다.

```python
changes = member.model_dump(exclude_unset=True)
```

### 2. 필드 생략과 명시적인 null 구분

PATCH에서는 필드를 보내지 않은 것과 `null`을 직접 보낸 것이 다르다.

```text
{}                    → 변경할 값이 없으므로 400
{"age": 31}           → age만 수정
{"email": null}       → 명시적인 null이므로 422
```

처음에는 `str | None`으로 선언했기 때문에 `null`도 허용되는 줄 알았다. 그러나 이번 API 계약에서는 필드 생략만 허용하고, 직접 보낸 `null`은 거부해야 했다. `model_fields_set`을 확인하는 validator를 추가하여 해결했다.

```python
@model_validator(mode="after")
def reject_explicit_null(self):
    for field in self.model_fields_set:
        if getattr(self, field) is None:
            raise ValueError(f"{field} cannot be null")
    return self
```

이 과정을 통해 Optional이라는 표현이 무조건 null을 저장해도 된다는 뜻은 아니고, 실제 서비스의 API 계약에 따라 별도 검증이 필요하다는 점을 배웠다.

### 3. 동적 UPDATE를 안전하게 만드는 방법

PATCH는 어떤 필드가 들어올지 정해져 있지 않아서 `SET` 절을 동적으로 만들어야 했다. 처음에는 사용자 요청의 key를 그대로 SQL에 넣어도 되는지 걱정되었다. 해결 방법은 코드에서 수정 가능한 컬럼을 미리 제한하고, 실제 값은 계속 `%s` parameter로 전달하는 것이었다.

```python
allowed_fields = {
    "name": "name",
    "email": "email",
    "age": "age",
}

set_parts = []
values = []

for field, value in changes.items():
    column = allowed_fields[field]
    set_parts.append(f"{column} = %s")
    values.append(str(value) if field == "email" else value)
```

컬럼명은 `allowed_fields` 안에서만 선택하고, 사용자가 보낸 값은 placeholder로 분리했다. 동적 SQL이라고 해서 모든 문자열을 f-string으로 붙이는 것이 아니라, 구조와 값을 구분해야 안전하다는 것을 알게 되었다.

### 4. 오류 상태 코드 구분

PATCH 테스트를 하면서 오류마다 의미가 다르다는 것을 다시 확인했다.

- `400 Bad Request`: 요청 형식은 맞지만 수정할 필드가 없음
- `404 Not Found`: 수정하거나 조회할 회원이 없음
- `409 Conflict`: 다른 회원이 이미 사용하는 이메일과 충돌
- `422 Unprocessable Entity`: 타입, 범위, 이메일 형식 또는 explicit null 규칙 위반

처음에는 실패하면 모두 400으로 처리해도 되는 줄 알았지만, 어느 단계에서 왜 실패했는지를 상태 코드로 구분하면 API를 사용하는 쪽에서도 문제를 훨씬 정확하게 파악할 수 있었다.

### 5. 수정 후 GET으로 다시 조회

교수님 화면에서는 Swagger에 다음 두 API가 표시되었다.

```text
PATCH /members/{member_id}
GET   /members/{member_id}
```

처음 작성한 코드에는 PUT/PATCH만 있어서 교수님 화면과 달랐다. 수업 진도를 다시 확인한 뒤 PUT을 제외하고, 수정된 회원을 다시 조회하는 GET endpoint를 추가했다.

```python
@app.get("/members/{member_id}", response_model=MemberResponse)
def get_member(member_id: int):
    with get_connection() as conn:
        with conn.cursor(row_factory=dict_row) as cur:
            cur.execute(
                """
                SELECT id, name, email, age
                FROM members
                WHERE id = %s
                """,
                (member_id,),
            )
            member = cur.fetchone()

    if member is None:
        raise HTTPException(status_code=404, detail="Member not found")

    return member
```

교안 예제에는 기본키 이름이 `member_id`로 적힌 부분이 있었지만, 내 PostgreSQL 테이블의 실제 컬럼은 `id`였다. 예제 코드를 그대로 복사했을 때 컬럼이 존재하지 않는 오류가 날 수 있으므로 DBeaver에서 실제 schema를 확인하고 모든 SQL을 `id` 기준으로 맞췄다.

## 오후 실습에서 잘 안됐던 부분과 해결 방법

### 1. 교수님은 id=8이 조회되는데 내 DB에서는 404가 나온 문제

교수님은 다음 SQL로 8번 회원을 조회했지만 내 DB에는 1~4번 회원만 존재했다.

```sql
SELECT id, name, email, age
FROM members
WHERE id = 8;
```

처음에는 GET 코드가 잘못된 줄 알았지만, DB를 직접 조회해 보니 단순히 데이터가 없는 것이 원인이었다. 코드 오류와 데이터 상태 문제를 구분하기 위해 API보다 먼저 SELECT 결과를 확인했다.

실습용으로 다음 회원을 등록했다.

```text
id: 8
name: 홍길동
email: hong8@example.com
age: 27
```

그 후 `GET /members/8`을 실행하여 다음 세 결과가 일치하는지 확인했다.

```text
Swagger PATCH Response
= PostgreSQL SELECT 결과
= GET /members/8 Response
```

### 2. 기존 이메일과 충돌한 문제

교수님 화면의 `hong@example.com`을 그대로 사용하려 했지만 내 DB에는 이미 같은 이메일을 사용하는 3번 회원이 있었다. 이메일 컬럼에는 UNIQUE 규칙이 있으므로 같은 값을 다시 INSERT하면 충돌한다.

기존 회원을 삭제하거나 덮어쓰지 않고, 실습용 이메일을 `hong8@example.com`으로 변경했다. 이 경험을 통해 같은 코드를 실행해도 각자의 DB 데이터 상태가 다르면 결과가 달라질 수 있으며, INSERT나 UPDATE 전에 중복 데이터를 확인해야 한다는 것을 배웠다.

### 3. IDENTITY 컬럼에 id를 직접 넣을 수 없었던 문제

8번 회원을 만들기 위해 `id=8`을 직접 INSERT했더니 PostgreSQL에서 `GENERATED ALWAYS` 컬럼에는 값을 직접 넣을 수 없다는 오류가 발생했다. 현재 `id`는 DB가 자동으로 생성하도록 설정된 IDENTITY 컬럼이었기 때문이다.

이번에는 교수님과 동일한 조회 조건을 만들기 위한 실습이어서 `OVERRIDING SYSTEM VALUE`를 사용해 명시적으로 8번을 등록하고, 이후 자동 생성 ID가 충돌하지 않도록 sequence도 8에 맞췄다.

일반적인 회원 등록 API에서는 id를 직접 지정하지 않고 DB에 맡기는 것이 올바르다. 이번 처리는 수업의 조회 Evidence를 동일하게 만들기 위한 예외적인 실습이라는 점도 함께 이해했다.

### 4. 터미널에서 한글 이름이 깨져 보인 문제

회원 등록 후 PowerShell 출력에서 `홍길동`이 깨진 문자처럼 보였다. 처음에는 DB에도 잘못 저장된 줄 알았지만, Python에서 문자열의 Unicode code point를 확인해 보니 `\ud64d\uae38\ub3d9`으로 정확하게 저장되어 있었다.

즉 데이터 자체가 손상된 것이 아니라 터미널 출력 인코딩 문제였다. DBeaver나 API JSON 결과처럼 UTF-8을 정상적으로 표시하는 화면에서 다시 확인해야 한다는 것을 배웠다. 앞으로 한글이 깨져 보이면 바로 데이터를 다시 입력하지 않고, 저장된 Unicode 값과 클라이언트 인코딩을 먼저 구분해서 확인할 것이다.

## 오후 수업을 마치며 느낀 점

PATCH는 단순한 UPDATE 한 줄이 아니라는 점이 가장 인상적이었다. 사용자가 어떤 필드를 보냈는지, 보내지 않은 필드는 어떻게 보존할지, null을 허용할지, 수정 대상이 존재하는지, 이메일이 다른 데이터와 충돌하지 않는지까지 차례대로 판단해야 했다.

또한 Swagger에서 200이 나왔다는 사실만으로 성공이라고 판단하면 부족하다는 것을 다시 느꼈다. API 응답, GET 재조회, PostgreSQL row가 모두 같아야 수정이 실제로 올바르게 반영됐다고 말할 수 있었다. 특히 `id=8` 조회가 실패했을 때 코드를 먼저 고치지 않고 DB의 실제 데이터를 확인한 과정이 도움이 되었다.

오늘 오전에는 POST와 Transaction을 통해 데이터를 안전하게 생성하는 방법을 배웠고, 오후에는 PATCH와 UPDATE를 통해 기존 데이터를 일부만 안전하게 변경하는 방법을 배웠다. 두 수업을 연결해 보니 API 요청, Pydantic 검증, SQL 실행, commit, 응답, DB Evidence까지 하나의 흐름으로 이해할 수 있었다.

## 추가로 복습할 내용

- PATCH에서 `{}`와 `{"field": null}`이 서로 다른 이유 다시 설명해 보기
- 존재하지 않는 회원을 수정했을 때 404가 반환되는지 테스트하기
- 다른 회원의 이메일로 변경하여 409가 반환되는지 테스트하기
- PATCH 전후에 보내지 않은 필드가 그대로 유지되는지 비교하기
- 다음 DELETE 수업에서 삭제 전 조회, DELETE, 삭제 후 404까지 Evidence로 남기기
