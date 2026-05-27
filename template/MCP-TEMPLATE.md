# MCP Server Project Template

이 문서는 프로젝트를 처음부터 완전히 동일하게 재현하기 위한 템플릿입니다. 아래 파일 구조와 각 파일의 전체 내용을 그대로 따르세요.

---

## 프로젝트 개요

- **목적**: FastMCP 기반 MCP (Model Context Protocol) 서버
- **전송 방식**: Streamable HTTP + SSE (포트 8000)
- **Python 버전**: 3.12
- **핵심 구조**: `tools/`, `resources/`, `prompts/` 폴더에 파일을 추가하면 `__init__.py`의 `pkgutil.walk_packages`로 자동 등록

---

## 디렉토리 구조

```
mcp/
├── .env.example
├── .gitignore
├── main.py
├── pyrightconfig.json
├── requirements.txt
├── server.py
├── tools/
│   ├── __init__.py
│   ├── calculator.py
│   └── weather.py
├── resources/
│   ├── __init__.py
│   └── examples.py
└── prompts/
    ├── __init__.py
    └── examples.py
```

---

## 파일 내용

### `.env.example`

```
APP_ENV=local
```

### `.gitignore`

```
.env
.venv/
__pycache__/
```

### `pyrightconfig.json`

```json
{
  "venvPath": ".",
  "venv": ".venv",
  "pythonVersion": "3.12",
  "typeCheckingMode": "basic"
}
```

### `requirements.txt`

```
mcp>=1.12.0
fastapi>=0.104.0
uvicorn[standard]>=0.27.0
python-dotenv>=1.0.0
```

### `server.py`

```python
# pyright: reportMissingImports=false
from contextlib import asynccontextmanager

from fastapi import FastAPI
from mcp.server.fastmcp import FastMCP
from mcp.server.transport_security import TransportSecuritySettings
from starlette.applications import Starlette


mcp = FastMCP(
    "MCP Server",
    stateless_http=True,
    json_response=True,
    streamable_http_path="/",
    transport_security=TransportSecuritySettings(
        enable_dns_rebinding_protection=False,
    ),
)

# tools/, resources/, prompts/ 디렉토리에 파일을 추가하면 자동으로 등록됩니다.
import tools  # noqa: F401, E402
import resources  # noqa: F401, E402
import prompts  # noqa: F401, E402

# Streamable HTTP + SSE 동시 지원
mcp_http_app = mcp.streamable_http_app()
mcp_sse_app = mcp.sse_app()
mcp_combined_app = Starlette(
    routes=[*mcp_http_app.routes, *mcp_sse_app.routes],
)


@asynccontextmanager
async def lifespan(_: FastAPI):
    async with mcp.session_manager.run():
        yield


app = FastAPI(title="MCP Server", lifespan=lifespan)


@app.get("/health")
def health_check():
    return {"status": "ok"}


# /mcp 아래에 streamable HTTP + SSE 모두 마운트
# - POST /mcp/       → Streamable HTTP (JSON-RPC)
# - GET  /mcp/sse    → SSE 연결
# - POST /mcp/messages/ → SSE 메시지 전송
app.mount("/mcp", mcp_combined_app)
```

### `main.py`

```python
import os
from dotenv import load_dotenv

load_dotenv(os.path.join(os.path.dirname(os.path.abspath(__file__)), ".env"))

import uvicorn
from server import app

if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### `tools/__init__.py`

```python
import pkgutil
import importlib

for loader, module_name, is_pkg in pkgutil.walk_packages(__path__):
    importlib.import_module(f"{__name__}.{module_name}")
```

### `tools/calculator.py`

```python
from server import mcp


@mcp.tool()
def add(a: int, b: int) -> int:
    """두 숫자를 더합니다."""
    return a + b


@mcp.tool()
def multiply(a: int, b: int) -> int:
    """두 숫자를 곱합니다."""
    return a * b
```

### `tools/weather.py`

```python
from server import mcp


@mcp.tool()
def get_weather(city: str) -> str:
    """도시의 날씨 정보를 반환합니다. (샘플 데이터)"""
    weather_data = {
        "서울": "맑음, 22°C",
        "부산": "흐림, 19°C",
        "제주": "비, 18°C",
    }
    return weather_data.get(city, f"{city}의 날씨 정보를 찾을 수 없습니다.")
```

### `resources/__init__.py`

```python
import pkgutil
import importlib

for loader, module_name, is_pkg in pkgutil.walk_packages(__path__):
    importlib.import_module(f"{__name__}.{module_name}")
```

### `resources/examples.py`

```python
from server import mcp


@mcp.resource("info://server")
def server_info() -> str:
    """MCP 서버 정보를 반환합니다."""
    return "AX Dev MCP Server Template v1.0"


@mcp.resource("greeting://{name}")
def greeting(name: str) -> str:
    """이름으로 인사말을 생성합니다."""
    return f"안녕하세요, {name}님! MCP 서버에 오신 것을 환영합니다."
```

### `prompts/__init__.py`

```python
import pkgutil
import importlib

for loader, module_name, is_pkg in pkgutil.walk_packages(__path__):
    importlib.import_module(f"{__name__}.{module_name}")
```

### `prompts/examples.py`

```python
from server import mcp


@mcp.prompt()
def summarize(text: str) -> str:
    """텍스트를 요약하는 프롬프트를 생성합니다."""
    return f"다음 텍스트를 3줄 이내로 요약해주세요:\n\n{text}"


@mcp.prompt()
def translate(text: str, target_language: str = "영어") -> str:
    """텍스트를 번역하는 프롬프트를 생성합니다."""
    return f"다음 텍스트를 {target_language}로 번역해주세요:\n\n{text}"
```

---

## 재현 절차

1. 위 디렉토리 구조대로 폴더와 파일을 생성합니다.
2. 각 파일의 내용을 그대로 복사합니다.
3. `.env.example`을 `.env`로 복사하여 필요한 환경변수를 설정합니다.
4. 가상환경 생성 및 의존성 설치:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

5. 서버 실행:

```bash
python main.py
```

6. 검증:

```bash
# 자동 임포트 에러 확인
python -c "import tools; import resources; import prompts"

# 헬스 체크
curl http://localhost:8000/health
```

---

## 새 도구/리소스/프롬프트 추가 규칙

- `tools/`, `resources/`, `prompts/` 폴더에 새 `.py` 파일을 만들면 자동 등록됩니다.
- 반드시 `from server import mcp`로 `mcp` 인스턴스를 import하여 데코레이터를 사용합니다.
- `__init__.py`는 절대 수정하지 않습니다.
- import 에러가 발생하면 전체 서버가 시작되지 않으므로 주의합니다.

---

## 핵심 설계 원칙

| 원칙 | 설명 |
|------|------|
| 자동 등록 | `pkgutil.walk_packages`로 폴더 내 모든 모듈을 자동 import |
| 단일 진입점 | `server.py`의 `mcp` 인스턴스 하나로 모든 도구/리소스/프롬프트 관리 |
| Streamable HTTP + SSE | `/mcp/` 경로 아래에 두 전송 방식 동시 마운트 |
| 환경변수 분리 | `.env` 파일로 환경 설정, 코드에 하드코딩 금지 |
| Docker 배포 | Coder 워크스페이스 기반, venv를 tar로 캐싱하여 빠른 시작 |
