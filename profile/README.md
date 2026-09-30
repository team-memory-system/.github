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

## 조직 저장소와 공용 의존성

이 조직은 기억 서버와 에이전트 브리지를 관리합니다. 구독 게이트웨이는
`chenjingdev`의 독립 프로젝트이고, 팀 메모리는 검증한 커밋을 지정해 가져다 씁니다.
게이트웨이 소스는 [honcho-agent-bridge 저장소](https://github.com/team-memory-system/honcho-agent-bridge)의
`subscription-gateway/` 서브모듈로 연결돼 있습니다. GitHub 파일 목록의 화살표 폴더를
누르면 원본의 지정된 커밋으로 이동합니다.

| | 무엇인가 | 어디에 설치하나 |
|---|---|---|
| [honcho-agent-bridge](https://github.com/team-memory-system/honcho-agent-bridge) | 수집기·설치기·에이전트 플러그인 (MIT) | 에이전트를 돌리는 기계마다 |
| [honcho-selfhost](https://github.com/team-memory-system/honcho-selfhost) | 기억 서버. 공식 Honcho 서브모듈에 자체 패치를 적용하는 배포 저장소 (AGPL-3.0) | 사람마다 컴퓨터 한 대 |
| [subscription-gateway](https://github.com/chenjingdev/subscription-gateway) | 다른 프로젝트에서도 독립적으로 쓰는 공용 구독 게이트웨이. Codex·Claude 계정을 API로 바꾸고 한도에 걸리면 다음 계정으로 넘깁니다 (AGPL-3.0) | 팀 메모리에서는 기억 서버를 두는 컴퓨터 |

직접 내려받을 저장소는 **`honcho-agent-bridge` 하나**입니다. 기억 서버를 이 컴퓨터에 두는
경우, 설치 과정이 나머지 둘을 내려받습니다. `honcho-selfhost`의 공식 서브모듈과 패치를
합쳐 `server/honcho/`에 실행 소스를 준비하고, `subscription-gateway`는 앱 폴더 아래
`runtime/subscription-gateway`로 clone 합니다. 이 소스 구조에는 플러그인 0.3.4 이상이 필요합니다.
어느 저장소의 어느 지점을 가져올지는 플러그인 안의 `server/honcho-source.json` 과
`server/gateway-source.json` 에 적혀 있습니다. 게이트웨이는 플러그인 버전별로
검증한 커밋에 고정되며, 원본 저장소의 최신 변경이 설치에 자동으로 들어오지 않습니다.

---

# 설치

## 먼저 고를 것 — 남의 서버에 붙을지, 이 컴퓨터를 서버로 둘지

설치를 시작하기 전에 이것부터 정해야 합니다. 나중에 바꾸려면 기억을 옮겨야 하기 때문입니다.
에이전트에게 설치를 맡길 때도 에이전트가 먼저 이걸 물어야 하고, 답을 듣기 전에는
아무것도 설치하지 않습니다.

| | **남의 서버에 붙는다** | **이 컴퓨터를 서버로 둔다** |
|---|---|---|
| 내 Postgres | 없음 | 이 컴퓨터에 생김 |
| 내 대화가 쌓이는 곳 | 안 쌓임 | 이 컴퓨터 |
| 볼 수 있는 것 | 거점 주인이 열어 준 `chat` 창구 | 내 기억 전부 |
| 필요한 것 | Node, 그리고 거점 주인에게 받은 네 값 | Docker Desktop, 디스크 몇 GB |
| 해당 절차 | **경우 A** | **경우 B**(나만) 또는 **경우 C**(팀 거점) |

- 팀원은 각자 자기 PC에 서버를 둡니다. **이 컴퓨터를 서버로 둔다**, 경우 B로 가세요.
  다른 사람도 이 서버에 붙게 할 거면 경우 C입니다.
- 내 대화는 모으지 않고 다른 사람의 기억에 물어보기만 할 거면 **남의 서버에 붙는다**,
  경우 A입니다.
- 내 서버가 이미 내 다른 컴퓨터에 있으면 이 컴퓨터에는 서버를 또 두지 않습니다.
  [경우 B-2](#경우-b-2--내-서버가-내-다른-컴퓨터에-있을-때)로 가세요.
- 이미 기억 서버가 있는 컴퓨터에서 또 설치하지 마세요. `bridge status`와 `doctor`로
  지금 상태를 먼저 확인합니다.

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

이미 깔려 있으면 새 버전으로 올립니다.

```sh
claude plugin marketplace update honcho-agent-bridge
claude plugin update honcho-agent-bridge@honcho-agent-bridge
codex plugin marketplace upgrade
codex plugin add honcho-agent-bridge@honcho-agent-bridge
```

### 2. 설치 명령

Claude Code에서 `/memory-setup`, Codex에서 `$setup-memory`.

**Docker Desktop, Node.js 18 이상, `git`, Ollama, 그리고 Codex나 Claude 구독이 있어야
합니다.** 기억 서버와 게이트웨이 소스는 플러그인에 들어 있지 않고 설치 과정이 내려받습니다.
Docker Desktop이 꺼져 있으면 설치 과정이 켜고 엔진이 뜰 때까지 기다립니다.

에이전트가 순서대로 물어봅니다. 하는 일은 이렇습니다.

1. **지금 상태 확인** — 어떤 에이전트가 깔려 있는지, 기억 서버가 이미 도는지
2. **서버를 어떻게 할지** — 이미 도는 서버가 있으면 그걸 씁니다. 없으면 이 컴퓨터에
   Docker로 올릴지 물어봅니다. `server plan` 이 내려받을 소스와 게이트웨이 설치를 먼저
   알려 주고, 실제 내려받기와 설치는 `server prepare` 가 합니다
3. **구독 계정 로그인** — `server prepare` 가 구독 게이트웨이를 설치하고 그 화면
   (`http://127.0.0.1:11450`)을 열어 달라고 합니다. 거기서 Codex나 Claude 계정으로
   로그인하면 기억 서버가 그 계정의 모델을 씁니다. 계정을 여러 개 넣으면 한도에 걸린
   계정은 건너뜁니다. 이 로그인은 내 `codex`·`claude` 명령의 로그인과 따로 보관됩니다
4. **어느 에이전트에 붙일지** — 감지된 것 전부가 기본값입니다
5. **기억 저장 위치** — 운영체제 기본 위치를 권합니다
6. **내 이름 (peer ID)** — 기억 안에서 나를 가리키는 이름입니다
7. **계획을 먼저 보여주고** 확인을 받은 뒤에 바꿉니다. 고치는 파일은 미리 백업합니다

에이전트 없이 터미널에서 할 때 마지막 단계는 이 명령입니다. 서버를 방금 이 컴퓨터에 올렸으면
`--honcho-url`은 필요 없습니다. 전에 다른 서버로 보내던 컴퓨터면 `server start`가 알려 준
`apiUrl`을 `--honcho-url`로 줍니다.

```sh
node <플러그인 폴더>/scripts/cli.mjs setup apply --agents codex,claude --user-peer <내 이름>
```

설치가 끝나면 세 가지만 확인합니다.

- **Codex:** 새 세션을 열면 Codex가 새 훅을 승인하라고 묻습니다(`/hooks`에서도 됩니다).
  "Syncing codex conversation to personal memory" 하나만 승인하세요. 승인하기 전에는
  Codex 대화가 모이지 않습니다.
- **Claude Code:** 설치 전에 열어 둔 세션은 `/reload-plugins`를 실행하거나 다시 시작합니다.
  새로 여는 세션은 할 것이 없습니다. 다시 불러오기 전의 대화도 다음 턴에 같이 들어갑니다.
- **대시보드:** 모인 대화와 기억은 `server start`가 알려 주는 `dashboardUrl`(보통
  `http://127.0.0.1:4173`)에서 봅니다.

그다음부터는 대화가 끝날 때마다 자동으로 모입니다.

**포트:** 기억 서버는 `127.0.0.1:8001`(API)과 `127.0.0.1:4173`(대시보드)을 씁니다. 다른
프로그램이 이미 쓰고 있으면 설치 과정이 다음 빈 번호를 골라 설치된 `.env`에 적고, 수집기도
그 주소로 붙습니다.

**기억을 바꾸는 도구는 꺼진 채로 시작합니다.** 에이전트가 쓰는 MCP 도구 중 기억을 쓰거나
지우는 12개(`delete_session` 등)는 꺼져 있고, 기억을 찾는 도구만 켜져 있습니다. 켜려면
데이터 폴더의 `mcp-tools.json`에서 뺍니다.

## 경우 B-2 — 내 서버가 내 다른 컴퓨터에 있을 때

한 사람의 대화는 서버 한 곳으로 모읍니다. 서버를 둔 컴퓨터 말고 다른 컴퓨터에서도 에이전트를
쓰면, 그 컴퓨터에는 수집기만 깔고 내 서버 주소를 줍니다. Docker는 필요 없습니다.

1. **플러그인을 설치합니다.** 경우 B의 1번과 같습니다.
2. **서버 주소를 주고 설치합니다.** 에이전트에게 "내 서버는 `<주소>`에 있다"고 말하면
   `server ...` 단계를 건너뛰고 `setup`만 합니다.
3. **서버가 토큰을 요구하면** 토큰은 채팅에 붙여넣지 말고 내 터미널에서 넣습니다.

   ```sh
   HONCHO_API_TOKEN=<토큰> node <플러그인 폴더>/scripts/cli.mjs setup apply \
     --agents codex,claude --user-peer <내 이름> --honcho-url <주소>
   ```

   `--api-token` 같은 명령행 인자로 주면 거부합니다.
4. 위 경우 B의 "세 가지"(Codex 훅 승인, `/reload-plugins`, 대시보드)를 확인합니다.
   대시보드는 서버를 둔 컴퓨터에 있습니다.

### 화면으로 하고 싶으면

터미널 대신 브라우저에서 같은 일을 할 수 있습니다. `/memory-setup`에서 화면을 열어 달라고
하거나, 저장소에서 직접 엽니다.

```sh
node scripts/cli.mjs ui open
```

상태 확인, 다른 사람의 기억에 연결, 훅 설치, 서버·게이트웨이 켜기, ChatGPT 내보내기 파일
올리기까지 한 화면입니다. 이 컴퓨터에서만 열립니다.

## 경우 C — 거점을 직접 운영할 때

기억 서버와 구독 게이트웨이를 자기 컴퓨터에서 돌립니다.

**필요한 것**

| | |
|---|---|
| 운영체제 | macOS 또는 윈도우. 네이티브 리눅스는 `portable` 프로필만 |
| Docker | Desktop 또는 Engine + Compose. 윈도우는 WSL 2 백엔드 |
| Node.js | 18 이상 |
| Ollama | 임베딩용. `PATH`에 있어야 합니다 |
| git | 기억 서버와 게이트웨이 소스를 내려받습니다 |
| 구독 계정 | Codex나 Claude. 게이트웨이 화면에서 로그인하고, 그 로그인은 내 `codex`·`claude` 명령과 따로 보관됩니다 |

**순서**

`scripts/cli.mjs` 는 설치된 플러그인 안에 있습니다. Claude Code 는
`~/.claude/plugins/cache/honcho-agent-bridge/honcho-agent-bridge/<버전>/`,
Codex 는 `~/.codex/plugins/cache/honcho-agent-bridge/honcho-agent-bridge/<버전>/`
입니다. 그 디렉터리에서 실행하세요. `server prepare` 가 `honcho-selfhost` 를
`server/honcho/` 로 clone 한 다음 이미지를 빌드합니다.

```sh
node scripts/cli.mjs server plan    --profile personal   # 무엇이 바뀌는지 먼저 봅니다
node scripts/cli.mjs server prepare --profile personal   # 게이트웨이를 설치하고, 로그인 전이면 여기서 멈춥니다
node scripts/cli.mjs gateway open                        # 게이트웨이 화면에서 Codex·Claude 로그인
node scripts/cli.mjs server prepare --profile personal   # 로그인한 뒤 다시: 라우터 주소·키·모델을 .env 에 씁니다
node scripts/cli.mjs server start   --profile personal
node scripts/cli.mjs server verify  --profile personal
node scripts/cli.mjs host status    --profile personal   # 게이트웨이와 Ollama 상태
```

**재부팅한 뒤에는** 게이트웨이가 스스로 다시 뜹니다. 설치할 때 자기 자동 시작(macOS는
LaunchAgent, 윈도우는 로그온 실행 항목)을 등록하기 때문입니다. 이 플러그인 자체는
launchd·작업 스케줄러·systemd에 아무것도 등록하지 않습니다. 그래서 Qwen 임베딩 모델을
올려 두는 Ollama 감시 프로세스는 `host start`를 실행하거나 설치 화면에서 켜야 다시 돕니다.
그동안 기억 저장은 계속되고, 임베딩은 Ollama 앱이 떠 있을 때만 됩니다.

기억 서버 자체는 Docker의 `restart: unless-stopped`로 다시 올라옵니다.

**팀원에게 창구를 열려면** Cloudflare 터널과 Access 정책이 필요합니다. 공유용 브리지에는
네 가지 장치가 있는데, **켜야 동작합니다.** 코드에 있다고 켜져 있는 것이 아닙니다.

| 장치 | 켜는 방법 |
|---|---|
| 도구를 `chat` 하나로 제한 | `HONCHO_MCP_ENABLED_TOOLS=chat` 또는 대시보드의 도구 설정 |
| 다른 워크스페이스·피어를 가리키지 못하게 고정 | `HONCHO_MCP_PIN_DEFAULTS=1`. 헤더와 도구 인자 양쪽을 막습니다 |
| 틀린 토큰을 연결 단계에서 거부 | `HONCHO_MCP_BEARER_TOKEN_FILE` 설정. 없으면 브리지가 열린 채로 뜹니다 |
| 모든 호출을 질의 원문과 함께 기록 | `HONCHO_AUDIT_DSN` |

거점을 올린 뒤 이 네 개가 실제로 설정돼 있는지 확인하세요. 하나라도 빠지면 팀원 설치기의
연결 확인이 성공으로 나온 뒤 첫 질문에서 거부되거나, 조회 기록이 아무것도 남지 않습니다.

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
- **지금 도는 거점의 공유 브리지는 위 네 장치 중 둘이 빠져 있습니다.** 틀린 토큰을 연결
  단계에서 거부하는 검사와 조회 기록이 아직 그 기계에 적용되지 않았습니다. 코드에는 있고
  프로세스 재시작과 설정 추가가 남았습니다
- 각 저장소의 `README.md`(또는 `honcho-selfhost`의 `AGENTS.md`)에 나머지 열린 항목이
  적혀 있습니다

## 기억은 어디에 있나

대화 원문과 파생된 기억은 **거점 컴퓨터의 PostgreSQL**에만 있습니다. 이 조직의 저장소에는
코드만 있습니다. 토큰과 설정은 각자 컴퓨터의 비공개 파일과 1Password에 있습니다.
