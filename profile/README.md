# team-memory-system

에이전트와 하는 대화를 각자의 기억 서버에 모으고, 팀원끼리는 "물어보는 창구" 하나만
열어 두는 자기호스팅 기억 시스템입니다.

## 어떤 구조인가

```
   내 컴퓨터들                        내 거점 컴퓨터
 ┌──────────────┐
 │ Codex        │  훅 ─┐
 │ Claude Code  │      ├──▶  수집기  ──▶  Honcho  ──▶  PostgreSQL
 │ agy          │      │              (기억 서버)      (기억 본체)
 └──────────────┘      │                  │
 ┌──────────────┐      │                  ├─ MCP 브리지 (내 것)      ◀── 내 에이전트
 │ ChatGPT 웹   │ 내보내기               └─ MCP 브리지 (공유)       ◀── 팀원
 └──────────────┘                              도구: chat 하나
```

**사람마다 기억 서버가 하나, 데이터베이스가 하나입니다.** 한 사람이 컴퓨터를 여러 대
쓰더라도 그 대화는 모두 자기 서버 한 곳으로 모입니다.

**팀원끼리 데이터베이스를 공유하지 않습니다.** 공유되는 것은 MCP 도구 `chat` 하나뿐입니다.
팀원은 질문을 하고 답을 받습니다. 원문 메시지를 읽지는 않습니다.

## 저장소 세 개

| | 무엇인가 | 어디에 설치하나 |
|---|---|---|
| [honcho-agent-bridge](https://github.com/team-memory-system/honcho-agent-bridge) | 수집기·설치기·에이전트 플러그인 (MIT) | 에이전트를 돌리는 기계마다 |
| [honcho-selfhost](https://github.com/team-memory-system/honcho-selfhost) | 기억 서버. `plastic-labs/honcho` 포크 (AGPL-3.0) | 사람마다 컴퓨터 한 대 |
| [llm-proxy](https://github.com/team-memory-system/llm-proxy) | 구독 계정을 API로 바꾸는 어댑터와 라우터 (AGPL-3.0) | 거점 컴퓨터만 |

직접 내려받을 저장소는 **`honcho-agent-bridge` 하나**입니다. 나머지 둘은 그것이 필요할 때
가져다 씁니다.

---

# 설치

## 경우 A — 남의 기억에 물어보기만 할 때

가장 많은 경우입니다. 내 대화를 모으지 않고, 거점의 `chat` 창구에만 붙습니다.

1. **플러그인을 설치합니다.** 아래 경우 B의 1번과 같습니다.
2. **Claude Code에서 `/memory-setup`, Codex에서 `$setup-memory`를 실행하고 "물어보기만"을
   고릅니다.** 이 컴퓨터에서만 열리는 설치 화면이 브라우저에 뜹니다.
3. **2번 칸 "다른 사람의 기억에 연결"에 기억 주인에게 받은 네 값을 넣고 연결을 누릅니다.**
   브리지 주소, 브리지 토큰, Cloudflare 서비스 토큰 ID와 비밀입니다. 화면이 그 값으로
   브리지에 실제로 닿은 뒤에만 저장하고, 쓸 수 있는 도구(`chat`)를 보여줍니다.
4. **에이전트를 다시 시작합니다.** Claude Code는 `/reload-plugins`, Codex는 새 세션입니다.

네 값은 채팅에 붙여넣지 마세요. 에이전트도 묻지 않게 되어 있습니다. 터미널에서 하려면
`bridge connect --url <주소>`에 나머지 셋을 환경변수(`HONCHO_MCP_BEARER_TOKEN`,
`CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`)로 줍니다. 명령행 인자로 주면 거부합니다.

이 값이 있으면 플러그인의 MCP 서버가 도구를 직접 구현하지 않고 거점의 브리지로 넘깁니다.
그래서 도구는 `chat` 하나만 보이고, 호출은 거점의 조회 기록에 남습니다.

## 경우 B — 내 기억도 만들 때

내 대화를 내 서버에 모읍니다. 플러그인 하나 깔고 설치 명령 한 번입니다.

### 1. 플러그인 설치

**Claude Code**
```sh
/plugin marketplace add team-memory-system/honcho-agent-bridge
/plugin install honcho-agent-bridge@honcho-agent-bridge
```

**Codex**
```sh
codex plugin marketplace add team-memory-system/honcho-agent-bridge
codex plugin add honcho-agent-bridge@honcho-agent-bridge
```

설치한 뒤 Claude Code는 플러그인을 다시 읽고, Codex는 새 세션을 엽니다.

### 2. 설치 명령

Claude Code에서 `/memory-setup`, Codex에서 `$setup-memory`.

에이전트가 순서대로 물어봅니다. 하는 일은 이렇습니다.

1. **지금 상태 확인** — 어떤 에이전트가 깔려 있는지, 기억 서버가 이미 도는지
2. **서버를 어떻게 할지** — 이미 도는 서버가 있으면 그걸 씁니다. 없으면 이 컴퓨터에
   Docker로 올릴지 물어봅니다
3. **어느 에이전트에 붙일지** — 감지된 것 전부가 기본값입니다
4. **기억 저장 위치** — 운영체제 기본 위치를 권합니다
5. **내 이름 (peer ID)** — 기억 안에서 나를 가리키는 이름입니다
6. **계획을 먼저 보여주고** 확인을 받은 뒤에 바꿉니다. 고치는 파일은 미리 백업합니다

끝나면 대화가 끝날 때마다 자동으로 모입니다. 따로 실행할 것이 없습니다.

### 화면으로 하고 싶으면

터미널 대신 브라우저에서 같은 일을 할 수 있습니다. `/memory-setup`에서 화면을 열어 달라고
하거나, 저장소에서 직접 엽니다.

```sh
node scripts/cli.mjs ui open
```

상태 확인, 다른 사람의 기억에 연결, 훅 설치, 서버·프록시 켜기, ChatGPT 내보내기 파일
올리기까지 한 화면입니다. 이 컴퓨터에서만 열립니다.

## 경우 C — 거점을 직접 운영할 때

기억 서버와 LLM 프록시를 자기 컴퓨터에서 돌립니다.

**필요한 것**

| | |
|---|---|
| 운영체제 | macOS 또는 윈도우. 네이티브 리눅스는 `portable` 프로필만 |
| Docker | Desktop 또는 Engine + Compose. 윈도우는 WSL 2 백엔드 |
| Node.js | 18 이상 |
| Ollama | 임베딩용. `PATH`에 있어야 합니다 |
| Codex 로그인 | `codex login`. 자격증명은 복사되지 않고 그 자리에서 읽습니다 |

**순서**

```sh
# 1. 기억 서버 준비와 기동
node scripts/cli.mjs server plan    --profile personal   # 무엇이 바뀌는지 먼저 봅니다
node scripts/cli.mjs server prepare --profile personal
node scripts/cli.mjs server start   --profile personal
node scripts/cli.mjs server verify  --profile personal

# 2. LLM 프록시 (구독 계정을 API로)
node scripts/cli.mjs host prepare   --profile personal
node scripts/cli.mjs host start     --profile personal
node scripts/cli.mjs host status    --profile personal
```

**프록시는 재부팅하면 다시 올려야 합니다.** launchd·작업 스케줄러·systemd에 아무것도
등록하지 않습니다. `host start`가 감시 프로세스를 분리 실행하고, 그 프로세스는 터미널을
닫아도 살아 있지만 재부팅은 넘기지 못합니다. 그동안 기억 저장은 계속되고 파생 처리만
멈춥니다.

기억 서버 자체는 Docker의 `restart: unless-stopped`로 다시 올라옵니다.

**팀원에게 창구를 열려면** Cloudflare 터널과 Access 정책이 필요합니다. 공유용 브리지는
도구가 `chat` 하나로 제한되고, 요청 헤더로도 도구 인자로도 다른 워크스페이스나 피어를
가리킬 수 없게 고정됩니다. 토큰이 틀리면 연결 단계에서 거부하고, 모든 호출을 질의 원문과
함께 기록합니다.

---

## 문제가 생기면

Claude Code에서 `/memory-doctor`, Codex에서 `$setup-memory`로 진단을 요청하세요. 설정
파일, 수집기 런타임, 서버 연결, 훅 설치 상태를 하나씩 확인해 무엇이 빠졌는지 알려줍니다.

```sh
node scripts/cli.mjs detect   # 이 컴퓨터에 무엇이 있는지
node scripts/cli.mjs doctor   # 무엇이 잘못됐는지
```

## 지금 안 되어 있는 것

숨기지 않고 적습니다.

- **팀원용 Cloudflare Access 정책과 서비스 토큰이 아직 없습니다.** 거점 운영자 본인의
  주소는 등록된 WARP 기기만 통과하게 막혀 있습니다. 팀원에게 창구를 열기 전에 필요합니다
- **공유 브리지는 한 곳만 연결됩니다.** 여러 사람의 기억에 물어보는 방법은 아직 없습니다
- **공유 브리지를 연결하면 그 컴퓨터의 기억 도구는 브리지 것으로 바뀝니다.** 내 기억도
  만드는 컴퓨터(경우 B)에서 연결하면 내 기억을 직접 검색하는 도구가 보이지 않습니다
- **기억 서버의 인증이 꺼져 있습니다** (`AUTH_USE_AUTH=false`)
- 각 저장소의 `README.md`(또는 `honcho-selfhost`의 `AGENTS.md`)에 나머지 열린 항목이
  적혀 있습니다

## 기억은 어디에 있나

대화 원문과 파생된 기억은 **거점 컴퓨터의 PostgreSQL**에만 있습니다. 이 조직의 저장소에는
코드만 있습니다. 토큰과 설정은 각자 컴퓨터의 비공개 파일과 1Password에 있습니다.
