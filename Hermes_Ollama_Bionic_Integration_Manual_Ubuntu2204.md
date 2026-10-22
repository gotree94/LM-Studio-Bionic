# Hermes Agent 연동 매뉴얼: Ollama 및 Bionic
## Ubuntu 22.04 LTS · RTX 5090 Laptop GPU 24GB · RAM 64GB

- 작성 기준: 2026-10-22
- 대상 환경: Intel Core Ultra 9 275HX, 64GB DDR5, NVIDIA RTX 5090 Laptop GPU 24GB, Ubuntu 22.04 LTS
- 목적: Hermes Agent가 로컬 LLM을 사용하면서 프로젝트 파일·터미널·개발 도구를 활용하도록 구성하기

> **중요한 제품 구분**
>
> - Ollama + Hermes Agent: 공식 Ollama 통합 문서가 있어 구체적인 설치 절차를 안내할 수 있다.
> - Bionic + Hermes Agent: `Bionic`이라는 이름을 쓰는 프로젝트/제품이 여럿일 수 있다. 현재 대화에서는 정확한 Bionic 공식 URL 또는 GitHub 저장소가 특정되지 않았다. 따라서 아래 Bionic 장에서는 **OpenAI 호환 API가 제공되는 경우의 연동 절차**만 안내한다. 이는 Bionic 전용 설치법이 아니며, Bionic이 해당 API를 제공하지 않으면 그대로 적용할 수 없다.
> - 앞서 Bionic을 LM Studio의 일부인 것처럼 표현한 내용은 확정된 사실이 아니다. LM Studio와 Bionic은 동일한 제품이라고 가정하면 안 된다.

---

# Part A. Ollama + Hermes Agent

## A1. 구성과 역할

```text
사용자
  |
  v
Hermes Agent
  |-- 프로젝트 파일 읽기/쓰기 도구
  |-- 로컬 터미널 및 테스트 명령
  |-- 작업 계획과 결과 정리
  |
  v  OpenAI 호환 API: http://127.0.0.1:11434/v1
Ollama
  |
  v
로컬 LLM --> NVIDIA RTX 5090 Laptop GPU
```

Ollama는 모델을 실행하는 런타임이다. 임의의 파일을 알아서 읽고 수정하지 않는다. 실제 파일 접근은 Hermes의 도구/터미널 백엔드가 수행한다. 모델은 도구 호출을 잘 지원해야 에이전트 작업이 안정적이다.

## A2. 사전 점검

터미널에서 실행한다.

```bash
lsb_release -a
uname -m
free -h
df -h /
nvidia-smi
```

기대 결과:
- Ubuntu 22.04 LTS
- `x86_64`
- 약 64GB 시스템 RAM
- NVIDIA RTX 5090 Laptop GPU가 `nvidia-smi`에 표시됨

GPU 드라이버가 이미 정상 동작한다면 무조건 재설치하지 않는다. `nvidia-smi`가 실패하면 먼저 Ubuntu의 드라이버 상태와 Secure Boot/커널 호환성을 확인한다.

## A3. Ollama 설치

공식 다운로드 및 Linux 설치 안내를 사용한다.

- https://ollama.com/download
- https://github.com/ollama/ollama

공식 설치 페이지에서 제공하는 현재 Linux 명령을 확인한 뒤 설치한다. 설치 후:

```bash
ollama --version
ollama list
```

서비스 상태를 확인한다.

```bash
systemctl status ollama --no-pager
```

서비스가 실행 중이 아니라면 배포판 설치 방식에 따라 다음을 사용할 수 있다.

```bash
sudo systemctl enable --now ollama
```

일부 설치 방식에서는 Ollama가 systemd 서비스로 설치되지 않을 수 있다. 그 경우 공식 문서의 현재 설치 절차를 따른다.

API 응답 확인:

```bash
curl http://127.0.0.1:11434/api/tags
curl http://127.0.0.1:11434/v1/models
```

`/api/tags`는 Ollama API이고 `/v1/models`는 OpenAI 호환 API 경로다. 모델이 아직 없다면 목록이 비어 있을 수 있다.

## A4. 모델 다운로드와 테스트

먼저 https://ollama.com/search 에서 현재 모델 이름과 태그를 확인한다. 이름을 추측하지 말고 라이브러리에 실제로 존재하는 태그를 사용한다.

예시:

```bash
ollama pull gemma4
ollama run gemma4
```

이 예시의 모델 태그는 변경될 수 있으므로, 실행 시점의 라이브러리와 GPU 메모리 요구량을 확인한다. 24GB VRAM에서는 약 8B~14B급 Q4 모델부터 시작해 보고, 더 큰 양자화 모델은 메모리 사용량을 확인하며 시험한다.

다른 터미널에서:

```bash
ollama list
ollama ps
watch -n 1 nvidia-smi
```

`ollama ps`는 모델이 로드된 상태와 컨텍스트 정보를 확인하는 데 유용하다. 추론 중 VRAM 사용량을 관찰한다.

## A5. Hermes Agent 설치

공식 Hermes Agent 설치 문서:
- https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
- https://github.com/NousResearch/hermes-agent

공식 문서의 설치 명령은 다음과 같다.

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

원격 설치 스크립트를 실행하기 전에 공식 도메인과 문서를 확인한다. 설치 후 새 터미널을 열거나 설치 프로그램이 안내하는 대로 PATH를 갱신한다.

확인:

```bash
hermes --help
```

## A6. 가장 간단한 공식 연동: `ollama launch hermes`

현재 Ollama 공식 통합 문서에서는 다음 경로를 안내한다.

```bash
ollama launch hermes
```

문서: https://github.com/ollama/ollama/blob/main/docs/integrations/hermes.mdx

이 흐름은 Hermes가 설치되어 있지 않으면 설치를 안내하고, 모델 선택과 제공자 설정을 도와준다. 화면의 지시를 따라 로컬 모델을 선택한다. 메시징 서비스 연결은 선택 사항이며, 처음에는 건너뛰어도 된다.

설정 후 Hermes를 실행해 한국어 테스트 프롬프트를 입력한다.

```text
현재 선택된 모델과 실행 제공자가 로컬 Ollama인지 확인할 수 있는 정보를 알려줘.
아직 파일을 수정하지 말고, 도구를 사용해 작업할 때 어떤 권한이 필요한지 설명해 줘.
```

모델 선택/재설정:

```bash
hermes model
```

## A7. 수동 설정: Ollama OpenAI 호환 API 연결

자동 통합을 사용하지 않을 경우 Hermes 설정 마법사에서 다음을 선택한다.

1. `hermes`를 실행해 설정 마법사를 시작한다.
2. `More providers...` 또는 이에 해당하는 항목을 선택한다.
3. `Custom endpoint`를 선택한다.
4. API Base URL을 입력한다.

```text
http://127.0.0.1:11434/v1
```

5. API key는 로컬 Ollama에서 보통 필요하지 않으므로 비워 둔다.
6. 모델 목록에서 `ollama list`에 표시되는 정확한 모델 태그를 선택한다.
7. 컨텍스트 길이 항목이 있다면 자동 감지를 먼저 시도한다. 이후 A8 절차로 실제 Ollama 컨텍스트가 충분한지 확인한다.

Hermes 설정의 예시는 다음과 같은 구조다. 설치된 Hermes 버전에서 `~/.hermes/config.yaml` 형식과 키 이름을 확인한 뒤 사용한다.

```yaml
model:
  default: "gemma4"
  provider: "custom"
  base_url: "http://127.0.0.1:11434/v1"
```

이 YAML은 구성 예시이며, 모델 이름은 실제로 다운로드한 태그와 정확히 일치해야 한다. Hermes 버전에 따라 설정 마법사에서 저장하는 방식이 우선일 수 있다. 동일한 설정을 중복해서 추가하지 않는다.

## A8. 중요한 설정: 컨텍스트 길이

에이전트는 일반 채팅보다 시스템 프롬프트, 도구 정의, 대화 기록을 많이 사용한다. Ollama의 기본 컨텍스트가 너무 작으면 Hermes가 시작을 거부하거나 도구 작업이 불안정할 수 있다.

**현재 Hermes 문서의 요구 사항을 먼저 확인한다.** 공식 문서 버전에 따라 최소 권장 컨텍스트가 달라질 수 있다. 최근 문서에는 도구 사용을 위해 64K 컨텍스트를 요구하는 버전이 안내되어 있으므로, 설치된 Hermes 버전과 Ollama 통합 문서의 현재 기준을 확인한다.

Ollama의 서버 기본 컨텍스트를 설정하는 한 가지 방법은 systemd override다.

```bash
sudo systemctl edit ollama.service
```

편집기에서 다음을 추가한다. 이 예시는 64K 설정이다.

```ini
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=65536"
```

저장한 뒤:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

확인:

```bash
ollama ps
```

모델을 실제로 로드한 뒤 CONTEXT 열이 설정한 값에 맞는지 확인한다. 설치된 Ollama 버전에서 환경 변수 이름과 지원 여부를 공식 문서에서 확인한다.

주의:
- 64K 컨텍스트는 메모리 사용량을 크게 늘릴 수 있다.
- 24GB VRAM에서 대형 모델과 64K 컨텍스트를 동시에 사용하는 것은 모델에 따라 부담이 될 수 있다.
- 모델이 메모리 부족으로 실패하면 더 작은 모델 또는 낮은 컨텍스트를 시험한다. 단, Hermes가 요구하는 최소 컨텍스트보다 낮게 설정하면 시작 자체가 거부될 수 있다.
- OpenAI 호환 `/v1/chat/completions` 요청에서 컨텍스트 길이를 임의로 지정하는 것만으로 Ollama 서버의 컨텍스트가 늘어나지는 않는다. 서버 측 또는 Modelfile 설정을 확인한다.

## A9. 로컬 프로젝트 파일에 접근하도록 설정

테스트 전용 폴더를 만든다.

```bash
mkdir -p ~/AIProjects/hermes-lab
cd ~/AIProjects/hermes-lab
git init
printf '# Hermes Lab\n\nTemporary project for testing local agent access.\n' > README.md
```

Hermes의 터미널 백엔드를 로컬로 설정하려면 설치 버전의 공식 설정 문서를 확인한다.

- https://hermes-agent.nousresearch.com/docs/user-guide/configuration

예시 설정:

```yaml
terminal:
  backend: local
  cwd: "/home/YOUR_USER/AIProjects/hermes-lab"
  timeout: 180
```

`YOUR_USER`는 실제 Linux 사용자명으로 변경한다. `cwd`는 기본 작업 디렉터리이며, 보안 샌드박스가 아니다. 로컬 백엔드는 현재 사용자 권한으로 다른 파일에도 접근할 수 있다.

파일 접근 확인 프롬프트:

```text
현재 작업 디렉터리의 경로를 확인하고 파일 목록을 보여 줘.
README.md를 읽고 내용을 요약해 줘.
아직 파일을 수정하거나 삭제하지 마.
```

성공하면 다음 단계로 진행한다.

```text
README.md를 수정하기 전에 변경 계획을 보여 줘.
변경할 내용과 이유를 설명하고, 내가 승인하기 전에는 파일을 수정하지 마.
```

에이전트가 파일을 읽지 못하면 다음을 확인한다.
1. Hermes의 터미널 도구가 활성화되어 있는가?
2. `terminal.backend`가 의도한 로컬 백엔드인가?
3. `cwd` 경로가 실제로 존재하는가?
4. 현재 사용자에게 파일 읽기 권한이 있는가?
5. 모델이 도구 호출을 제대로 지원하는가?
6. 컨텍스트가 충분한가?

## A10. 보안 원칙

- 처음에는 별도 실험 폴더에서만 실행한다.
- 중요한 프로젝트는 작업 전 `git status`를 확인하고 커밋 또는 백업을 만든다.
- `sudo`, 파일 삭제, 시스템 설정 변경, 패키지 제거 명령은 별도로 검토한다.
- `.ssh`, API 키, 비밀번호, 브라우저 프로필 등은 에이전트 작업 범위에 포함하지 않는다.
- 로컬 추론을 사용하더라도 웹 검색이나 외부 API 도구가 데이터를 외부로 전송할 수 있다.
- 강한 격리가 필요하면 Docker 등 지원되는 격리 백엔드를 검토한다. 컨테이너에 마운트한 디렉터리는 컨테이너 안에서도 접근 가능하므로 마운트 범위를 최소화한다.

---

# Part B. Bionic + Hermes Agent

## B1. 먼저 확인해야 할 점

`Bionic`이라는 이름만으로는 정확한 제품을 특정할 수 없다. 현재 확인된 정보만으로는 사용자가 의미한 Bionic의 공식 저장소, 버전, API 지원 여부가 확정되지 않았다.

따라서 **Bionic 전용 설치 명령이나 설정 파일을 추측해서 제시하지 않는다.** 다음의 일반 연동 절차는 Bionic이 로컬에서 OpenAI 호환 API 서버를 제공하는 경우에만 적용된다.

Bionic의 공식 URL 또는 GitHub 저장소를 확보하면 다음을 확인해야 한다.
- Ubuntu 22.04 지원 여부
- Linux x86-64 패키지 제공 여부
- 로컬 모델을 자체 실행하는지, 외부 추론 서버를 사용하는지
- OpenAI 호환 `/v1` API가 있는지
- API Base URL과 모델 ID
- 인증 키가 필요한지
- 도구 실행과 파일 접근 기능이 있는지
- Hermes를 Bionic 내부에 설치하는 구조인지, Hermes가 Bionic의 모델 API를 호출하는 구조인지

## B2. 가능한 두 가지 구조

### 구조 1: Bionic이 모델 서버인 경우

```text
Hermes Agent
   |
   | OpenAI 호환 API
   v
Bionic의 로컬 모델 서버
   |
   v
로컬 LLM / GPU
```

이 구조는 Bionic이 API 서버를 실제로 제공하고, Hermes가 해당 API와 호환될 때만 가능하다.

### 구조 2: Bionic이 에이전트 앱인 경우

```text
사용자
  |
  v
Bionic Agent
  |
  +--> 로컬 모델 서버(Ollama 등)
  |
  +--> 파일/터미널 도구
```

이 경우 Hermes를 Bionic 안에 “추가 설치”할 수 있는지는 Bionic의 플러그인/외부 에이전트 지원 방식에 따라 달라진다. 단순히 모델 API를 연결하는 것과 Hermes를 내부 에이전트로 임베드하는 것은 다른 작업이다.

## B3. Bionic이 OpenAI 호환 API를 제공하는 경우

Bionic의 공식 UI나 문서에서 API 서버를 활성화하고 다음 값을 확인한다.

- Base URL: 예를 들어 `http://127.0.0.1:PORT/v1`
- Model ID: API가 보고하는 실제 모델 ID
- API Key: 해당 서버가 요구하는 경우에만 사용
- Context: Bionic/모델 서버에서 실제로 설정한 컨텍스트 길이

`PORT`를 임의로 추정하지 말고 Bionic이 실제로 표시하는 포트를 사용한다.

API 연결 테스트:

```bash
curl http://127.0.0.1:PORT/v1/models
```

`PORT`를 실제 포트 번호로 바꾼다. API가 인증을 요구한다면 해당 제품의 공식 방식으로 인증 헤더를 추가한다. 인증 없이 연결이 안 된다고 해서 API Key에 임의 값을 넣지 않는다.

Hermes 설정 마법사에서:
1. `hermes model`을 실행한다.
2. `Custom endpoint`를 선택한다.
3. Bionic이 제공한 Base URL을 입력한다.
4. 해당 서버가 요구하는 경우에만 API 키를 입력한다.
5. `/v1/models`에서 확인한 모델 ID를 선택한다.
6. 짧은 대화 테스트를 수행한다.

설정 예시:

```yaml
model:
  default: "ACTUAL_MODEL_ID"
  provider: "custom"
  base_url: "http://127.0.0.1:PORT/v1"
```

위 값은 템플릿이다. `ACTUAL_MODEL_ID`와 `PORT`를 실제 Bionic API의 값으로 바꿔야 한다. 해당 Bionic이 OpenAI 호환 API를 지원하지 않으면 이 방법은 동작하지 않는다.

## B4. 연결 테스트 후 확인할 사항

1. Hermes가 Bionic API의 모델 목록을 조회할 수 있는가?
2. 단순 텍스트 대화가 되는가?
3. 도구 호출(tool calling)을 지원하는가?
4. Hermes의 시스템 프롬프트와 도구 정의를 수용할 만큼 컨텍스트가 충분한가?
5. 모델의 도구 호출 결과가 올바른 형식으로 반환되는가?
6. 파일/터미널 작업은 Hermes의 로컬 도구가 수행하는가, 아니면 Bionic의 별도 샌드박스가 수행하는가?
7. 모델 추론과 별도로 Bionic 또는 Hermes가 외부 네트워크로 데이터를 보내는가?

텍스트 채팅만 되는 API라고 해서 Hermes의 도구 기반 작업이 정상 작동한다고 단정할 수 없다. 도구 호출 호환성은 별도 테스트가 필요하다.

---

# Part C. 어떤 구성을 선택할까?

| 상황 | 추천 |
|---|---|
| 오늘 바로 로컬 에이전트를 실행하고 싶음 | Ollama + `ollama launch hermes` |
| 모델 관리 GUI가 중요함 | LM Studio + Hermes |
| Bionic이 OpenAI 호환 로컬 API를 제공함 | Hermes의 Custom endpoint로 연결 가능성 검증 |
| Bionic이 이미 파일 작업을 하는 에이전트임 | Hermes를 중복 설치하기 전에 외부 모델/API/플러그인 지원 확인 |
| 코드베이스 개발과 테스트가 주목적임 | Hermes와 OpenHands를 같은 테스트 프로젝트로 비교 |

## 최종 권장

우선 공식 통합 경로인 `ollama launch hermes`를 사용해 **Ollama + Hermes를 별도 실험 폴더에서 검증**한다. 이 경로는 모델 실행과 파일 작업 도구를 구분해 문제를 진단하기 쉽다.

Bionic은 정확한 제품을 식별한 다음 연결 방법을 결정한다. 사용자가 말한 Bionic의 공식 홈페이지 또는 GitHub URL을 확인하기 전에는 LM Studio나 다른 앱으로 간주하지 않는다.

---

# 공식 참고 링크

- Ollama 공식 다운로드: https://ollama.com/download
- Ollama 모델 라이브러리: https://ollama.com/search
- Ollama + Hermes 공식 통합: https://github.com/ollama/ollama/blob/main/docs/integrations/hermes.mdx
- Hermes Agent 공식 저장소: https://github.com/NousResearch/hermes-agent
- Hermes 설치 가이드: https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
- Hermes 모델 제공자 안내: https://hermes-agent.nousresearch.com/docs/integrations/providers/
- Hermes 설정 안내: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

> 제품과 명령은 업데이트될 수 있다. 설치 시점에는 공식 문서의 최신 내용을 확인하고, 특히 컨텍스트 요구량과 모델 태그는 설치한 버전의 기준을 따른다.
