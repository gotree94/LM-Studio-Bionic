# 로컬 AI 에이전트 환경 추천 및 구성 가이드
## Ubuntu 22.04 LTS · Intel Core Ultra 9 275HX · RTX 5090 Laptop 24GB · RAM 64GB

작성 기준: 2026-10-22

## 1. 결론

이 노트북에서 로컬 모델을 사용하면서 실제 파일을 읽고, 코드를 수정하고, 빌드와 테스트까지 수행하려면 **모델 실행 엔진과 에이전트를 분리**하는 구성을 추천한다.

### 1순위: Ollama + Hermes Agent

- Ollama: 모델 다운로드와 로컬 추론 API 제공
- Hermes Agent: 계획 수립, 터미널 명령, 파일 작업, 도구 호출 및 지속적인 작업 흐름
- 로컬 파일 시스템: 실제 개발 프로젝트와 Git 저장소
- 보안: 처음에는 지정한 프로젝트 디렉터리만 대상으로 작업하고, 중요한 변경은 Git으로 백업한다.

### GUI 선호: LM Studio + Hermes Agent

모델 탐색과 로딩 상태를 GUI에서 관리하고, LM Studio의 로컬 API 서버를 Hermes에 연결한다. LM Studio는 모델 실행 도구이지 그 자체로 모든 로컬 파일을 자동으로 조작하는 에이전트는 아니다.

### 코드 작업 중심: LM Studio 또는 Ollama + OpenHands

OpenHands는 소프트웨어 개발 에이전트 대안이다. 로컬 모델 서버와 연결할 수 있지만, 프로젝트 파일 접근은 OpenHands의 백엔드/워크스페이스 설정에 달려 있다.

---

## 2. 하드웨어에 맞는 모델 전략

| 항목 | 사용자 사양 | 권장 방향 |
|---|---|---|
| CPU | Intel Core Ultra 9 275HX, 24코어 / 24스레드 | 전처리와 CPU 작업 |
| RAM | 64GB DDR5-5600 | 시스템 작업과 일부 CPU 오프로딩 |
| GPU | RTX 5090 Laptop GPU, 24GB GDDR7 | 주 추론 장치 |
| OS | Ubuntu 22.04 LTS | Linux x86-64 앱과 호환 NVIDIA 드라이버 |

24GB VRAM은 양자화된 중대형 모델을 시험하기에 좋은 용량이다. 단, 모델 가중치 외에 컨텍스트의 KV 캐시, 런타임 버퍼, GPU 데스크톱 사용량도 VRAM을 차지한다.

시작 권장:
- 빠른 시작: 약 8B~14B급 모델의 Q4 양자화
- 코드 에이전트: 도구 호출을 잘 수행하는 14B급 모델부터 검증
- 더 높은 품질: 지원되는 27B~35B급 MoE 모델의 Q4 변형을 시도하되 실제 메모리 사용량 확인
- 컨텍스트: 처음에는 8K 정도로 시작하고, 안정성을 확인한 뒤 증가

Ollama의 공식 Hermes 통합 문서에서는 로컬 모델 후보로 Gemma 4와 Qwen3.6 계열을 안내하고 있으며, Qwen3.6은 약 24GB VRAM 환경에서 양자화 버전을 시도할 수 있는 후보로 소개한다. 모델의 실제 태그와 VRAM 요구량은 다운로드 전에 확인한다.

---

## 3. 도구별 비교

| 도구 | 주 역할 | 로컬 파일 접근 | 추천 용도 |
|---|---|---|---|
| Ollama | 로컬 모델 런타임 | 자체적으로 임의 파일을 읽거나 편집하지 않음 | 간단한 모델 실행/API |
| LM Studio | GUI 모델 관리자 및 로컬 API 서버 | 자체 모델 실행과 별도로 에이전트/도구가 필요 | GUI 중심 모델 관리 |
| Hermes Agent | 범용 에이전트, 도구 및 터미널 실행 | 설정한 terminal backend와 사용자 권한에 따름 | 범용 자동화 및 개발 |
| OpenHands | 소프트웨어 개발 에이전트 | 워크스페이스/백엔드 설정에 따름 | 코드베이스 분석, 구현, 테스트 |
| Open WebUI | 웹 채팅 UI 및 통합 | 기능과 도구 설정에 따름 | 웹 기반 채팅 및 문서 질의 |

중요한 개념: **모델 추론과 파일 조작은 서로 다른 기능이다.** 모델은 답변을 생성하고, 에이전트의 도구가 실제 파일 읽기·쓰기와 셸 명령을 수행한다.

---

## 4. 왜 Ollama만으로 로컬 파일을 조작할 수 없는가?

Ollama는 모델을 실행하고 API를 제공한다. 모델에게 `/home/user/project/main.c`를 분석하라고 요청해도, 파일 내용이 프롬프트에 포함되거나 파일을 읽는 도구가 호출되지 않으면 모델은 실제 파일을 볼 수 없다.

권장 구조:

```text
사용자
  |
  v
Hermes Agent (작업 계획 / 도구 호출)
  |                         \
  |                          \--> 로컬 파일 / 터미널 / Git / 테스트
  v
Ollama 또는 LM Studio API
  |
  v
로컬 LLM (RTX 5090 Laptop GPU)
```

---

## 5. Hermes + Ollama 시작하기

### 5.1 시스템 확인

```bash
lsb_release -a
uname -m
free -h
nvidia-smi
df -h /
```

`nvidia-smi`가 GPU와 드라이버를 정상적으로 표시해야 한다. 정상 드라이버가 이미 있다면 무조건 재설치하지 않는다.

### 5.2 Ollama 설치

공식 설치 안내: https://ollama.com/download

공식 Linux 안내에 따라 설치한 후 확인한다.

```bash
ollama --version
```

### 5.3 모델 다운로드

Ollama 모델 라이브러리에서 실제로 제공되는 이름과 태그를 확인한다.

https://ollama.com/search

예를 들어 다음과 같은 명령 형태로 모델을 다운로드한다. **모델 태그는 예시이므로 현재 라이브러리에서 존재 여부와 VRAM 요구량을 확인한 뒤 사용한다.**

```bash
ollama pull gemma4
```

실행 테스트:

```bash
ollama run gemma4
```

다른 터미널에서:

```bash
ollama list
nvidia-smi
```

### 5.4 Hermes 설치

공식 설치 안내:
https://hermes-agent.nousresearch.com/docs/getting-started/quickstart

공식 Linux 설치 절차를 확인하고 진행한다. 공식 문서에 안내된 설치 명령은 다음과 같다.

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

설치 스크립트는 원격 코드를 실행하므로, 실행 전에 공식 도메인과 문서를 확인하는 것이 좋다. 설치 후 터미널을 다시 열거나 공식 문서의 안내대로 셸 환경을 갱신한다.

Ollama의 공식 Hermes 통합은 다음 명령을 안내한다.

```bash
ollama launch hermes
```

이 명령은 설치/설정 과정을 도와주는 통합 경로다. 버전에 따라 대화형 안내가 달라질 수 있으므로 화면의 지시를 따른다.

공식 참고:
- Ollama의 Hermes 통합: https://github.com/ollama/ollama/blob/main/docs/integrations/hermes.mdx
- Hermes 설정: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

### 5.5 로컬 파일을 대상으로 할 프로젝트 폴더 만들기

```bash
mkdir -p ~/AIProjects/agent-lab
cd ~/AIProjects/agent-lab
git init
```

먼저 테스트용 저장소에서 시작하고, 실제 프로젝트 전체를 대상으로 하기 전에 파일 접근 범위와 권한을 검증한다.

### 5.6 Hermes 터미널 백엔드 설정

Hermes의 설정에서 다음과 같은 항목을 설정할 수 있다. 실제 설치 버전의 설정 파일과 문서를 먼저 확인한다.

```yaml
terminal:
  backend: local
  cwd: "/home/your_user/AIProjects/agent-lab"
  timeout: 180
```

- `backend: local`: 명령을 로컬 컴퓨터에서 실행한다.
- `cwd`: 상대 경로 명령의 기본 디렉터리를 지정한다.
- `timeout`: 명령 시간 제한이다.

`cwd`는 기본 작업 위치일 뿐 보안 경계가 아니다. 로컬 백엔드는 사용자 계정의 권한을 그대로 사용할 수 있다. 실험 단계에서는 전용 프로젝트 디렉터리를 쓰고, 시스템 파일과 개인 데이터가 있는 홈 디렉터리 전체를 무분별하게 대상으로 지정하지 않는다.

---

## 6. LM Studio + Hermes 대안

GUI로 모델을 관리하고 싶다면 LM Studio가 편리할 수 있다.

1. 공식 페이지에서 Linux 버전을 설치한다: https://lmstudio.ai/
2. 모델 탐색기에서 호환 모델을 다운로드한다.
3. 모델을 로드하고 로컬 API 서버를 시작한다.
4. 서버 주소와 포트를 LM Studio 설정에서 확인한다.
5. Hermes의 모델 설정에서 해당 로컬 OpenAI 호환 엔드포인트를 지정한다.
6. 모델 이름과 컨텍스트 길이가 실제 서버 설정과 일치하는지 확인한다.

LM Studio의 API 서버 문서:
https://lmstudio.ai/docs/developer/core/server/settings

보안 주의: API 서버를 로컬 네트워크 전체에 노출할 필요가 없다면 `127.0.0.1`에만 바인딩한다. MCP 서버가 로컬 파일이나 네트워크에 접근할 수 있으므로 신뢰할 수 있는 서버만 활성화한다.

---

## 7. 에이전트 파일 작업을 안전하게 시작하는 방법

### 단계 1: 읽기 전용 분석

```text
현재 프로젝트의 파일 목록과 디렉터리 구조를 살펴보고 요약해 줘.
아직 파일을 수정하거나 명령을 실행하지 말고,
개선 계획만 제시해 줘.
```

### 단계 2: 제한된 파일 작업

```text
현재 프로젝트의 README.md를 읽고,
실제 파일 구조에 맞는 개선안을 제시해 줘.
수정하기 전에 변경할 내용을 요약하고 확인을 요청해 줘.
```

### 단계 3: 테스트 실행

```text
프로젝트의 빌드 및 테스트 명령을 확인해 줘.
실행 전 명령과 영향을 설명하고,
승인된 명령만 실행한 뒤 실제 결과를 기록해 줘.
테스트하지 않은 항목은 성공했다고 쓰지 마.
```

### 안전 원칙

- `sudo`, 시스템 설정 변경, 패키지 제거, 대량 삭제는 별도 확인한다.
- `.ssh`, API 키, 토큰, 비밀번호 파일은 프로젝트 작업 대상에서 제외한다.
- 작업 전 Git 상태를 확인하고 커밋 또는 백업을 만든다.
- 출처가 불분명한 MCP 서버나 플러그인을 무조건 설치하지 않는다.
- 로컬 모델을 사용해도 웹 검색, 외부 API, 메신저 도구가 외부로 데이터를 보낼 수 있다.
- 강한 격리가 필요하면 Docker 백엔드 또는 별도 작업 디렉터리를 고려한다.

---

## 8. 성능 및 문제 해결

### GPU 확인

```bash
watch -n 1 nvidia-smi
```

모델 응답 생성 중 VRAM 사용량과 GPU 사용률을 확인한다. GPU 사용률은 작업 단계에 따라 변하므로 항상 100%일 필요는 없다.

### 메모리 부족

```bash
free -h
nvidia-smi
df -h
```

대응 순서:
1. 컨텍스트를 8K로 낮춘다.
2. 더 작은 모델 또는 Q4 양자화를 선택한다.
3. 다른 GPU 사용 앱을 닫는다.
4. 모델의 예상 메모리 요구량을 확인한다.

### 에이전트가 파일을 읽지 못함

1. 에이전트가 실제로 로컬 터미널 도구를 사용할 수 있는지 확인한다.
2. `terminal.backend`가 의도한 값인지 확인한다.
3. `cwd`가 존재하는 경로인지 확인한다.
4. 실행 사용자에게 파일 읽기 권한이 있는지 확인한다.
5. Docker 등 격리 백엔드를 쓴다면 해당 프로젝트 디렉터리가 컨테이너에 마운트되어 있는지 확인한다.
6. 모델이 도구 호출을 제대로 수행하는지 테스트한다.

### 에이전트가 채팅만 하고 작업하지 않음

원인은 파일 권한만이 아니라 모델의 도구 호출 능력, 컨텍스트 부족, API 호환성 또는 도구 설정일 수 있다. 짧은 작업으로 확인하고, 필요하면 에이전트용으로 검증된 모델로 바꿔 본다.

---

## 9. 최종 추천

### 가장 먼저 설치할 조합

**Ollama + Hermes Agent**

이 조합으로 로컬 모델 추론과 실제 파일/터미널 작업을 분리해서 구성한다. 우선 작은 테스트 저장소에서 파일 읽기 → 계획 작성 → 승인된 파일 수정 → 테스트 실행의 순서를 검증한다.

### GUI를 선호한다면

**LM Studio + Hermes Agent**

모델 관리와 메모리 확인을 GUI로 하고, 파일 작업은 Hermes가 담당하도록 한다.

### 주 용도가 코드베이스 개발이라면

**LM Studio 또는 Ollama + OpenHands**도 비교 테스트한다. 같은 모델로 작은 Python 또는 C 프로젝트를 분석하고, 테스트를 실행하고, 결과를 수정하는 작업을 수행해 정확도와 안정성을 비교한다.

---

## 10. 공식 참고 링크

- Ollama 다운로드: https://ollama.com/download
- Ollama 모델 라이브러리: https://ollama.com/search
- Ollama의 Hermes 통합: https://github.com/ollama/ollama/blob/main/docs/integrations/hermes.mdx
- Hermes 설치/빠른 시작: https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
- Hermes 설정 및 터미널 백엔드: https://hermes-agent.nousresearch.com/docs/user-guide/configuration
- LM Studio: https://lmstudio.ai/
- LM Studio API 서버 설정: https://lmstudio.ai/docs/developer/core/server/settings
- OpenHands 로컬 모델 연결: https://github.com/OpenHands/docs/blob/main/openhands/usage/llms/local-llms.mdx
- Open WebUI와 Ollama: https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-ollama/
