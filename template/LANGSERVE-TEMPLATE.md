# LangServe AI Agent - 프로젝트 재현 템플릿

이 문서는 프로젝트를 처음부터 동일하게 재현하기 위한 완전한 명세서입니다.

---

## 프로젝트 개요

- **목적**: LangChain 체인을 LangServe로 REST API 엔드포인트로 자동 노출하는 백엔드 서버
- **핵심 동작**: `add_routes()`로 체인을 등록하면 invoke/batch/stream/playground 엔드포인트가 자동 생성됨
- **런타임**: Python 3.12, uvicorn, FastAPI
- **배포**: Docker 컨테이너 (Coder 워크스페이스 환경)

---

## 디렉토리 구조

```
langserver/
├── main.py                      # FastAPI 앱 진입점 (create_app, add_routes)
├── debug.py                     # 서버 없이 체인을 콘솔에서 테스트하는 스크립트
├── requirements.txt             # Python 의존성
├── Dockerfile                   # 배포용 Docker 이미지
├── .env.example                 # 환경변수 예시
├── scripts/
│   └── coder-init.sh           # Coder 워크스페이스 초기화 스크립트
└── src/
    ├── core/
    │   ├── config.py            # Pydantic Settings 환경변수 로딩
    │   └── llm.py              # ChatOpenAI 인스턴스 생성
    ├── prompts/
    │   ├── prompts.py          # 프롬프트 텍스트 (순수 문자열)
    │   └── templates.py        # PromptTemplate 객체 생성
    ├── services/
    │   └── chat_service.py     # Runnable chain 생성 서비스
    └── utils/
        └── logger.py           # 로깅 유틸리티
```

**참고**: `__init__.py` 파일 없음. Python 3.3+ implicit namespace packages 방식 사용.

---

## 각 파일의 전체 코드

### `.env.example`

```env
# Application Settings
APP_ENV=local
DEBUG=True
API_V1_STR=/api/v1


# Azure OpenAI Settings
LLM_BASE_URL=https://llm.lge-vs.com
LLM_MODEL=gpt-4o
LLM_API_KEY=your_azure_api_key
```

---

### `requirements.txt`

```txt
fastapi==0.109.0
httpx==0.27.0
ipykernel==7.2.0
jupyter==1.1.1
langchain>=0.3,<0.4
langchain-community>=0.3,<0.4
langchain-core>=0.3,<1.0
langchain-openai>=0.2,<1.0
langgraph>=0.4
langserve[all]==0.3.3
langsmith>=0.1.147,<0.4
numpy==1.26.4
pydantic>=2.7.4,<3.0
pydantic-settings>=2.4.0,<3.0
python-dotenv==1.0.1
tiktoken==0.12.0
uvicorn[standard]==0.27.0
```

---

### `src/core/config.py`

```python
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    PROJECT_NAME: str = "LangServe AI Agent"
    APP_ENV: str = os.getenv("APP_ENV", "local")  # 애플리케이션 실행 환경 (local, dev, prod 등)
    DEBUG: bool = os.getenv("DEBUG", "False").lower() == "true"  # 디버그 모드 활성화 여부
    API_V1_STR: str = "/api/v1"  # API 버전 접두사
    
    # LLM 설정
    LLM_BASE_URL: str = os.getenv("LLM_BASE_URL", "https://llm.lge-vs.com")  # LLM API 기본 주소
    LLM_MODEL: str = os.getenv("LLM_MODEL", "gpt-4o")  # 사용할 모델 이름
    LLM_API_KEY: str = os.getenv("LLM_API_KEY", "")  # LLM API 키

    class Config:
        env_file = ".env"  # .env 파일에서 설정 로드
        case_sensitive = True

settings = Settings()
```

---

### `src/core/llm.py`

```python
from langchain_openai import ChatOpenAI
from src.core.config import settings

def get_llm():
    """
    설정에 따라 구성된 ChatOpenAI 인스턴스를 반환합니다.
    """
    return ChatOpenAI(
        openai_api_base=settings.LLM_BASE_URL,
        model=settings.LLM_MODEL,
        openai_api_key =settings.LLM_API_KEY,
        temperature=0.7
    )
```

---

### `src/prompts/prompts.py`

```python
# 기본 채팅 프롬프트
BASIC_CHAT_PROMPT = """
당신은 글쓰기 전문가입니다.

{question} 주제에 대해서 5줄 이내의 시를 써주세요.
"""
```

---

### `src/prompts/templates.py`

```python
from langchain_core.prompts import PromptTemplate
from src.prompts.prompts import BASIC_CHAT_PROMPT

# prompts.py에서 텍스트를 가져와 템플릿 생성
CHAT_PROMPT = PromptTemplate.from_template(BASIC_CHAT_PROMPT)
```

---

### `src/services/chat_service.py`

```python
from langchain_core.output_parsers import StrOutputParser
from src.core.llm import get_llm
from src.prompts.templates import CHAT_PROMPT
from src.utils.logger import get_logger

logger = get_logger(__name__)

def get_chat_chain():
    """
    LangServe에서 사용할 수 있는 Runnable chain을 반환합니다.
    """
    try:
        llm = get_llm()
        chain = CHAT_PROMPT | llm | StrOutputParser()
        logger.info("Chat chain이 성공적으로 초기화되었습니다.")
        return chain
    except Exception as e:
        logger.error(f"Chat chain 초기화 실패: {e}")
        raise e

# LangServe용 체인 생성
try:
    chat_chain = get_chat_chain()
except Exception as e:
    logger.error(f"Chat chain 초기화 실패: {e}")
    chat_chain = None
```

---

### `src/utils/logger.py`

```python
import logging
import sys

def get_logger(name: str):
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    
    # 중복 핸들러 방지
    if not logger.handlers:
        handler = logging.StreamHandler(sys.stdout)
        handler.setFormatter(logging.Formatter(
            "[%(asctime)s] %(levelname)s in %(module)s: %(message)s"
        ))
        logger.addHandler(handler)
        
    return logger
```

---

### `main.py`

```python
import os
import uvicorn
from dotenv import load_dotenv

# 설정을 사용하는 모듈을 import 하기 전에 환경변수 먼저 로드
dotenv_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), ".env")
load_dotenv(dotenv_path=dotenv_path)

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from langserve import add_routes
from src.core.config import settings
from src.services.chat_service import chat_chain
from src.utils.logger import get_logger

logger = get_logger(__name__)

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.PROJECT_NAME,
        openapi_url=f"{settings.API_V1_STR}/openapi.json",
        debug=settings.DEBUG
    )

    # CORS 미들웨어 (샘플용으로 모든 origin 허용)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # LangServe 라우트 추가 (체인을 /chat 엔드포인트로 노출)
    if chat_chain:
        add_routes(
            app,
            chat_chain,
            path=f"{settings.API_V1_STR}/chat",
            enabled_endpoints=[
                "invoke",
                "batch",
                "stream",
                "stream_log",
                "stream_events",
                "playground",
                "input_schema",
                "output_schema",
                "config_schema",
                "config_hashes",
            ],
            playground_type="default",
        )
        logger.info("LangServe 라우트가 /chat 경로에 추가되었습니다.")
    else:
        logger.error("Chat chain을 사용할 수 없습니다 - LangServe 라우트가 추가되지 않았습니다.")

    @app.get("/health")
    def health_check():
        return {"status": "ok", "env": settings.APP_ENV}

    return app

app = create_app()

if __name__ == "__main__":
    logger.info(f"LangServe 서버를 8000번 포트에서 시작합니다. (Debug={settings.DEBUG})")
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

---

### `debug.py`

```python
import asyncio
import os

from dotenv import load_dotenv


dotenv_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), ".env")
load_dotenv(dotenv_path=dotenv_path)

from src.services.chat_service import chat_chain
from src.utils.logger import get_logger


logger = get_logger("debug_script")


async def debug_chat() -> None:
    """Run local console chat without starting the HTTP server."""
    print("=== 디버그 모드: LangServe Chat Chain (종료: exit/quit) ===")

    if not chat_chain:
        print("오류: Chat chain 초기화 실패. .env 설정을 확인하세요.")
        return

    while True:
        query = input("\n사용자: ").strip()
        if query.lower() in {"exit", "quit"}:
            break
        if not query:
            continue

        try:
            print("생각 중...", end="\r")
            response = await chat_chain.ainvoke({"question": query})
            print(f"AI: {str(response).strip()}")
        except Exception as error:
            logger.error("오류: %s", error)


if __name__ == "__main__":
    try:
        asyncio.run(debug_chat())
    except KeyboardInterrupt:
        print("\n종료합니다...")
```

---

### `Dockerfile`

```dockerfile
FROM python:3.12-slim

# 필수 시스템 패키지 설치
USER root
RUN apt-get update && apt-get install -y \
    git \
    curl \
    wget \
    ca-certificates \
    procps \
    bash \
    sudo \
    && rm -rf /var/lib/apt/lists/*

# coder 사용자 생성 및 sudo 권한 부여
RUN useradd -m -s /bin/bash coder && \
    echo "coder ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    mkdir -p /workspaces && \
    chown -R coder:coder /workspaces

# 샘플 코드를 템플릿 위치에 복사 (PVC 마운트로 사라지지 않는 위치)
RUN mkdir -p /opt/workspace-template
COPY --chown=coder:coder . /opt/workspace-template/langserver

# Python 패키지를 venv로 사전 빌드 후 tar로 패킹 (NFS 위 pip install 회피)
RUN python3 -m venv /workspaces/.venv/langserver && \
    /workspaces/.venv/langserver/bin/pip install --no-cache-dir -r /opt/workspace-template/langserver/requirements.txt && \
    tar cf /opt/workspace-template/venv.tar -C /workspaces .venv/langserver && \
    rm -rf /workspaces/.venv

# 이미지 태그 기록 (venv 버전 추적용)
ARG IMAGE_TAG=latest
RUN echo "${IMAGE_TAG}" > /opt/workspace-template/.image-tag

# code-server 사전 설치 (워크스페이스 시작 시간 단축)
RUN curl -fsSL https://code-server.dev/install.sh | sh -s -- --method=standalone --prefix=/tmp/code-server

# code-server 확장 사전 설치 (coder 유저로 실행)
USER coder
RUN /tmp/code-server/bin/code-server --install-extension ms-python.python && \
    /tmp/code-server/bin/code-server --install-extension ms-toolsai.jupyter
USER root

# Coder agent 초기화 스크립트 복사
ENV REPO_NAME=langserver
COPY scripts/coder-init.sh /tmp/coder-init.sh
RUN chmod +x /tmp/coder-init.sh

USER coder
WORKDIR /workspaces

ENTRYPOINT ["/tmp/coder-init.sh"]
```

---

### `scripts/coder-init.sh`

```bash
#!/bin/bash
set -e

echo "=== Coder Agent Initialization ==="

# --- Section 1: Template copy ---
# 첫 실행 시 샘플 코드를 /workspaces로 복사
if [ ! -d "/workspaces/${REPO_NAME}" ]; then
    echo "Copying sample code to /workspaces/${REPO_NAME}..."
    cp -r /opt/workspace-template/${REPO_NAME} /workspaces/
    echo "Sample code copied successfully!"
fi

# --- Section 2: Virtualenv setup ---
setup_venv() {
  VENV_PATH="/workspaces/.venv/${REPO_NAME}"
  IMAGE_TAG_FILE="/opt/workspace-template/.image-tag"
  VENV_TAG_FILE="${VENV_PATH}/.image-tag"
  REQ_FILE="/opt/workspace-template/${REPO_NAME}/requirements.txt"
  
  echo "Setting up Python virtual environment..."
  
  # 3a. Check Python version compatibility
  if [ -d "$VENV_PATH" ]; then
    SYSTEM_PY_VERSION=$(python3 -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')
    if [ -f "$VENV_PATH/bin/python3" ]; then
      VENV_PY_VERSION=$("$VENV_PATH/bin/python3" -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")' 2>/dev/null || echo "")
      if [ -n "$VENV_PY_VERSION" ] && [ "$VENV_PY_VERSION" != "$SYSTEM_PY_VERSION" ]; then
        echo "Python version changed (venv: $VENV_PY_VERSION, system: $SYSTEM_PY_VERSION), recreating venv..."
        rm -rf "$VENV_PATH"
      fi
    fi
  fi
  
  # 3b. Create venv if it doesn't exist
  if [ ! -d "$VENV_PATH" ]; then
    echo "Creating new virtual environment at $VENV_PATH..."
    mkdir -p "$(dirname "$VENV_PATH")"
    
    VENV_TAR="/opt/workspace-template/venv.tar"
    if [ -f "$VENV_TAR" ]; then
      echo "Extracting pre-built venv from image..."
      tar xf "$VENV_TAR" -C /workspaces || return 1
    else
      echo "No pre-built venv found, building from scratch..."
      python3 -m venv "$VENV_PATH" || return 1
      "$VENV_PATH/bin/pip" install --no-cache-dir -r "$REQ_FILE" || return 1
    fi
    
    # Copy image tag to venv
    if [ -f "$IMAGE_TAG_FILE" ]; then
      cp "$IMAGE_TAG_FILE" "$VENV_TAG_FILE" 2>/dev/null || echo "unknown" > "$VENV_TAG_FILE"
    else
      echo "unknown" > "$VENV_TAG_FILE"
    fi
    echo "Virtual environment created successfully!"
  else
    # 3c. Sync venv if image tag changed
    CURRENT_TAG="unknown"
    NEW_TAG="unknown"
    
    if [ -f "$VENV_TAG_FILE" ]; then
      CURRENT_TAG=$(cat "$VENV_TAG_FILE")
    fi
    
    if [ -f "$IMAGE_TAG_FILE" ]; then
      NEW_TAG=$(cat "$IMAGE_TAG_FILE")
    fi
    
    if [ "$CURRENT_TAG" != "$NEW_TAG" ]; then
      echo "Image tag changed (venv: $CURRENT_TAG, image: $NEW_TAG), syncing venv..."
      VENV_TAR="/opt/workspace-template/venv.tar"
      if [ -f "$VENV_TAR" ]; then
        echo "Replacing venv from pre-built tar..."
        rm -rf "$VENV_PATH"
        tar xf "$VENV_TAR" -C /workspaces || return 1
      else
        "$VENV_PATH/bin/pip" install --no-cache-dir -r "$REQ_FILE" || return 1
      fi
      
      # Update tag file
      if [ -f "$IMAGE_TAG_FILE" ]; then
        cp "$IMAGE_TAG_FILE" "$VENV_TAG_FILE" 2>/dev/null || echo "unknown" > "$VENV_TAG_FILE"
      else
        echo "unknown" > "$VENV_TAG_FILE"
      fi
      echo "Virtual environment synced successfully!"
    else
      echo "Venv is up to date (tag: $CURRENT_TAG), skipping sync"
    fi
  fi
  
  # 3d. Activate venv for this process
  export VIRTUAL_ENV="$VENV_PATH"
  export PATH="$VENV_PATH/bin:$PATH"
  echo "Virtual environment activated for current process"
  
  # 3e. Setup pip cache on PVC
  export PIP_CACHE_DIR="/workspaces/.pip-cache"
  mkdir -p "$PIP_CACHE_DIR"
  
  # 3f. Setup .bashrc for new terminal sessions
  WORKSPACES_BASHRC="/workspaces/.bashrc"
  
  # Create /workspaces/.bashrc if not exists
  if [ ! -f "$WORKSPACES_BASHRC" ]; then
    echo "# Coder workspace configuration" > "$WORKSPACES_BASHRC"
  fi
  
  # Check if this repo's venv config already exists in /workspaces/.bashrc
  if ! grep -q "VIRTUAL_ENV=\"/workspaces/.venv/${REPO_NAME}\"" "$WORKSPACES_BASHRC" 2>/dev/null; then
    echo "" >> "$WORKSPACES_BASHRC"
    echo "# Auto-activate venv for ${REPO_NAME}" >> "$WORKSPACES_BASHRC"
    echo "export VIRTUAL_ENV=\"/workspaces/.venv/${REPO_NAME}\"" >> "$WORKSPACES_BASHRC"
    echo "export PATH=\"/workspaces/.venv/${REPO_NAME}/bin:\$PATH\"" >> "$WORKSPACES_BASHRC"
    echo "export PIP_CACHE_DIR=\"/workspaces/.pip-cache\"" >> "$WORKSPACES_BASHRC"
  fi
  
  # Ensure ~/.bashrc sources /workspaces/.bashrc (for coder user)
  USER_BASHRC="/home/coder/.bashrc"
  if [ -f "$USER_BASHRC" ]; then
    if ! grep -q "source /workspaces/.bashrc" "$USER_BASHRC" 2>/dev/null && ! grep -q ". /workspaces/.bashrc" "$USER_BASHRC" 2>/dev/null; then
      echo "" >> "$USER_BASHRC"
      echo "# Source workspace configuration" >> "$USER_BASHRC"
      echo "if [ -f /workspaces/.bashrc ]; then" >> "$USER_BASHRC"
      echo "    source /workspaces/.bashrc" >> "$USER_BASHRC"
      echo "fi" >> "$USER_BASHRC"
    fi
  fi
  
  echo "Virtual environment setup complete!"
  return 0
}

# Call setup_venv with error handling
setup_venv || echo "WARNING: venv setup failed, continuing without venv..."

# --- Section 3: Environment variable checks ---
# 환경 변수 확인
if [ -z "$CODER_AGENT_URL" ]; then
    echo "ERROR: CODER_AGENT_URL not set"
    echo "Keeping container alive for debugging..."
    sleep infinity
fi

if [ -z "$CODER_AGENT_TOKEN" ]; then
    echo "ERROR: CODER_AGENT_TOKEN not set"
    echo "Keeping container alive for debugging..."
    sleep infinity
fi

# --- Section 4: Coder agent download and exec ---
# Coder agent 다운로드
BINARY_DIR="/tmp/coder"
BINARY_NAME="coder"
mkdir -p "$BINARY_DIR"
cd "$BINARY_DIR"

BINARY_URL="${CODER_AGENT_URL}/bin/coder-linux-amd64"
echo "Downloading Coder agent from: $BINARY_URL"

# 재시도 로직
MAX_RETRIES=5
RETRY_COUNT=0
while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    if curl -fsSL "$BINARY_URL" -o "$BINARY_NAME"; then
        echo "Download successful!"
        break
    fi
    
    RETRY_COUNT=$((RETRY_COUNT + 1))
    if [ $RETRY_COUNT -lt $MAX_RETRIES ]; then
        echo "Download failed, retrying in 5 seconds... ($RETRY_COUNT/$MAX_RETRIES)"
        sleep 5
    else
        echo "ERROR: Failed to download after $MAX_RETRIES attempts"
        sleep infinity
    fi
done

chmod +x "$BINARY_NAME"

# Agent 실행
echo "Starting Coder agent..."
export CODER_AGENT_AUTH="token"
exec ./"$BINARY_NAME" agent
```

---

## 핵심 설계 패턴

### 1. 체인 생성 패턴

모든 서비스는 동일한 패턴을 따릅니다:

```
프롬프트 텍스트 (prompts.py) → PromptTemplate (templates.py) → Runnable chain (services/*.py) → add_routes (main.py)
```

체인 공식: `PROMPT | LLM | OutputParser`

### 2. 환경변수 로딩 순서

`main.py`와 `debug.py`에서 **모듈 import 전에** `load_dotenv()`를 호출합니다. 이는 `src/core/config.py`가 `os.getenv()`를 클래스 정의 시점에 호출하기 때문입니다.

```python
# 반드시 이 순서로:
load_dotenv(dotenv_path=dotenv_path)  # 1. 환경변수 로드
from src.core.config import settings   # 2. 그 후 설정 import
```

### 3. LangServe add_routes 규칙

- `add_routes()`는 반드시 **Runnable** 객체를 받아야 합니다 (일반 함수 불가)
- `path`는 `settings.API_V1_STR` 접두사를 포함합니다
- `enabled_endpoints`로 노출할 엔드포인트를 명시적으로 지정합니다

### 4. 모듈 레벨 체인 초기화

`chat_service.py`에서 체인을 **모듈 레벨**에서 생성하고 `try/except`로 감쌉니다. 실패 시 `None`으로 설정되며, `main.py`에서 `if chat_chain:` 으로 분기합니다.

---

## 새 체인 추가 순서

1. `src/prompts/prompts.py` → 프롬프트 텍스트 변수 추가
2. `src/prompts/templates.py` → `PromptTemplate.from_template()` 으로 템플릿 생성
3. `src/services/새서비스.py` → `get_llm()` + 프롬프트 + `StrOutputParser()` 조합으로 Runnable 반환
4. `main.py` → `add_routes(app, chain, path=f"{settings.API_V1_STR}/새경로")` 추가

---

## 검증 방법

```bash
# import 에러 확인 (배포 전 필수)
python -c "from main import create_app; app = create_app()"

# 의존성 설치
pip install -r requirements.txt

# 서버 실행
python main.py

# 디버그 모드 (콘솔 대화)
python debug.py
```

---

## 기술 스택 요약

| 구성 요소 | 라이브러리/버전 |
|----------|--------------|
| 웹 프레임워크 | FastAPI 0.109.0 |
| ASGI 서버 | uvicorn 0.27.0 |
| LLM 클라이언트 | langchain-openai >=0.2,<1.0 |
| 체인 프레임워크 | langchain >=0.3,<0.4 |
| API 자동 노출 | langserve 0.3.3 |
| 설정 관리 | pydantic-settings >=2.4.0,<3.0 |
| 환경변수 | python-dotenv 1.0.1 |
| Python 버전 | 3.12 (Docker 기준) |

---

## 주의사항

- 포트는 반드시 **8000** (인프라와 연결됨)
- CORS는 `allow_origins=["*"]` 전체 허용 (프론트엔드 연동)
- `__init__.py` 파일 없이 implicit namespace packages 사용
- `openai_api_key` 파라미터에 공백이 있음 (`openai_api_key =settings.LLM_API_KEY`) - 원본 그대로 유지
- `main.py`에서 `.env` 경로를 절대 경로로 계산하므로 어디서 실행해도 동작
