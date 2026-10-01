# team-memory-system

에이전트와 하는 대화를 각자의 기억 서버에 모으고, 팀원끼리는 "물어보는 창구" 하나만
열어 두는 자기호스팅 기억 시스템입니다.

## 어떤 구조인가

```
  내 다른 컴퓨터 (회사 노트북 등)             내 기억 서버를 둔 컴퓨터
 ┌──────────────────────┐                ┌────────────────────────────────────────┐
 │ Claude Code · Codex  │                │ Claude Code · Codex                    │
 │   └ 훅 → 수집기 ─────┼── Cloudflare ──┼▶ 문지기 → Honcho ─▶ PostgreSQL        │
 └──────────────────────┘   통로+Access  │   (토큰 확인)  │                        │
                                         │               ├─ 임베딩: Ollama(Qwen3 4B)│
 ┌──────────────────────┐                │               └─ 대화 정리 모델:        │
 │ 회사 공용 기억 서버   │ ◀── 정한 폴더의 │                  구독 게이트웨이         │
 └──────────────────────┘     대화만     │                  (내 Codex·Claude 계정) │
                                         └────────────────────────────────────────┘
```

- **사람마다 기억 서버가 하나, 데이터베이스가 하나입니다.** 컴퓨터를 여러 대 쓰더라도 대화는
  모두 자기 서버 한 곳으로 모입니다. 서버를 둔 컴퓨터 말고는 수집기만 깔고 서버 주소를 줍니다.
- **회사 서버에도 보낼 수 있습니다.** 정한 폴더에서 한 대화만 회사 공용 서버에도 함께 갑니다.
  내 서버에는 지금처럼 모든 대화가 갑니다.
- **팀원끼리 데이터베이스를 공유하지 않습니다.** 공유되는 것은 MCP 도구 `chat` 하나뿐입니다.
  팀원은 질문하고 답을 받을 뿐, 원문 메시지를 읽지 않습니다.

## 조직 저장소

| | 무엇인가 | 어디에 설치하나 |
|---|---|---|
| [honcho-agent-bridge](https://github.com/team-memory-system/honcho-agent-bridge) | 팀 메모리 앱, 수집기, 설치기, 에이전트 플러그인 (MIT) | 에이전트를 돌리는 컴퓨터마다 |
| [honcho-selfhost](https://github.com/team-memory-system/honcho-selfhost) | 기억 서버. 공식 Honcho 서브모듈에 자체 패치를 적용하는 배포 저장소 (AGPL-3.0) | 사람마다 컴퓨터 한 대 |
| [subscription-gateway](https://github.com/team-memory-system/subscription-gateway) | Codex·Claude 구독 계정을 모델 API로 바꾸고, 한도에 걸리면 다음 계정으로 넘기는 공용 게이트웨이. 다른 프로젝트에서도 따로 씁니다 (AGPL-3.0) | 기억 서버를 두는 컴퓨터 |

직접 설치하는 것은 **`honcho-agent-bridge` 하나**입니다. 기억 서버를 이 컴퓨터에 두면 설치
과정이 나머지 둘을 내려받습니다. 어느 커밋을 가져올지는 플러그인 안의
`server/honcho-source.json`, `server/gateway-source.json`에 고정돼 있어서, 원본 저장소의
최신 변경이 설치에 저절로 들어오지 않습니다.

---

# 설치

## 먼저 설치할 것

플러그인을 깔기 전에 아래 프로그램부터 있어야 합니다. 에이전트에게 설치를 맡기면 기능을 고른 뒤
이것부터 확인하고, 없으면 공식 설치 방법으로 설치할지 묻습니다. 앱의 **시작하기** 화면 첫 단계도 같은
확인입니다.

| 프로그램 | 언제 필요한가 | macOS | 윈도우 |
|---|---|---|---|
| **Claude Code 또는 Codex** | 항상 | 각 제품의 설치 안내 | 각 제품의 설치 안내 |
| **Node.js 18 이상** | 항상. 플러그인과 앱이 Node로 돕니다 | [nodejs.org](https://nodejs.org) LTS 설치 파일, 또는 `brew install node` | `winget install --id OpenJS.NodeJS.LTS -e` |
| **git** | 항상. 플러그인 설치와 서버 소스 받기에 씁니다 | `xcode-select --install` | `winget install --id Git.Git -e` |
| **Cloudflare WARP** | 대화 동기화를 **다른 컴퓨터의 서버**로 할 때 꼭. chat에는 있으면 좋음 | [WARP 내려받기](https://developers.cloudflare.com/cloudflare-one/team-and-resources/devices/warp/download-warp/), 또는 `brew install --cask cloudflare-warp` | `winget install --id Cloudflare.Warp -e` |
| Docker Desktop, Ollama | 서버 설치 | 없으면 앱이 설치합니다 | 없으면 앱이 설치합니다 |

WARP는 설치한 뒤 팀에 가입해야 합니다. 팀 이름은 Cloudflare 계정을 관리하는 사람에게 받습니다.

```sh
warp-cli registration new <팀 이름>   # 브라우저에서 팀 계정으로 로그인
warp-cli connect
warp-cli status                       # Connected 이면 됩니다
```

## 그다음: 플러그인 설치

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

이미 깔려 있으면 새 버전으로 올립니다.

```sh
claude plugin marketplace update honcho-agent-bridge
claude plugin update honcho-agent-bridge@honcho-agent-bridge
codex plugin marketplace upgrade
codex plugin add honcho-agent-bridge@honcho-agent-bridge
```

설치한 뒤 Claude Code는 `/reload-plugins`, Codex는 새 세션을 엽니다.

## 그리고: 팀 메모리 앱 열기

Claude Code에서 `/memory-setup`, Codex에서 `$setup-memory`를 실행하면 에이전트가 앱을 열어
줍니다. 직접 열려면 플러그인 폴더에서 이렇게 합니다.

```sh
node scripts/cli.mjs ui open        # http://127.0.0.1:4180
```

앱은 이 컴퓨터에서만 열립니다. 처음 열면 **시작하기** 화면에서 이 컴퓨터에서 쓸 기능을 고릅니다.
세 기능은 따로따로 켜고, 여러 개를 같이 켤 수 있습니다. 에이전트에게 설치를 맡겨도 같은 세 가지를
묻습니다.

| 기능 | 하는 일 | 필요한 것 |
|---|---|---|
| **서버 설치** | 이 컴퓨터에 내 기억 서버를 둔다. 한 사람에게 하나면 된다 | 디스크 몇 GB, Codex나 Claude 구독. Docker와 Ollama는 앱이 설치합니다 |
| **대화 동기화** | 이 컴퓨터의 Claude Code·Codex 대화를 내 기억 서버로 보낸다. 서버는 이 컴퓨터에 있어도, 다른 컴퓨터에 있어도 된다 | 서버가 다른 컴퓨터에 있으면 그 주소와 서버 토큰, Cloudflare WARP |
| **다른 사람 기억에 묻기 (chat)** | 팀원이 열어 준 창구로 그 사람 기억에 질문한다. 원문은 보지 않고 답만 받는다 | 기억 주인에게 받은 네 값 |

흔한 조합은 이렇습니다.

| 컴퓨터 | 켤 기능 |
|---|---|
| 내 서버를 두는 컴퓨터 | 서버 설치 + 대화 동기화 |
| 내 다른 컴퓨터 (회사 노트북 등) | 대화 동기화 (+ chat) |
| 팀원 기억에 묻기만 하는 컴퓨터 | chat |

고른 기능은 체크리스트로 보이고, 단계마다 지금 된 것과 안 된 것을 실제 상태로 알려 줍니다.
한 사람의 기억 서버는 한 대만 둡니다. 나중에 서버를 옮기려면 기억을 옮겨야 합니다.

---

## 서버 설치

앱의 **서버** 화면에서 위에서부터 차례로 누릅니다.

| 단계 | 앱이 하는 일 | 내가 할 일 |
|---|---|---|
| 1. 무엇을 설치할지 보기 | 받을 것과 바꿀 것을 한글로 보여 줍니다. 아직 아무것도 설치하지 않습니다 | 읽기 |
| 2. Docker와 Ollama 준비 | 없으면 받아서 설치합니다. Docker Desktop은 `/Applications`에, Ollama는 앱 폴더에 둡니다 | Docker 창이 뜨면 약관 동의, 권장 설정, macOS 암호 입력 |
| 3. 구독 게이트웨이 설치 | 게이트웨이를 받아 설치하고, 로그인할 때 자동으로 켜지게 합니다 | 없음 |
| 4. 게이트웨이에 구독 계정 로그인 | 로그인 창을 엽니다 | Codex나 Claude 계정으로 로그인 |
| 5. 기억 서버 준비 | 게이트웨이 주소·키·모델을 서버 설정에 쓰고, 임베딩 모델(Qwen3-Embedding 4B, 약 2.5GB)을 받습니다 | 없음 |
| 6. 시작 | 서버 이미지를 만들고 켭니다. 처음에는 몇 분 걸립니다 | 없음 |

끝나면 **끝에서 끝까지 점검**을 눌러 서버, 임베딩, 게이트웨이까지 모두 답하는지 확인합니다.
그다음 이 컴퓨터의 대화도 모으려면 [대화 동기화](#대화-동기화)를 켭니다. 서버를 방금 이 컴퓨터에
올렸으면 서버 주소는 비워 둡니다.

알아 둘 것:

- **Docker Desktop 라이선스.** 개인, 교육, 비영리 오픈소스, 작은 회사(직원 250명 미만이고
  연 매출 1천만 달러 미만)에서는 무료입니다. 그보다 큰 회사나 정부 기관에서는 유료 구독이
  필요합니다.
- **윈도우**에서는 Docker 설치 때 관리자 허락 창에서 **예**를 눌러야 하고, WSL 2 때문에 다시
  시작하라고 할 수 있습니다. 다시 켠 뒤 앱을 열고 이어서 준비를 누릅니다.
- **재부팅한 뒤에는** 모두 스스로 다시 뜹니다. 게이트웨이와 임베딩 감시 프로그램은 로그인할 때
  켜지게 등록돼 있고, 기억 서버는 Docker와 함께 올라옵니다. 임베딩 감시는 앱이 받은 Ollama도
  함께 띄웁니다.
- **포트.** 기억 서버는 보통 `127.0.0.1:8001`(API)과 `127.0.0.1:4173`(대시보드)을 씁니다.
  이미 쓰고 있으면 다음 빈 번호를 고르고, 앱과 수집기도 그 주소를 씁니다. 모두 이 컴퓨터
  안에서만 열립니다.
- **기억을 바꾸는 도구는 꺼진 채로 시작합니다.** 에이전트가 쓰는 MCP 도구 31개 중 기억을 쓰거나
  지우는 도구는 꺼져 있고, 찾는 도구만 켜져 있습니다. **도구·기록** 화면에서 켜고 끕니다.

### 다른 컴퓨터에서도 이 서버로 보내게 열기

회사 노트북 같은 내 다른 컴퓨터의 대화도 이 서버로 모으려면, 서버를 Cloudflare 통로로 엽니다.
앱은 서버 앞에 **문지기**를 세우고 통로를 그 문지기로만 잇습니다. 문지기는 서버 토큰이 있는
요청만 기억 서버로 넘깁니다. Cloudflare Access가 한 겹 더 지킵니다.

**가. Cloudflare에서 통로 만들기** — 둘 중 편한 쪽으로 합니다.

- **에이전트에게 맡기기.** Claude Code에 Cloudflare 플러그인(`cloudflare@claude-plugins-official`)을
  깔고 Cloudflare 계정으로 로그인한 뒤 이렇게 부탁합니다.
  > 내 기억 서버용 Cloudflare 통로를 만들어 줘. 공개 주소는 `<이름>.<내 도메인>`,
  > 서비스는 `http://localhost:<문지기 포트>`. 그 주소에 Access 앱을 만들고 내 WARP 팀
  > 사용자만 들어오게 해 줘. 통로 토큰은 채팅에 보여 주지 말고 팀 메모리 앱에 바로 넣어 줘.

  문지기 포트는 앱의 **서버 → 다른 컴퓨터에서 쓰기**에 적혀 있습니다(보통 8010).
- **대시보드에서 직접.**
  1. Cloudflare Zero Trust → Networks → Tunnels → **Create a tunnel** → Cloudflared를 고르고
     이름을 붙입니다.
  2. 설치 명령이 나오면 명령은 실행하지 말고, 그 안의 긴 토큰만 복사합니다.
  3. **Public hostname**에 쓸 주소를 정하고, Service는 HTTP, URL은 `localhost:<문지기 포트>`로
     둡니다.
  4. Access → Applications에서 그 주소를 등록하고, 내 팀의 WARP 사용자만 들어오게 정책을
     겁니다. WARP를 켤 수 없는 컴퓨터가 있으면 서비스 토큰도 하나 만듭니다.

**나. 앱에서 열기.** 서버 → **다른 컴퓨터에서 쓰기**에 공개 주소와 통로 토큰을 넣고 **열기**를
누릅니다. 앱이 문지기를 켜고, Cloudflare 프로그램(cloudflared)이 없으면 받고, 통로를 이어
로그인할 때 자동으로 열리게 해 둡니다. **밖에서 확인**을 누르면 밖에서 닿는지, Access가 막고
있는지, 토큰이 틀렸는지 구분해 알려 줍니다.

**다. 서버 토큰 복사.** 같은 칸의 **서버 토큰 복사**로 받은 값을 다른 컴퓨터의 앱에 넣습니다.
**서버 토큰 바꾸기**를 누르면 지금 토큰을 쓰는 컴퓨터는 모두 끊기니, 새 토큰을 다시 넣어야
합니다. **닫기**를 눌러도 토큰은 남겨 두어서, 다시 열면 다른 컴퓨터 설정을 고칠 필요가
없습니다.

**팀원이라면** Cloudflare 계정이 필요 없습니다. 관리자가 팀원 몫의 통로를 만들어 통로 토큰과
주소를 건네면, 팀원은 **나**부터 하면 됩니다.

---

## 대화 동기화

앱의 **연결 → 대화 동기화**에서 켭니다. 서버를 이 컴퓨터에 설치했다면 서버 주소는 비워 두고,
내 이름과 모을 에이전트만 고른 뒤 **미리 보기 → 설정하기**를 누르면 됩니다. 아래는 서버가 **다른
컴퓨터**에 있을 때입니다. 이 컴퓨터에는 서버를 두지 않고 수집기만 둡니다. Docker는 필요 없습니다.

1. **Cloudflare WARP를 팀 계정으로 켭니다.** [먼저 설치할 것](#먼저-설치할-것)의 WARP를 깔고 팀에 가입합니다. 에이전트가
   `warp-cli`로 도와줄 수 있습니다.
   ```sh
   warp-cli registration new <팀 이름>   # 처음 한 번, 브라우저에서 팀 계정으로 로그인
   warp-cli connect
   warp-cli status
   ```
2. **앱의 연결 화면**에서 기억 서버 주소(서버 컴퓨터의 공개 주소)와 서버 토큰을 넣고, 모을
   에이전트를 고른 뒤 **미리 보기 → 설정하기**를 누릅니다.
   - WARP를 켤 수 없는 컴퓨터라면 "Cloudflare WARP를 켤 수 없는 컴퓨터라면"을 펼쳐 Access
     서비스 토큰 ID와 비밀을 넣습니다.
   - Access가 막으면 "WARP를 팀 계정으로 켜거나 서비스 토큰을 넣으세요"라고 따로 알려 줍니다.
     토큰이 틀리면 "서버가 이 토큰을 받지 않습니다"라고 알려 줍니다.
3. **에이전트를 다시 시작합니다.** Claude Code는 `/reload-plugins`, Codex는 새 세션을 열고
   훅 "Syncing codex conversation to personal memory"를 승인합니다(`/hooks`에서도 됩니다).
4. **시작하기** 화면의 **첫 기억 확인**이 이 컴퓨터에서 한 대화가 서버에 들어왔는지 보여 줍니다.

토큰과 비밀은 채팅에 붙여 넣지 마세요. 앱은 이 값들을 명령행이 아닌 환경변수로만 넘기고,
화면과 기록에 다시 보여 주지 않습니다.

### 회사 서버에도 보내기

정한 폴더에서 한 대화만 회사 공용 기억 서버 같은 다른 서버에도 보냅니다. **연결 → 회사 서버에도
보내기**에서 이름, 서버 주소, 보낼 폴더, 그 서버의 토큰을 넣고 **더하기**를 누릅니다.

- 폴더 안에서 연 에이전트 대화만 갑니다. 폴더 밖의 대화와 ChatGPT에서 가져온 기록은 가지
  않습니다. 한 대화는 처음 열린 폴더로 판단합니다.
- 회사 서버가 꺼져 있어도 내 서버로 가는 것은 멈추지 않습니다. 못 보낸 대화는 기다렸다가
  이어서 보내고, 같은 대화가 두 번 들어가지 않습니다.
- 더하기 전의 지난 대화는 저절로 가지 않습니다. **지난 대화 보내기**로 날짜를 정해 따로
  보냅니다. 한 번에 500개까지 보내고, 남으면 다시 누르면 이어서 보냅니다.
- 에이전트가 기억을 찾을 때는 내 서버만 봅니다.

---

## 다른 사람 기억에 묻기 (chat)

기억 주인이 열어 준 공유 창구의 `chat`으로 그 사람 기억에 묻습니다. 대화 동기화와 같이 켤 수 있습니다.

1. **연결 → 다른 사람 기억에 묻기 (chat)**에 기억 주인에게 받은 네 값을 넣고 **연결**을 누릅니다.
   창구 주소, 창구 토큰, Cloudflare 서비스 토큰 ID와 비밀입니다. 앱이 그 값으로 창구에
   실제로 닿은 뒤에만 저장합니다.
2. **에이전트를 다시 시작합니다.** 이 컴퓨터가 대화 동기화를 하지 않으면 에이전트는 창구의 `chat`
   도구로 묻습니다. 대화 동기화도 하면 내 기억 도구는 그대로 두고, 팀원 기억에는 `shared_chat`
   도구로 묻습니다.

창구 쪽 설정(도구를 `chat` 하나로 제한, 다른 사람을 가리키지 못하게 고정, 모든 호출 기록)은
기억 주인의 서버에서 켭니다. [honcho-selfhost](https://github.com/team-memory-system/honcho-selfhost)의
`AGENTS.md`를 보세요.

---

## 앱 화면

| 화면 | 하는 일 |
|---|---|
| **기억** | 모인 대화 목록과 원문, 검색, 정리된 기억, 사람·에이전트별 보기 |
| **묻기** | 내 기억에 묻기(깊이 조절, 여러 관점 합치기), 또는 게이트웨이 모델에 바로 묻기 |
| **게이트웨이** | 구독 계정 추가·로그인·순서, 순서대로 쓰기/나눠 쓰기, 모델 목록과 시험, 기억 서버가 쓰는 모델 |
| **연결** | 대화 동기화, 회사 서버에도 보내기, 다른 사람 기억에 묻기, ChatGPT 기록 가져오기 |
| **서버** | 설치, 시작·멈추기, 다른 컴퓨터에서 쓰기, 게이트웨이·임베딩 상태, 끝에서 끝까지 점검 |
| **도구·기록** | 에이전트가 쓸 기억 도구 켜고 끄기, 공유 창구로 들어온 호출 기록 |

⌘K(윈도우는 Ctrl+K)로 화면과 동작을 바로 찾습니다.

## 터미널로 하고 싶으면

앱이 하는 일은 모두 CLI로도 됩니다. 플러그인 폴더(Claude Code는
`~/.claude/plugins/cache/honcho-agent-bridge/honcho-agent-bridge/<버전>/`)에서 실행합니다.

```sh
node scripts/cli.mjs server plan    --profile personal   # 무엇이 바뀌는지
node scripts/cli.mjs server prepare --profile personal   # Docker·Ollama·게이트웨이 설치. 로그인 전이면 멈춤
node scripts/cli.mjs gateway open                        # 게이트웨이 화면에서 로그인
node scripts/cli.mjs server prepare --profile personal   # 로그인한 뒤 다시
node scripts/cli.mjs server start   --profile personal
node scripts/cli.mjs server verify  --profile personal
node scripts/cli.mjs setup apply --agents codex,claude --user-peer <내 이름> [--honcho-url <주소>]
HONCHO_TUNNEL_TOKEN=… node scripts/cli.mjs server share enable --public-url https://<공개 주소>
HONCHO_TARGET_API_TOKEN=… node scripts/cli.mjs target add company --url <주소> --folders <폴더들>
```

토큰은 명령행 인자로 주면 거부합니다. 위처럼 환경변수로 줍니다.

## 문제가 생기면

Claude Code에서 `/memory-doctor`, Codex에서 `$setup-memory`로 진단을 요청하세요. 앱의
**연결** 화면 위쪽에도 점검에서 걸린 것이 한글로 나옵니다.

```sh
node scripts/cli.mjs detect   # 이 컴퓨터에 무엇이 있는지
node scripts/cli.mjs doctor   # 무엇이 잘못됐는지
```

자주 걸리는 것:

- **게이트웨이에 로그인했는데 모델이 없다고 나옴:** 예전에 설치한 게이트웨이의 프로그램이
  남아 포트를 잡고 있을 수 있습니다. 컴퓨터를 다시 시작하거나, 게이트웨이 화면에서 알려 주는
  프로그램을 끄세요.
- **다른 컴퓨터에서 "Cloudflare Access가 막았습니다":** 그 컴퓨터의 WARP가 팀 계정으로 켜져
  있는지(`warp-cli status`) 확인하거나 Access 서비스 토큰을 넣습니다.
- **Codex 대화가 안 모임:** 새 세션에서 훅 승인을 했는지 확인합니다.

## 지금 안 되어 있는 것

숨기지 않고 적습니다.

- **실제 Cloudflare 통로를 거친 연결은 아직 끝까지 시험하지 못했습니다.** 문지기와 앱 화면,
  다른 컴퓨터가 문지기를 거쳐 연결하는 것까지는 확인했습니다.
- **윈도우의 Docker 자동 설치는 아직 실제 기기에서 시험하지 못했습니다.** macOS는 Docker와
  Ollama가 없는 컴퓨터에서 처음부터 설치해 대화 정리·검색·묻기까지 확인했습니다.
- **공유 창구는 한 곳만 연결됩니다.** 여러 사람의 기억에 물어보는 방법은 아직 없습니다.
- **기억 서버 자체의 인증은 꺼져 있습니다** (`AUTH_USE_AUTH=false`). 서버는 이 컴퓨터 안에서만
  열리고, 밖에서 오는 요청은 문지기의 서버 토큰과 Cloudflare Access가 지킵니다.

## 기억은 어디에 있나

대화 원문과 정리된 기억은 **기억 서버를 둔 컴퓨터의 PostgreSQL**에만 있습니다. 회사 서버에도
보내기를 켰다면 정한 폴더의 대화는 그 서버에도 있습니다. 이 조직의 저장소에는 코드만 있고,
토큰과 설정은 각자 컴퓨터의 비공개 파일에 있습니다.
