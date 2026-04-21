# AI 비서 Plan 1: 인프라 + Notion 연동 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ntfy 알림 서버와 Notion API 연동을 완료해, Python 코드로 일정 DB·가계부 DB를 읽고 쓰고, iPhone으로 알림을 보낼 수 있는 기반을 구축한다.

**Architecture:** Python 서버(Windows)가 Notion API로 두 개의 DB(일정, 가계부)를 읽고 쓴다. 알림은 Docker로 실행한 ntfy 서버가 Cloudflare Tunnel을 통해 외부에서 접근 가능하게 된다. `상태` 속성은 Notion API 제약으로 Select 타입으로 생성한다(기능 동일).

**Tech Stack:** Python 3.11+, notion-client 2.x, requests, python-dotenv, pytest, Docker Desktop, cloudflared

---

## 파일 구조

```
C:\ai-assistant\
├── .env                     # API 키 (git 제외)
├── .env.example             # 키 양식 (git 포함)
├── .gitignore
├── requirements.txt
├── setup_notion.py          # 최초 1회 DB 생성 스크립트
├── notion.py                # Notion API 함수 (읽기/쓰기)
├── notify.py                # ntfy 알림 함수
└── tests\
    ├── __init__.py
    ├── conftest.py          # pytest fixtures (Notion 테스트 데이터)
    ├── test_notify.py
    ├── test_notion.py
    └── test_ledger.py
```

---

## Task 1: 프로젝트 스캐폴딩

**Files:**
- Create: `C:\ai-assistant\.gitignore`
- Create: `C:\ai-assistant\.env.example`
- Create: `C:\ai-assistant\requirements.txt`
- Create: `C:\ai-assistant\tests\__init__.py`

- [ ] **Step 1: .gitignore 작성**

`C:\ai-assistant\.gitignore`:
```
.env
__pycache__/
*.pyc
.pytest_cache/
```

- [ ] **Step 2: .env.example 작성**

`C:\ai-assistant\.env.example`:
```
ANTHROPIC_API_KEY=sk-ant-...
NOTION_API_KEY=secret_...
NOTION_SCHEDULE_DB_ID=
NOTION_LEDGER_DB_ID=
NTFY_URL=https://xxxx.cfargotunnel.com/my-assistant-a7f3k
```

- [ ] **Step 3: requirements.txt 작성**

`C:\ai-assistant\requirements.txt`:
```
anthropic
notion-client
requests
python-dotenv
schedule
flask
pytest
```

- [ ] **Step 4: .env 파일 생성**

PowerShell:
```powershell
Copy-Item .env.example .env
```

- [ ] **Step 5: 패키지 설치**

```bash
pip install -r requirements.txt
```

Expected: 모든 패키지 설치 완료 출력

- [ ] **Step 6: tests\__init__.py 생성 (빈 파일)**

- [ ] **Step 7: git 초기화 및 첫 커밋**

```bash
git init
git add .gitignore .env.example requirements.txt tests/__init__.py
git commit -m "chore: project scaffolding"
```

---

## Task 2: ntfy 알림 서버 + notify.py

**Files:**
- Create: `C:\ai-assistant\notify.py`
- Create: `C:\ai-assistant\tests\test_notify.py`

- [ ] **Step 1: Docker Desktop 설치 확인**

```powershell
docker --version
```

Expected: `Docker version 2x.x.x` 출력. 없으면 https://www.docker.com/products/docker-desktop 에서 설치.

- [ ] **Step 2: ntfy 컨테이너 실행**

```powershell
docker run -d --name ntfy -p 80:80 -v ntfy-data:/var/lib/ntfy binwiederhier/ntfy serve
```

- [ ] **Step 3: ntfy 동작 확인**

브라우저에서 `http://localhost` 접속 → ntfy 웹 UI 확인

- [ ] **Step 4: failing test 작성**

`tests\test_notify.py`:
```python
import os
import pytest
from unittest.mock import patch, Mock
from notify import send


def test_send_default_priority():
    mock_response = Mock()
    mock_response.status_code = 200

    with patch("notify.requests.post", return_value=mock_response) as mock_post:
        with patch.dict(os.environ, {"NTFY_URL": "http://localhost/test-channel"}):
            send("테스트 제목", "테스트 내용")

    call_kwargs = mock_post.call_args
    assert call_kwargs.kwargs["headers"]["Title"] == "테스트 제목"
    assert call_kwargs.kwargs["data"] == "테스트 내용".encode("utf-8")
    assert call_kwargs.kwargs["headers"]["Priority"] == "default"


def test_send_urgent_priority():
    mock_response = Mock()
    mock_response.status_code = 200

    with patch("notify.requests.post", return_value=mock_response) as mock_post:
        with patch.dict(os.environ, {"NTFY_URL": "http://localhost/test-channel"}):
            send("긴급", "긴급 내용", priority="urgent")

    call_kwargs = mock_post.call_args
    assert call_kwargs.kwargs["headers"]["Priority"] == "urgent"
```

- [ ] **Step 5: 테스트 실패 확인**

```bash
pytest tests/test_notify.py -v
```

Expected: `ModuleNotFoundError: No module named 'notify'`

- [ ] **Step 6: notify.py 구현**

`notify.py`:
```python
import os
import requests
from dotenv import load_dotenv

load_dotenv()


def send(title: str, body: str, priority: str = "default") -> None:
    requests.post(
        os.getenv("NTFY_URL"),
        data=body.encode("utf-8"),
        headers={
            "Title": title,
            "Priority": priority,
            "Tags": "calendar",
        },
    )
```

- [ ] **Step 7: 테스트 통과 확인**

```bash
pytest tests/test_notify.py -v
```

Expected: `2 passed`

- [ ] **Step 8: .env에 NTFY_URL 입력 후 실제 알림 수신 확인**

`.env`:
```
NTFY_URL=http://localhost/my-assistant-a7f3k
```

iPhone에 ntfy 앱 설치 → `http://localhost/my-assistant-a7f3k` 구독.

Python에서 직접 테스트:
```python
from dotenv import load_dotenv
load_dotenv()
from notify import send
send("테스트", "iPhone에서 알림이 왔나요?", priority="high")
```

Expected: iPhone ntfy 앱에 알림 수신

- [ ] **Step 9: 커밋**

```bash
git add notify.py tests/test_notify.py
git commit -m "feat: add ntfy notification module"
```

---

## Task 3: Cloudflare Tunnel 설정 (외부 접근)

이 Task는 수동 설정 단계다. 코드 변경 없음.

- [ ] **Step 1: cloudflared 설치**

PowerShell (관리자 권한):
```powershell
winget install Cloudflare.cloudflared
```

- [ ] **Step 2: Tunnel 실행**

```powershell
cloudflared tunnel --url http://localhost:80
```

Expected: `https://xxxx.cfargotunnel.com` 형태의 주소 출력

- [ ] **Step 3: .env NTFY_URL 업데이트**

`.env`:
```
NTFY_URL=https://xxxx.cfargotunnel.com/my-assistant-a7f3k
```

(`xxxx` 부분을 실제 출력된 주소로 교체)

- [ ] **Step 4: 외부 접근 알림 테스트**

```python
from dotenv import load_dotenv
load_dotenv()
from notify import send
send("외부 접근 테스트", "Cloudflare Tunnel 경유 알림", priority="high")
```

Expected: iPhone에서 알림 수신 (Wi-Fi 끄고 LTE로 테스트 권장)

- [ ] **Step 5: 커밋**

```bash
git add .env.example
git commit -m "docs: update env example with cloudflare tunnel url"
```

---

## Task 4: Notion Integration + DB 생성

- [ ] **Step 1: Notion Integration 생성**

1. https://www.notion.so/my-integrations 접속
2. "새 API 통합" 클릭, 이름: "AI 비서", 워크스페이스 선택 후 제출
3. 생성된 `secret_...` 형태의 토큰 복사
4. `.env`에 입력:
   ```
   NOTION_API_KEY=secret_...
   ```

- [ ] **Step 2: Notion에 AI 비서 전용 페이지 생성**

Notion에서 새 빈 페이지 생성 (이름: "AI 비서").

페이지 우측 상단 `...` → "연결 추가" → "AI 비서" 통합 선택.

페이지 URL에서 ID 복사:
`https://notion.so/AI-비서-**{PAGE_ID}**?v=...` → 하이픈 포함 32자

- [ ] **Step 3: setup_notion.py 작성**

`setup_notion.py`:
```python
import os
from notion_client import Client
from dotenv import load_dotenv

load_dotenv()
notion = Client(auth=os.getenv("NOTION_API_KEY"))
PARENT_PAGE_ID = input("AI 비서 Notion 페이지 ID를 입력하세요 (하이픈 포함): ").strip().replace("-", "")


def create_schedule_db() -> str:
    db = notion.databases.create(
        parent={"page_id": PARENT_PAGE_ID},
        title=[{"text": {"content": "AI 비서 일정"}}],
        properties={
            "일정 이름": {"title": {}},
            "실행 시간": {"date": {}},
            "작업 유형": {"select": {"options": [
                {"name": "자소서", "color": "pink"},
                {"name": "면접", "color": "blue"},
                {"name": "공부", "color": "green"},
                {"name": "운동", "color": "orange"},
                {"name": "약속", "color": "purple"},
                {"name": "업무", "color": "gray"},
                {"name": "기타", "color": "default"},
            ]}},
            "상태": {"select": {"options": [
                {"name": "제안됨", "color": "gray"},
                {"name": "확정", "color": "blue"},
                {"name": "완료", "color": "green"},
                {"name": "취소", "color": "red"},
            ]}},
            "우선순위": {"select": {"options": [
                {"name": "높음", "color": "red"},
                {"name": "보통", "color": "yellow"},
                {"name": "낮음", "color": "gray"},
            ]}},
            "마감일": {"date": {}},
            "원래 목표": {"rich_text": {}},
            "하위 계획": {"rich_text": {}},
            "추정 근거": {"rich_text": {}},
            "실제 소요 시간(분)": {"number": {"format": "number"}},
            "AI 생성 여부": {"checkbox": {}},
            "설명": {"rich_text": {}},
        },
    )
    return db["id"]


def create_ledger_db() -> str:
    db = notion.databases.create(
        parent={"page_id": PARENT_PAGE_ID},
        title=[{"text": {"content": "가계부"}}],
        properties={
            "이름": {"title": {}},
            "금액": {"number": {"format": "won"}},
            "카테고리": {"select": {"options": [
                {"name": "식비", "color": "orange"},
                {"name": "교통", "color": "blue"},
                {"name": "쇼핑", "color": "pink"},
                {"name": "구독", "color": "purple"},
                {"name": "기타", "color": "gray"},
            ]}},
            "날짜": {"date": {}},
            "결제수단": {"select": {"options": [
                {"name": "카드", "color": "blue"},
                {"name": "현금", "color": "green"},
                {"name": "계좌이체", "color": "gray"},
            ]}},
            "메모": {"rich_text": {}},
        },
    )
    return db["id"]


if __name__ == "__main__":
    schedule_db_id = create_schedule_db()
    print(f"일정 DB 생성 완료: {schedule_db_id}")

    ledger_db_id = create_ledger_db()
    print(f"가계부 DB 생성 완료: {ledger_db_id}")

    print("\n.env에 아래 값을 추가하세요:")
    print(f"NOTION_SCHEDULE_DB_ID={schedule_db_id}")
    print(f"NOTION_LEDGER_DB_ID={ledger_db_id}")
```

- [ ] **Step 4: 스크립트 실행**

```bash
python setup_notion.py
```

Expected:
```
AI 비서 Notion 페이지 ID를 입력하세요: xxxxxxxx...
일정 DB 생성 완료: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
가계부 DB 생성 완료: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

.env에 아래 값을 추가하세요:
NOTION_SCHEDULE_DB_ID=xxxxxxxx...
NOTION_LEDGER_DB_ID=xxxxxxxx...
```

- [ ] **Step 5: .env에 두 DB ID 입력**

- [ ] **Step 6: Notion Calendar 연결**

Notion Calendar 앱(App Store)에서 "AI 비서" 계정 연결 후, "AI 비서 일정" DB의 `실행 시간` 속성을 캘린더 기준으로 설정.

- [ ] **Step 7: 커밋**

```bash
git add setup_notion.py
git commit -m "feat: add notion DB setup script"
```

---

## Task 5: notion.py — 일정 DB 읽기

**Files:**
- Create: `C:\ai-assistant\notion.py`
- Create: `C:\ai-assistant\tests\conftest.py`
- Create: `C:\ai-assistant\tests\test_notion.py`

- [ ] **Step 1: conftest.py 작성 (테스트 fixtures)**

`tests\conftest.py`:
```python
import os
import pytest
from datetime import datetime, timedelta, timezone
from notion_client import Client
from dotenv import load_dotenv

load_dotenv()
KST = timezone(timedelta(hours=9))
_notion = Client(auth=os.getenv("NOTION_API_KEY"))
SCHEDULE_DB_ID = os.getenv("NOTION_SCHEDULE_DB_ID")


@pytest.fixture
def sample_confirmed_event():
    start = (datetime.now(KST) + timedelta(hours=1)).strftime("%Y-%m-%dT%H:%M:%S+09:00")
    end = (datetime.now(KST) + timedelta(hours=3)).strftime("%Y-%m-%dT%H:%M:%S+09:00")
    page = _notion.pages.create(
        parent={"database_id": SCHEDULE_DB_ID},
        properties={
            "일정 이름": {"title": [{"text": {"content": "테스트 확정 일정"}}]},
            "실행 시간": {"date": {"start": start, "end": end}},
            "상태": {"select": {"name": "확정"}},
            "작업 유형": {"select": {"name": "면접"}},
        },
    )
    yield page
    _notion.pages.update(page_id=page["id"], archived=True)


@pytest.fixture
def sample_completed_event():
    start = (datetime.now(KST) - timedelta(days=3)).strftime("%Y-%m-%dT10:00:00+09:00")
    end = (datetime.now(KST) - timedelta(days=3)).strftime("%Y-%m-%dT12:00:00+09:00")
    page = _notion.pages.create(
        parent={"database_id": SCHEDULE_DB_ID},
        properties={
            "일정 이름": {"title": [{"text": {"content": "테스트 완료 일정"}}]},
            "실행 시간": {"date": {"start": start, "end": end}},
            "상태": {"select": {"name": "완료"}},
            "작업 유형": {"select": {"name": "면접"}},
            "실제 소요 시간(분)": {"number": 90},
        },
    )
    yield page
    _notion.pages.update(page_id=page["id"], archived=True)
```

- [ ] **Step 2: failing test 작성**

`tests\test_notion.py`:
```python
import os
import pytest
from datetime import datetime, timedelta, timezone
from notion_client import Client
from dotenv import load_dotenv
from notion import get_confirmed_events, get_completed_events_by_type

load_dotenv()
KST = timezone(timedelta(hours=9))
_notion = Client(auth=os.getenv("NOTION_API_KEY"))


def test_get_confirmed_events_returns_list():
    start = datetime.now(KST).isoformat()
    end = (datetime.now(KST) + timedelta(days=7)).isoformat()
    result = get_confirmed_events(start, end)
    assert isinstance(result, list)


def test_get_confirmed_events_only_confirmed(sample_confirmed_event):
    start = datetime.now(KST).isoformat()
    end = (datetime.now(KST) + timedelta(days=7)).isoformat()
    result = get_confirmed_events(start, end)
    for event in result:
        assert event["status"] == "확정"


def test_get_completed_events_by_type_returns_list():
    result = get_completed_events_by_type("면접")
    assert isinstance(result, list)


def test_get_completed_events_by_type_only_completed(sample_completed_event):
    result = get_completed_events_by_type("면접")
    for event in result:
        assert event["status"] == "완료"
        assert event["work_type"] == "면접"
```

- [ ] **Step 3: 테스트 실패 확인**

```bash
pytest tests/test_notion.py -v
```

Expected: `ModuleNotFoundError: No module named 'notion'`

- [ ] **Step 4: notion.py 읽기 함수 구현**

`notion.py`:
```python
import os
from notion_client import Client
from dotenv import load_dotenv

load_dotenv()
notion = Client(auth=os.getenv("NOTION_API_KEY"))
SCHEDULE_DB_ID = os.getenv("NOTION_SCHEDULE_DB_ID")
LEDGER_DB_ID = os.getenv("NOTION_LEDGER_DB_ID")


def _parse_event(page: dict) -> dict:
    props = page["properties"]
    title = props["일정 이름"]["title"]
    date = props["실행 시간"]["date"]
    return {
        "id": page["id"],
        "name": title[0]["text"]["content"] if title else "",
        "start": date["start"] if date else None,
        "end": date.get("end") if date else None,
        "status": props["상태"]["select"]["name"] if props["상태"]["select"] else None,
        "work_type": props["작업 유형"]["select"]["name"] if props["작업 유형"]["select"] else None,
        "actual_minutes": props["실제 소요 시간(분)"]["number"],
        "deadline": props["마감일"]["date"]["start"] if props["마감일"]["date"] else None,
    }


def get_confirmed_events(start_iso: str, end_iso: str) -> list[dict]:
    results = notion.databases.query(
        database_id=SCHEDULE_DB_ID,
        filter={
            "and": [
                {"property": "상태", "select": {"equals": "확정"}},
                {"property": "실행 시간", "date": {"on_or_after": start_iso}},
                {"property": "실행 시간", "date": {"before": end_iso}},
            ]
        },
    ).get("results", [])
    return [_parse_event(p) for p in results]


def get_completed_events_by_type(work_type: str) -> list[dict]:
    results = notion.databases.query(
        database_id=SCHEDULE_DB_ID,
        filter={
            "and": [
                {"property": "상태", "select": {"equals": "완료"}},
                {"property": "작업 유형", "select": {"equals": work_type}},
            ]
        },
        sorts=[{"property": "실행 시간", "direction": "descending"}],
    ).get("results", [])
    return [_parse_event(p) for p in results]
```

- [ ] **Step 5: 테스트 통과 확인**

```bash
pytest tests/test_notion.py -v
```

Expected: `4 passed`

- [ ] **Step 6: 커밋**

```bash
git add notion.py tests/test_notion.py tests/conftest.py
git commit -m "feat: add notion schedule DB read functions"
```

---

## Task 6: notion.py — 일정 DB 쓰기

**Files:**
- Modify: `C:\ai-assistant\notion.py`
- Modify: `C:\ai-assistant\tests\test_notion.py`

- [ ] **Step 1: failing test 추가**

`tests\test_notion.py` 상단 import에 추가:
```python
from notion import add_schedule_event, update_event_status, update_event_time
```

파일 하단에 추가:
```python
def test_add_schedule_event():
    start = (datetime.now(KST) + timedelta(days=1)).strftime("%Y-%m-%dT20:00:00+09:00")
    end = (datetime.now(KST) + timedelta(days=1)).strftime("%Y-%m-%dT21:30:00+09:00")
    page_id = add_schedule_event(
        name="토스 면접 준비 - 예상 질문 정리",
        start_iso=start,
        end_iso=end,
        work_type="면접",
        status="확정",
        priority="보통",
        deadline="2026-04-25",
        original_goal="다음 주 금요일까지 토스 면접 준비해야 해",
        sub_plan="예상 질문 정리",
        estimation_basis="최근 면접 기록 3개 평균 5시간 20분",
        ai_generated=True,
        description="회사/직무/경험 기반 예상 질문 후보 수집",
    )
    assert isinstance(page_id, str)
    assert len(page_id) > 0
    _notion.pages.update(page_id=page_id, archived=True)


def test_update_event_status():
    start = (datetime.now(KST) + timedelta(days=2)).strftime("%Y-%m-%dT10:00:00+09:00")
    end = (datetime.now(KST) + timedelta(days=2)).strftime("%Y-%m-%dT11:00:00+09:00")
    page_id = add_schedule_event(
        name="상태 변경 테스트", start_iso=start, end_iso=end, work_type="공부", status="확정"
    )
    update_event_status(page_id, "완료")
    page = _notion.pages.retrieve(page_id=page_id)
    assert page["properties"]["상태"]["select"]["name"] == "완료"
    _notion.pages.update(page_id=page_id, archived=True)


def test_update_event_time():
    start = (datetime.now(KST) + timedelta(days=3)).strftime("%Y-%m-%dT10:00:00+09:00")
    end = (datetime.now(KST) + timedelta(days=3)).strftime("%Y-%m-%dT11:00:00+09:00")
    page_id = add_schedule_event(
        name="시간 변경 테스트", start_iso=start, end_iso=end, work_type="공부", status="확정"
    )
    new_start = (datetime.now(KST) + timedelta(days=4)).strftime("%Y-%m-%dT14:00:00+09:00")
    new_end = (datetime.now(KST) + timedelta(days=4)).strftime("%Y-%m-%dT15:00:00+09:00")
    update_event_time(page_id, new_start, new_end)
    page = _notion.pages.retrieve(page_id=page_id)
    assert page["properties"]["실행 시간"]["date"]["start"] == new_start
    _notion.pages.update(page_id=page_id, archived=True)
```

- [ ] **Step 2: 테스트 실패 확인**

```bash
pytest tests/test_notion.py::test_add_schedule_event -v
```

Expected: `ImportError: cannot import name 'add_schedule_event'`

- [ ] **Step 3: notion.py에 쓰기 함수 추가**

`notion.py` 하단에 추가:
```python
def add_schedule_event(
    name: str,
    start_iso: str,
    end_iso: str,
    work_type: str = "기타",
    status: str = "확정",
    priority: str = "보통",
    deadline: str | None = None,
    original_goal: str = "",
    sub_plan: str = "",
    estimation_basis: str = "",
    actual_minutes: int | None = None,
    ai_generated: bool = True,
    description: str = "",
) -> str:
    properties = {
        "일정 이름": {"title": [{"text": {"content": name}}]},
        "실행 시간": {"date": {"start": start_iso, "end": end_iso}},
        "작업 유형": {"select": {"name": work_type}},
        "상태": {"select": {"name": status}},
        "우선순위": {"select": {"name": priority}},
        "원래 목표": {"rich_text": [{"text": {"content": original_goal}}]},
        "하위 계획": {"rich_text": [{"text": {"content": sub_plan}}]},
        "추정 근거": {"rich_text": [{"text": {"content": estimation_basis}}]},
        "AI 생성 여부": {"checkbox": ai_generated},
        "설명": {"rich_text": [{"text": {"content": description}}]},
    }
    if deadline:
        properties["마감일"] = {"date": {"start": deadline}}
    if actual_minutes is not None:
        properties["실제 소요 시간(분)"] = {"number": actual_minutes}

    page = notion.pages.create(
        parent={"database_id": SCHEDULE_DB_ID},
        properties=properties,
    )
    return page["id"]


def update_event_status(page_id: str, status: str) -> None:
    notion.pages.update(
        page_id=page_id,
        properties={"상태": {"select": {"name": status}}},
    )


def update_event_time(page_id: str, start_iso: str, end_iso: str) -> None:
    notion.pages.update(
        page_id=page_id,
        properties={"실행 시간": {"date": {"start": start_iso, "end": end_iso}}},
    )
```

- [ ] **Step 4: 전체 테스트 통과 확인**

```bash
pytest tests/test_notion.py -v
```

Expected: `7 passed`

- [ ] **Step 5: 커밋**

```bash
git add notion.py tests/test_notion.py
git commit -m "feat: add notion schedule DB write functions"
```

---

## Task 7: notion.py — 가계부 DB 쓰기

**Files:**
- Modify: `C:\ai-assistant\notion.py`
- Create: `C:\ai-assistant\tests\test_ledger.py`

- [ ] **Step 1: failing test 작성**

`tests\test_ledger.py`:
```python
import os
import pytest
from notion_client import Client
from dotenv import load_dotenv
from notion import add_ledger_entry

load_dotenv()
_notion = Client(auth=os.getenv("NOTION_API_KEY"))


def test_add_ledger_entry():
    page_id = add_ledger_entry(
        name="스타벅스 아메리카노",
        amount=4500,
        category="식비",
        date_str="2026-04-20",
        payment_method="카드",
        memo="회의 중 구매",
    )
    assert isinstance(page_id, str)
    page = _notion.pages.retrieve(page_id=page_id)
    props = page["properties"]
    assert props["금액"]["number"] == 4500
    assert props["카테고리"]["select"]["name"] == "식비"
    _notion.pages.update(page_id=page_id, archived=True)
```

- [ ] **Step 2: 테스트 실패 확인**

```bash
pytest tests/test_ledger.py -v
```

Expected: `ImportError: cannot import name 'add_ledger_entry'`

- [ ] **Step 3: notion.py에 가계부 함수 추가**

`notion.py` 하단에 추가:
```python
def add_ledger_entry(
    name: str,
    amount: int,
    category: str,
    date_str: str,
    payment_method: str = "카드",
    memo: str = "",
) -> str:
    page = notion.pages.create(
        parent={"database_id": LEDGER_DB_ID},
        properties={
            "이름": {"title": [{"text": {"content": name}}]},
            "금액": {"number": amount},
            "카테고리": {"select": {"name": category}},
            "날짜": {"date": {"start": date_str}},
            "결제수단": {"select": {"name": payment_method}},
            "메모": {"rich_text": [{"text": {"content": memo}}]},
        },
    )
    return page["id"]
```

- [ ] **Step 4: 전체 테스트 통과 확인**

```bash
pytest tests/ -v
```

Expected: `10 passed` (notify 2 + notion 7 + ledger 1)

- [ ] **Step 5: 최종 커밋**

```bash
git add notion.py tests/test_ledger.py
git commit -m "feat: add ledger DB write function"
```

---

## Plan 1 완료 기준

- [ ] iPhone으로 ntfy 알림 수신 확인 (LTE 환경)
- [ ] Notion에 "AI 비서 일정" DB와 "가계부" DB 생성 확인
- [ ] `pytest tests/ -v` → 10 passed
- [ ] Notion Calendar 앱에서 "AI 비서 일정" DB 연결 확인
