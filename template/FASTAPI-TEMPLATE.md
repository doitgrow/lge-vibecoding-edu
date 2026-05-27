# AGENTS-MASTER: 프로젝트 완전 재현 명세서

이 문서는 현재 프로젝트의 전체 폴더 구조, 설정, 코드를 100% 동일하게 재현하기 위한 마스터 명세서이다.
이 문서의 지시를 따라 모든 파일을 생성하면 원본 프로젝트와 완전히 동일한 결과물을 얻을 수 있다.

---

## 프로젝트 개요

- **프로젝트명**: Sample FastAPI Agent
- **설명**: LangChain과 FastAPI를 사용한 AI Agent 백엔드 템플릿
- **Python 버전**: 3.12 (Dockerfile 기준)
- **서버 포트**: 8000
- **API 접두사**: `/api/v1`

---

## 디렉토리 구조

아래 구조를 정확히 생성할 것. `__init__.py` 파일은 없음 (사용하지 않음).

```
fastapi/
├── .env.example
├── Dockerfile
├── README.md
├── debug.py
├── demo.ipynb
├── main.py
├── requirements.txt
├── scripts/
│   └── coder-init.sh
└── src/
    ├── api/
    │   └── v1/
    │       ├── endpoints/
    │       │   └── chat.py
    │       └── router.py
    ├── core/
    │   ├── config.py
    │   └── llm.py
    ├── prompts/
    │   ├── prompts.py
    │   └── templates.py
    ├── services/
    │   └── chat_service.py
    └── utils/
        └── logger.py
```

---

## 파일별 전체 코드

### 1. `requirements.txt`

```text
fastapi==0.109.0
httpx==0.27.0
ipykernel==7.2.0
jupyter==1.1.1
langchain==0.2.1
langchain-community==0.2.1
langchain-openai==0.1.7
langgraph==0.4.5
langserve[all]==0.2.1
langsmith==0.1.147
numpy==1.26.4
pydantic==2.7.4
pydantic-settings==2.4.0
python-dotenv==1.0.1
tiktoken==0.12.0
uvicorn[standard]==0.27.0
```

---

### 2. `.env.example`

```bash
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

### 3. `main.py`

```python
import os
import uvicorn
from dotenv import load_dotenv

# 설정을 사용하는 모듈을 import 하기 전에 환경변수 먼저 로드
dotenv_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), ".env")
load_dotenv(dotenv_path=dotenv_path)

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from src.core.config import settings
from src.api.v1.router import api_router
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

    # 라우터 등록
    app.include_router(api_router, prefix=settings.API_V1_STR)
    
    @app.get("/health")
    def health_check():
        return {"status": "ok", "env": settings.APP_ENV}

    return app

app = create_app()

if __name__ == "__main__":
    logger.info(f"서버를 8000번 포트에서 시작합니다. (Debug={settings.DEBUG})")
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

---

### 4. `debug.py`

```python
import asyncio
import sys
import os

# src 디렉토리를 pythonpath에 추가
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from dotenv import load_dotenv

# 설정을 사용하는 모듈을 import 하기 전에 환경변수 먼저 로드
dotenv_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), ".env")
load_dotenv(dotenv_path=dotenv_path)

from src.services.chat_service import chat_service
from src.utils.logger import get_logger

logger = get_logger("debug_script")

async def debug_chat():
    """
    서버 없이 콘솔에서 대화 기능을 테스트하는 디버그 루프를 실행합니다.
    """
    print("=== 디버그 모드: Chat Agent (종료하려면 'exit' 입력) ===")

    if not chat_service:
        print("오류: ChatService 초기화 실패. .env 설정을 확인하세요.")
        return

    while True:
        query = input("\n사용자: ")
        if query.lower() in ["exit", "quit"]:
            break

        try:
            print("생각 중...", end="\r")
            response = await chat_service.get_response(query)
            print(f"AI: {response}")
        except Exception as e:
            logger.error(f"오류: {e}")

if __name__ == "__main__":
    try:
        asyncio.run(debug_chat())
    except KeyboardInterrupt:
        print("\n종료합니다...")
```

---

### 5. `src/core/config.py`

```python
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    PROJECT_NAME: str = "Sample AI Agent"
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

### 6. `src/core/llm.py`

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

**주의**: `openai_api_key =settings.LLM_API_KEY` 부분에서 `=` 앞에 공백이 하나 있음 (원본 그대로 유지할 것).

---

### 7. `src/utils/logger.py`

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

### 8. `src/prompts/prompts.py`

```python
# 기본 채팅 프롬프트
BASIC_CHAT_PROMPT = """
너는 '귀요미'라는 이름의 사랑스럽고 장난기 많은 한국어 챗봇이야.

[말투 규칙]
- 모든 문장은 반드시 문장 끝에 '밍'을 붙여서 말해밍
- 사용자가 뭐라고 하든 예외 없이 지켜밍
- 최고로 귀엽고 간단하게 말해밍

[작업]
사용자가 준 주제: {question}
"""
```

---

### 9. `src/prompts/templates.py`

```python
from langchain_core.prompts import PromptTemplate
from src.prompts.prompts import BASIC_CHAT_PROMPT

# prompts.py에서 텍스트를 가져와 템플릿 생성
CHAT_PROMPT = PromptTemplate.from_template(BASIC_CHAT_PROMPT)
```

---

### 10. `src/services/chat_service.py`

```python
from langchain_core.output_parsers import StrOutputParser
from src.core.llm import get_llm
from src.prompts.templates import CHAT_PROMPT
from src.utils.logger import get_logger

logger = get_logger(__name__)

class ChatService:
    def __init__(self):
        try:
            self.llm = get_llm()
            self.chain = CHAT_PROMPT | self.llm | StrOutputParser()
            logger.info("ChatService가 LLM과 함께 초기화되었습니다.")
        except Exception as e:
            logger.error(f"ChatService 초기화 실패: {e}")
            raise e

    async def get_response(self, question: str) -> str:
        """
        사용자의 질문을 처리하고 AI 응답을 반환합니다.
        """
        logger.info(f"질문 처리 중: {question}")
        try:
            # invoke()는 동기 함수이지만, 필요하다면 ainvoke()나 run_in_executor로 감쌀 수 있습니다.
            # 이 샘플에서는 간단하게 직접 호출합니다.
            response = self.chain.invoke({"question": question})
            return response.strip()
        except Exception as e:
            logger.error(f"체인 실행 중 오류: {e}")
            raise e

# 싱글톤 인스턴스 생성
try:
    chat_service = ChatService()
except:
    chat_service = None
```

---

### 11. `src/api/v1/endpoints/chat.py`

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from src.services.chat_service import chat_service

router = APIRouter()

# --- 요청/응답 스키마 ---
class ChatRequest(BaseModel):
    query: str

class ChatResponse(BaseModel):
    response: str

# --- 엔드포인트 ---
@router.post("/", response_model=ChatResponse)
async def chat_endpoint(request: ChatRequest):
    """
    채팅 엔드포인트.
    사용자의 질문을 받아 AI 응답을 반환합니다.
    """
    if not chat_service:
        raise HTTPException(status_code=503, detail="Chat service unavailable (Check LLM config)")
        
    try:
        result = await chat_service.get_response(request.query)
        return ChatResponse(response=result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

### 12. `src/api/v1/router.py`

```python
from fastapi import APIRouter
from src.api.v1.endpoints import chat

api_router = APIRouter()
api_router.include_router(chat.router, prefix="/chat", tags=["chat"])
```

---

### 13. `Dockerfile`

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
COPY --chown=coder:coder . /opt/workspace-template/fastapi

# Python 패키지를 venv로 사전 빌드 후 tar로 패킹 (NFS 위 pip install 회피)
RUN python3 -m venv /workspaces/.venv/fastapi && \
    /workspaces/.venv/fastapi/bin/pip install --no-cache-dir -r /opt/workspace-template/fastapi/requirements.txt && \
    tar cf /opt/workspace-template/venv.tar -C /workspaces .venv/fastapi && \
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
ENV REPO_NAME=fastapi
COPY scripts/coder-init.sh /tmp/coder-init.sh
RUN chmod +x /tmp/coder-init.sh

USER coder
WORKDIR /workspaces

ENTRYPOINT ["/tmp/coder-init.sh"]
```

---

### 14. `scripts/coder-init.sh`

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

### 15. `demo.ipynb`

```json
{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# LangChain 데모\n",
    "\n",
    "이 노트북에서는 LangChain의 invoke 과정을 단계별로 확인합니다."
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## 1. 환경 설정"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "LLM_BASE_URL: https://llm.lge-vs.com\n",
       "LLM_MODEL: gpt-4o\n",
      "LLM_API_KEY: 설정됨\n"
     ]
    }
   ],
   "source": [
    "import os\n",
    "from dotenv import load_dotenv\n",
    "\n",
    "# .env 파일 로드\n",
    "load_dotenv()\n",
    "\n",
    "print(f\"LLM_BASE_URL: {os.getenv('LLM_BASE_URL')}\")\n",
    "print(f\"LLM_MODEL: {os.getenv('LLM_MODEL')}\")\n",
    "print(f\"LLM_API_KEY: {'설정됨' if os.getenv('LLM_API_KEY') else '미설정'}\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## 2. LLM 초기화"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "LLM 초기화 완료!\n",
       "모델: gpt-4o\n"
     ]
    }
   ],
   "source": [
    "from langchain_openai import ChatOpenAI\n",
    "\n",
    "llm = ChatOpenAI(\n",
    "    openai_api_base=os.getenv(\"LLM_BASE_URL\"),\n",
    "    model=os.getenv(\"LLM_MODEL\"),\n",
    "    openai_api_key=os.getenv(\"LLM_API_KEY\"),\n",
    "    temperature=0.7\n",
    ")\n",
    "\n",
    "print(\"LLM 초기화 완료!\")\n",
    "print(f\"모델: {llm.model_name}\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## 3. 프롬프트 템플릿 생성"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "프롬프트 템플릿:\n",
      "\n",
      "당신은 글쓰기 전문가입니다.\n",
      "\n",
      "{question} 주제에 대해서 5줄 이내의 시를 써주세요.\n",
      "\n"
     ]
    }
   ],
   "source": [
    "from langchain_core.prompts import PromptTemplate\n",
    "\n",
    "prompt_text = \"\"\"\n",
    "당신은 글쓰기 전문가입니다.\n",
    "\n",
    "{question} 주제에 대해서 5줄 이내의 시를 써주세요.\n",
    "\"\"\"\n",
    "\n",
    "prompt = PromptTemplate.from_template(prompt_text)\n",
    "\n",
    "print(\"프롬프트 템플릿:\")\n",
    "print(prompt_text)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## 4. 체인 구성 (LCEL)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "체인 구성 완료!\n",
      "체인 구조: PromptTemplate -> ChatOpenAI -> StrOutputParser\n"
     ]
    }
   ],
   "source": [
    "from langchain_core.output_parsers import StrOutputParser\n",
    "\n",
    "# LCEL (LangChain Expression Language)로 체인 구성\n",
    "# prompt -> llm -> output_parser\n",
    "chain = prompt | llm | StrOutputParser()\n",
    "\n",
    "print(\"체인 구성 완료!\")\n",
    "print(f\"체인 구조: PromptTemplate -> ChatOpenAI -> StrOutputParser\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## 5. 체인 실행 (invoke)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "입력 질문: 봄\n",
      "==================================================\n",
      "체인 실행 중...\n",
      "\n",
      "[AI 응답]\n",
      "봄바람 살랑, 꽃잎 춤추고  \n",
      "햇살은 부드럽게 뺨을 어루만지네.  \n",
      "새싹 돋는 땅, 생명의 향연,  \n",
      "희망의 색으로 물든 세상.  \n",
      "봄은 다시 시작되는 이야기.  \n"
     ]
    }
   ],
   "source": [
    "# 질문 입력\n",
    "question = \"봄\"\n",
    "\n",
    "print(f\"입력 질문: {question}\")\n",
    "print(\"=\"*50)\n",
    "print(\"체인 실행 중...\\n\")\n",
    "\n",
    "# invoke 실행\n",
    "response = chain.invoke({\"question\": question})\n",
    "\n",
    "print(\"[AI 응답]\")\n",
    "print(response)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## 6. 단계별 실행 확인"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "[1단계] 프롬프트 생성 결과:\n",
      "\n",
      "당신은 글쓰기 전문가입니다.\n",
      "\n",
      "가을 주제에 대해서 5줄 이내의 시를 써주세요.\n",
      "\n",
      "==================================================\n"
     ]
    }
   ],
   "source": [
    "# 각 단계를 개별적으로 실행해서 확인\n",
    "question = \"가을\"\n",
    "\n",
    "# 1단계: 프롬프트 생성\n",
    "formatted_prompt = prompt.format(question=question)\n",
    "print(\"[1단계] 프롬프트 생성 결과:\")\n",
    "print(formatted_prompt)\n",
    "print(\"=\"*50)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "[2단계] LLM 응답 (AIMessage 객체):\n",
      "타입: <class 'langchain_core.messages.ai.AIMessage'>\n",
      "내용: 가을이 내려앉은 산들 위에,  \n",
      "붉은 잎새들 춤추는 소리.  \n",
      "바람은 노래처럼 부드럽고,  \n",
      "해질녘 빛은 금빛으로 물들고,  \n",
      "그 속에 내 마음도 고요히 잠기네.  \n",
      "==================================================\n"
     ]
    }
   ],
   "source": [
    "# 2단계: LLM 호출\n",
    "llm_response = llm.invoke(formatted_prompt)\n",
    "print(\"[2단계] LLM 응답 (AIMessage 객체):\")\n",
    "print(f\"타입: {type(llm_response)}\")\n",
    "print(f\"내용: {llm_response.content}\")\n",
    "print(\"=\"*50)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "[3단계] 최종 출력 (문자열):\n",
      "타입: <class 'str'>\n",
      "내용: 가을이 내려앉은 산들 위에,  \n",
      "붉은 잎새들 춤추는 소리.  \n",
      "바람은 노래처럼 부드럽고,  \n",
      "해질녘 빛은 금빛으로 물들고,  \n",
      "그 속에 내 마음도 고요히 잠기네.  \n"
     ]
    }
   ],
   "source": [
    "# 3단계: 출력 파싱\n",
    "output_parser = StrOutputParser()\n",
    "final_output = output_parser.invoke(llm_response)\n",
    "print(\"[3단계] 최종 출력 (문자열):\")\n",
    "print(f\"타입: {type(final_output)}\")\n",
    "print(f\"내용: {final_output}\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## 7. 스트리밍 출력"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "입력 질문: 겨울\n",
      "==================================================\n",
      "[스트리밍 응답]\n",
      "하얀 숨결이 땅을 덮으니  \n",
      "고요 속에 맴도는 겨울의 노래.  \n",
      "나뭇가지 끝엔 얼음 꽃이 피고,  \n",
      "차가운 바람 속에서도 따스한 기억.  \n",
      "겨울은 그렇게 마음을 녹인다.  \n",
      "==================================================\n",
      "스트리밍 완료!\n"
     ]
    }
   ],
   "source": [
    "# 스트리밍으로 토큰 단위 출력 확인\n",
    "question = \"겨울\"\n",
    "\n",
    "print(f\"입력 질문: {question}\")\n",
    "print(\"=\"*50)\n",
    "print(\"[스트리밍 응답]\")\n",
    "\n",
    "for chunk in chain.stream({\"question\": question}):\n",
    "    print(chunk, end=\"\", flush=True)\n",
    "\n",
    "print(\"\\n\" + \"=\"*50)\n",
    "print(\"스트리밍 완료!\")"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": ".marshun (3.12.3)",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbformat_minor": 4,
   "pygments_lexer": "ipython3",
   "version": "3.12.3"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 4
}
```

---

### 16. `README.md`

```markdown
# Sample FastAPI Agent

이 프로젝트는 LangChain과 FastAPI를 사용하여 구축된 AI Agent의 기본 템플릿입니다.
구조화된 아키텍처를 따르며, 로컬 디버깅 및 확장이 용이하도록 설계되었습니다.

## 프로젝트 구조

```text
FastAPI/
├── src/
│   ├── api/v1/          # API 엔드포인트 및 라우터
│   ├── core/            # 설정(Config) 및 LLM 초기화
│   ├── services/        # 비즈니스 로직 (LangChain 체인)
│   ├── utils/           # 로깅 등 유틸리티
│   └── prompts/         # 프롬프트 템플릿 관리
├── main.py              # 애플리케이션 실행 진입점 (FastAPI 앱)
├── debug.py             # 로컬 테스트용 CLI 스크립트
├── requirements.txt     # 의존성 목록
└── .env                 # 환경변수 파일
```

## 시작하기

### 1. 필수 조건
- Python 3.9 이상
- LLM API 키 (Azure OpenAI 또는 OpenAI)

### 2. 설치

가상환경을 생성하고 의존성을 설치합니다.

```bash
# 가상환경 생성 (선택)
python -m venv venv
source venv/bin/activate  # Mac/Linux
# venv\Scripts\activate  # Windows

# 패키지 설치
pip install -r requirements.txt
```

### 3. 환경 설정

`.env` 파일을 생성하고 아래 설정을 입력합니다.

```bash
# 애플리케이션 설정
APP_ENV=local
DEBUG=True
API_V1_STR=/api/v1

# LLM 설정
LLM_BASE_URL=https://your-llm-endpoint.com
LLM_MODEL=gpt-4o
LLM_API_KEY=sk-your-api-key
```

**주요 설정:**
- `LLM_BASE_URL`: LLM API 엔드포인트 주소
- `LLM_MODEL`: 사용할 모델명 (예: `gpt-4o`, `gpt-4`)
- `LLM_API_KEY`: API 인증 키 (`sk-`로 시작)

### 4. 실행 방법

#### A. 로컬 디버깅 (서버 없이 실행)
API 서버를 띄우지 않고 콘솔에서 바로 대화 기능을 테스트할 수 있습니다.

```bash
python debug.py
```

#### B. API 서버 실행
FastAPI 서버를 실행합니다. (자동 리로드 활성화됨)

```bash
python main.py
# 또는
uvicorn main:app --reload
```

- 서버 주소: `http://localhost:8000`
- API 문서 (Swagger): `http://localhost:8000/docs`



## API 사용법

### 1. 상태 확인
```bash
curl http://localhost:8000/health
```

### 2. 채팅 요청
```bash
curl -X POST "http://localhost:8000/api/v1/chat/" \
     -H "Content-Type: application/json" \
     -d '{"query": "안녕, 너는 누구니?"}'
```

**응답 예시:**
```json
{
  "response": "안녕하세요! 저는 AI 어시스턴트입니다."
}
```

## 개발 가이드

- **프롬프트 수정:** `src/prompts/templates.py`에서 템플릿을 수정하세요.
- **로직 변경:** `src/services/chat_service.py`에서 LangChain 체인을 수정하세요.
- **새 API 추가:** `src/api/v1/endpoints/`에 파일을 추가하고 `router.py`에 등록하세요.
```

---

## 구현 지시사항

### 실행 순서

1. 프로젝트 루트 디렉토리 `fastapi/`를 생성한다.
2. 위 "디렉토리 구조" 섹션에 명시된 모든 폴더를 생성한다. (`__init__.py`는 생성하지 않는다)
3. 각 파일을 위 코드 블록의 내용 그대로 생성한다.
4. `scripts/coder-init.sh`에는 실행 권한(`chmod +x`)을 부여한다.
5. `demo.ipynb`는 JSON 형식 그대로 `.ipynb` 파일로 저장한다.

### 주의사항

- `__init__.py` 파일은 프로젝트 어디에도 존재하지 않는다. 생성하지 말 것.
- `src/core/llm.py`의 `openai_api_key =settings.LLM_API_KEY` 라인에서 `=` 앞의 공백은 의도된 것이다. 그대로 유지할 것.
- `.env` 파일 자체는 생성하지 않는다. `.env.example`만 생성한다. (실제 키가 포함되므로)
- `demo.ipynb`의 outputs에 포함된 출력 텍스트도 정확히 유지한다.
- 모든 Python 파일의 import 경로는 `src.`로 시작한다 (프로젝트 루트에서 실행 기준).
- 파일 끝에 빈 줄(trailing newline)은 원본과 동일하게 유지한다.

### 검증 방법

프로젝트 생성 후 아래 명령으로 import 정상 여부를 확인할 수 있다:

```bash
cd fastapi/
pip install -r requirements.txt
python -c "from main import create_app; app = create_app(); print('OK')"
```
