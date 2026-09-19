# Hysteria 2 분석 & 활용 정리 (한국어)

> 이 문서는 Claude Code 세션에서 진행한 Hysteria 2 코드베이스 분석과
> Q&A, 수익화 아이디어 논의를 정리한 기록입니다.
>
> - 정리 일자: 2026-09-18
> - 대상 저장소(포크): https://github.com/bmshin94/hysteria
> - 원본 저장소(업스트림): https://github.com/apernet/hysteria
> - 공식 문서: https://v2.hysteria.network/
> - 프로토콜 스펙: [PROTOCOL.md](../PROTOCOL.md)

---

## 목차

1. [Hysteria란 무엇인가](#1-hysteria란-무엇인가)
2. [핵심 동작 원리](#2-핵심-동작-원리)
3. [저장소 구조](#3-저장소-구조)
4. [설치 및 사용법](#4-설치-및-사용법)
5. [Q&A 정리](#5-qa-정리)
6. [수익화 아이디어](#6-수익화-아이디어)
7. [참고 링크](#7-참고-링크)

---

## 1. Hysteria란 무엇인가

**한 줄 요약: QUIC 기반의 고속 · 검열 저항 프록시(서버 + 클라이언트).**

- 언어: **Go** (Go 파일 약 214개 / 약 39,000 LOC)
- 라이선스: **MIT**
- 결과물: `hysteria` 실행 파일 **1개** (서버 모드 / 클라이언트 모드 겸용)
- 이 저장소는 `apernet/hysteria`를 포크한 것 (`README.md`의 배지가 업스트림을 가리킴)

### 이런 상황에 쓴다

| 상황 | 활용 |
|---|---|
| 해외망 속도 저하 | 패킷 로스가 많은 국제 회선에서 처리량 회복 |
| 검열/차단 환경 | HTTP/3 위장으로 탐지·차단 회피 |
| 원격 사내망 접속 | TUN 모드로 VPN처럼 사용 |
| 게임 가속 | UDP 릴레이 + 낮은 지연 |
| 프록시 서비스 운영 | HTTP 인증 + 트래픽 통계 API로 유저 관리형 서비스 구성 |
| NAT 뒤 서버 노출 | `realm` 홀펀칭으로 공인 IP 없이 서버 운영 |

> ⚠️ 프록시 운영은 국가·조직 정책에 따라 제약이 있을 수 있습니다. 개인 인프라 용도를
> 넘어 서비스로 제공할 경우 해당 관할의 법령 검토가 필요합니다.

---

## 2. 핵심 동작 원리

### 2.1 QUIC 위에서 동작 (TCP 아님)

표준 QUIC([RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000)) +
Unreliable Datagram 확장([RFC 9221](https://datatracker.ietf.org/doc/rfc9221/)) 위에 구현.
UDP 기반이라 패킷 손실이 큰 구간에서도 성능이 덜 무너진다.

### 2.2 HTTP/3 웹서버 위장 (masquerade)

서버는 반드시 HTTP/3 서버([RFC 9114](https://datatracker.ietf.org/doc/rfc9114/))로도 동작해야 한다.

클라이언트가 접속 시 보내는 특수 요청:

```
:method: POST
:path: /auth
:host: hysteria
Hysteria-Auth:    [인증 문자열]
Hysteria-CC-RX:   [클라이언트 최대 수신 속도, bytes/s]
Hysteria-Padding: [랜덤 패딩]
```

인증 성공 시 서버 응답:

```
:status: 233 HyOK
Hysteria-UDP:     [true/false]
Hysteria-CC-RX:   [uint 또는 "auto"]
Hysteria-Padding: [랜덤 패딩]
```

- 인증에 실패하거나 무관한 접속자는 **평범한 웹서버처럼 응답**하거나
  업스트림 사이트로 리버스 프록시된다 (`extras/masq/server.go`).
- 따라서 능동 탐지(active probing)로 Hysteria 서버를 특정하기 어렵다.
  차단하려면 정상 HTTP/3 트래픽까지 함께 막아야 하므로 **부수적 피해**가 크다.
- `Hysteria-Padding`으로 요청/응답 길이 패턴까지 흐린다.

### 2.3 커스텀 혼잡 제어 (Brutal)

`core/internal/congestion/`

- 일반 TCP: 패킷 손실 = 혼잡 신호 → 전송률 급감
- Hysteria: 사용자가 선언한 대역폭을 기준으로 **전송률을 유지**하고 손실분만 재전송

→ 손실률이 높은 장거리 회선에서 체감 속도 차이가 크다.

> 회선을 공격적으로 사용하는 방식이므로, 설정의 `bandwidth` 값은 **실제 회선 속도로
> 정직하게** 적는 것이 권장된다.

### 2.4 그 외 회피/확장 기술

- **obfs** (`extras/obfs/`): `salamander`, `gecko` — QUIC 패킷 자체를 난독화해 DPI 회피
- **udphop** (`extras/transport/udphop/`): 포트 호핑으로 QoS/차단 회피
- **mimic** (`app/internal/mimic/`): [Mimic](https://github.com/hack3ric/mimic)을 자식
  프로세스로 실행해 **UDP를 커널에서 TCP처럼 위장** (UDP 자체를 막는 망 대응)
- **realm** (`extras/realm/`): STUN + 홀펀칭으로 **공인 IP 없이** 서버 운영
- **ECH** (`app/cmd/ech.go`): Encrypted Client Hello 키 생성 지원

---

## 3. 저장소 구조

### 3.1 최상위

```
core/        프로토콜 핵심 (독립 Go 모듈) — 라이브러리로 재사용 가능
app/         실제 CLI 프로그램 (독립 Go 모듈)
extras/      부가 기능 모음 (독립 Go 모듈)
.github/     CI/CD 워크플로 (build/test/docker/release/autotag 등)
scripts/     install_server.sh (원커맨드 서버 설치 스크립트)
hyperbole.py 자체 제작 빌드 스크립트
platforms.txt 릴리스 빌드 대상 (35개 플랫폼/아키텍처 조합)
PROTOCOL.md  RFC 스타일 프로토콜 스펙
Dockerfile   멀티스테이지 빌드 (golang:alpine → alpine)
```

세 모듈은 `go.work`(Go 1.26)로 묶인 **워크스페이스** 구조다.
`core`만 따로 가져다 별도 앱(예: 모바일 클라이언트)을 만들 수 있게 분리되어 있다.

### 3.2 `core/` — 프로토콜 핵심

```
core/client/              클라이언트 구현 (재연결, UDP 세션)
core/server/              서버 구현 (스트림/UDP 처리, 트래픽 로깅 훅)
core/internal/congestion/ Brutal 혼잡 제어
core/internal/frag/       UDP 프래그먼트 처리
core/internal/pmtud/      Path MTU Discovery
core/internal/protocol/   와이어 포맷
core/internal/integration_tests/ 통합 테스트
```

### 3.3 `app/` — CLI

CLI 명령 (`app/cmd/`):

| 명령 | 설명 |
|---|---|
| `hysteria server` | 서버 모드 |
| `hysteria client` | 클라이언트 모드 |
| `hysteria ping` | 서버 연결 확인 |
| `hysteria speedtest` | 속도 측정 |
| `hysteria share --qr` | 설정 공유용 URI / QR 코드 생성 |
| `hysteria cert` | 자체 서명 TLS 인증서 생성 |
| `hysteria ech` | ECH 키 및 클라이언트 설정 생성 |
| `hysteria check-update` | 업데이트 확인 |
| `hysteria version` | 버전 출력 |

클라이언트 인바운드 모드 (`app/internal/`):

| 모드 | 설명 |
|---|---|
| `socks5/` | SOCKS5 프록시 (가장 일반적) |
| `http/` | HTTP 프록시 |
| `tun/` | 가상 네트워크 인터페이스 — 시스템 전체 트래픽 우회 |
| `tproxy/` | 리눅스 투명 프록시 |
| `redirect/` | iptables REDIRECT 기반 |
| `forwarding/` | 특정 TCP/UDP 포트만 터널링 |
| `proxymux/` | 한 포트에서 SOCKS5/HTTP 동시 처리 |

### 3.4 `extras/` — 부가 기능

| 패키지 | 설명 |
|---|---|
| `auth/` | 인증 백엔드 4종: `password`, `userpass`, **`http`**, `command` |
| `obfs/` | 난독화: `salamander`, `gecko` |
| `masq/` | 위장 웹서버 / 리버스 프록시 |
| `outbounds/` | 아웃바운드 선택 (direct / SOCKS5 / HTTP 체이닝), DNS 리졸버 |
| `outbounds/acl/` | ACL 규칙 엔진 (도메인·IP·GeoIP, v2geo 데이터 지원) |
| `sniff/` | TLS SNI 등으로 도메인 추출 → ACL 정확도 향상 |
| `trafficlogger/` | **유저별 트래픽 통계 HTTP API** |
| `realm/` | STUN + 홀펀칭 + 포트 매핑 |
| `transport/udphop/` | 포트 호핑 |

### 3.5 서비스화에 중요한 두 개의 확장점

**(1) HTTP 인증 위임** — `extras/auth/http.go`

서버가 접속 요청마다 외부 API에 인증을 물어본다.

요청 (서버 → 내 API, `POST` / `application/json`):

```json
{ "addr": "1.2.3.4:5678", "auth": "유저 토큰", "tx": 10000000 }
```

응답 (내 API → 서버, HTTP 200):

```json
{ "ok": true, "id": "user_123" }
```

- `ok`: 접속 허용 여부
- `id`: 트래픽 통계에서 이 유저를 식별할 키
- 타임아웃 10초, `insecure` 옵션으로 TLS 검증 생략 가능

**(2) 트래픽 통계 API** — `extras/trafficlogger/http.go`

`Authorization` 헤더에 설정한 시크릿을 넣어 호출한다.

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/traffic` | 유저별 `{tx, rx}` 누적 바이트. `?clear=true`로 읽고 초기화 |
| GET | `/online` | 유저별 현재 접속 수 |
| POST | `/kick` | 유저 목록을 받아 강제 접속 종료 |
| GET | `/dump/streams` | 현재 스트림 상세 덤프 |

> 이 두 가지 덕분에 **Go 코드를 수정하지 않고도** 유저 관리·과금·차단을
> 외부 웹 애플리케이션으로 구현할 수 있다.

---

## 4. 설치 및 사용법

### 4.1 서버 설치 (Linux + systemd)

```bash
bash <(curl -fsSL https://get.hy2.sh/)
```

`scripts/install_server.sh`가 수행하는 일:

- OS/아키텍처 자동 감지 (386, amd64, arm, arm64, mipsle, s390x 등)
- `/usr/local/bin/hysteria` 설치
- 전용 실행 유저 생성 (root 실행 회피)
- `/etc/hysteria/config.yaml` 생성
- `/etc/systemd/system/`에 systemd 유닛 등록
- SELinux 컨텍스트 설정
- systemd 필수 (`FORCE_NO_SYSTEMD`로 우회 가능)

### 4.2 서버 설정 예시

```yaml
listen: :443

# 도메인이 있는 경우: 인증서 자동 발급 (권장)
acme:
  domains:
    - my.domain.com
  email: me@example.com

# 도메인이 없는 경우: 자체 서명 인증서
# tls:
#   cert: /etc/hysteria/cert.crt
#   key: /etc/hysteria/private.key

auth:
  type: password
  password: 충분히_긴_랜덤_문자열

# 인증 실패한 접속자에게 보여줄 위장 사이트
masquerade:
  type: proxy
  proxy:
    url: https://news.ycombinator.com/
    rewriteHost: true

# (선택) 유저 관리형 서비스로 쓸 때
# auth:
#   type: http
#   http:
#     url: https://my.service/hysteria/auth
# trafficStats:
#   listen: 127.0.0.1:7653
#   secret: 통계_API_시크릿
```

```bash
systemctl enable --now hysteria-server.service
journalctl -u hysteria-server -f
```

### 4.3 클라이언트 설정 예시

```yaml
server: my.domain.com:443

auth: 충분히_긴_랜덤_문자열

bandwidth:
  up: 50 mbps      # 실제 회선 속도를 정직하게 입력
  down: 200 mbps

socks5:
  listen: 127.0.0.1:1080
http:
  listen: 127.0.0.1:8080
```

```bash
hysteria client -c client.yaml
```

브라우저/앱에서 `SOCKS5 127.0.0.1:1080` 또는 `HTTP 127.0.0.1:8080`을 사용한다.

### 4.4 Docker

```bash
docker run -d --name hysteria -p 443:443/udp \
  -v ./config.yaml:/etc/hysteria.yaml \
  tobyxdd/hysteria server -c /etc/hysteria.yaml
```

### 4.5 소스 빌드

```bash
python hyperbole.py build       # 현재 플랫폼
python hyperbole.py build -r    # 릴리스 빌드
python hyperbole.py build -a    # platforms.txt 전체
```

Python 개발 도구 의존성은 `pyproject.toml` 참고 (requests, flask, pysocks, cryptography).

---

## 5. Q&A 정리

### Q. 플러그인인가, 스킬인가, MCP인가?

**셋 다 아니다.** Hysteria는 Go로 작성된 **독립 실행형 네트워크 프록시 소프트웨어**로,
`nginx`, `WireGuard`, `OpenVPN`과 같은 인프라 소프트웨어 계열이다. AI와 직접적인 관련이 없다.

| | 정체 | 실행 위치 | 언어 |
|---|---|---|---|
| Hysteria | 네트워크 프록시 데몬 | 독립 프로세스 | Go |
| 플러그인 | Claude Code 기능 확장 묶음 | Claude Code 내부 | JSON/MD |
| 스킬 | AI에게 주는 전문 지식 문서 | AI 컨텍스트 | Markdown |
| MCP | AI ↔ 외부 도구 연결 프로토콜 | MCP 서버 프로세스 | 무관 |

다만 Hysteria의 트래픽 통계 API를 감싸는 **MCP 서버를 별도로 만들 수는 있다**
(그건 우리가 만드는 것이지, Hysteria가 MCP인 것은 아니다).

또한 "플러그인 시스템"은 없지만 확장점은 있다:
`auth: command`(외부 실행 파일), `auth: http`(외부 API), `outbounds` 체이닝,
Go 코드상의 `PluggableOutbound` 인터페이스.

### Q. API 토큰이 필요한가?

**아니다.** 회원가입·API 키·클라우드 계정·구독료가 전혀 없는 MIT 오픈소스다.
서버와 클라이언트가 직접 통신하며 중개 업체가 개입하지 않는다.

혼동하기 쉬운 부분 — 아래는 "외부 서비스 토큰"이 아니라 **내가 정하는 값**이다:

| 항목 | 설명 | 누가 정하나 |
|---|---|---|
| `auth.password` | 서버 접속 비밀번호 | 내가 직접 |
| `trafficStats.secret` | 통계 API 보호 키 | 내가 직접 |
| TLS 인증서 | HTTPS 증명서 | Let's Encrypt 자동 발급(무료) 또는 자체 서명 |

실제로 비용이 드는 것은 소프트웨어가 아니라 **VPS 임대료(월 $3~6 수준)** 와
선택 사항인 도메인 비용이다.

외부 토큰이 필요한 경우는 **선택적**이다:
- ACME **DNS-01** 챌린지 사용 시 DNS 제공자 API 토큰
  (코드상 지원: Cloudflare, DuckDNS, Gandi, GoDaddy, Namecheap, Njalla, Porkbun, Vultr)
- `hysteria check-update`는 `api.hy2.io`를 조회한다 (비활성화 가능)

### Q. 왜 GitHub에서 유명한가?

1. **실수요가 매우 크다** — 검열 환경에서 사실상 필수 도구
2. **체감 성능 차이가 명확하다** — 손실 많은 회선에서 눈에 띄게 빠름
3. **기술적 독창성** — QUIC + 커스텀 혼잡 제어 + HTTP/3 위장의 조합
4. **RFC 스타일 스펙 공개**(`PROTOCOL.md`) → 서드파티 재구현이 활발
   (v2rayN, Clash, sing-box, Shadowrocket 등)
5. **문서/배포 품질** — 전용 문서 사이트, 중국어 번역, 원커맨드 설치, 35개 플랫폼 빌드
6. **생태계 형성** — 관리 패널, 봇, 클라이언트 앱이 계속 생기는 선순환
7. **커뮤니티 운영** — Telegram, GitHub Discussions, MIT 라이선스

### Q. 로컬 에이전트 구축에 도움이 되나?

**직접적으로는 아니지만(LLM/에이전트 기능 없음), 우회적으로는 유용하다.**

1. **API 아웃바운드 안정화** — 차단·지연되는 망에서 LLM API 호출 경로 확보
   (`HTTPS_PROXY=socks5://127.0.0.1:1080` 한 줄로 적용)
2. **분산 에이전트 연결** — `realm` 홀펀칭으로 공인 IP 없는 집 GPU 서버(로컬 LLM 등)에
   외부에서 접근
3. **에이전트용 툴 소재** — `/traffic`, `/online`, `/kick` API를 MCP 서버로 감싸면
   "인프라 운영 에이전트"를 만들 수 있다
4. **학습 자료** — Go 동시성, 인터페이스 설계, mock 기반 테스트, 워크스페이스 구조

주의: 에이전트의 모든 트래픽을 무조건 우회시키면 의도치 않은 지역/트래픽 문제가
생길 수 있으므로 ACL로 대상을 명시적으로 통제하는 편이 좋다.

### Q. React나 PHP로 만들 수 있나?

**두 가지로 나눠서 봐야 한다.**

**(1) Hysteria 자체를 React/PHP로 재구현 → 사실상 불가능**

| 필요 요소 | React/PHP |
|---|---|
| QUIC 저수준 제어 | 불가 |
| UDP 소켓 직접 조작 | PHP는 가능하나 성능상 비현실적 |
| 패킷 단위 혼잡 제어 | 불가 |
| TUN 인터페이스 / eBPF | 불가 |
| 단일 바이너리 크로스 컴파일 | 불가 |

React는 브라우저 환경이라 raw 소켓 접근 자체가 막혀 있고, PHP는 요청-응답형이라
장시간 유지되는 고성능 네트워크 데몬에 부적합하다. Go/Rust/C++를 쓰는 이유다.

**(2) Hysteria를 "관리하는" 웹 서비스를 React/PHP로 → 완벽하게 가능하며, 이쪽이 정답**

Hysteria가 이미 HTTP 인터페이스를 열어두었기 때문에 Go 코드를 전혀 건드리지 않아도 된다.

PHP 인증 엔드포인트 예시 (`auth: http`의 수신 측):

```php
<?php
// config.yaml → auth: { type: http, http: { url: "https://my.site/auth" } }

$req = json_decode(file_get_contents('php://input'), true);
// $req = ["addr" => "1.2.3.4:5678", "auth" => "유저 토큰", "tx" => 10000000]

$user = $db->findByToken($req['auth']);

if (!$user || $user->expired_at < time() || $user->traffic_used > $user->traffic_quota) {
    echo json_encode(["ok" => false]);
    exit;
}

echo json_encode(["ok" => true, "id" => "user_{$user->id}"]);
```

React 관리자 대시보드 예시 (통계 API 소비 측):

```jsx
function Dashboard() {
  const [traffic, setTraffic] = useState({});
  const [online, setOnline] = useState({});

  useEffect(() => {
    const tick = async () => {
      const h = { Authorization: SECRET };
      setTraffic(await (await fetch(`${API}/traffic`, { headers: h })).json());
      setOnline(await (await fetch(`${API}/online`, { headers: h })).json());
    };
    tick();
    const t = setInterval(tick, 5000);
    return () => clearInterval(t);
  }, []);

  const kick = (id) =>
    fetch(`${API}/kick`, {
      method: 'POST',
      headers: { Authorization: SECRET },
      body: JSON.stringify([id]),
    });

  return <UserTable traffic={traffic} online={online} onKick={kick} />;
}
```

구성도:

```
┌──────────────────┐     ┌────────────────────┐     ┌──────────────┐
│  React 대시보드   │ ←→  │  PHP(Laravel) API  │ ←→  │  MySQL       │
│  유저/통계/결제    │     │  인증 + 과금        │     │  유저/요금제  │
└──────────────────┘     └────────────────────┘     └──────────────┘
                                   ↕ HTTP
                          ┌────────────────────┐
                          │  Hysteria 서버      │ ← Go 바이너리 (수정 불필요)
                          │  (다중 지역 노드)    │
                          └────────────────────┘
```

---

## 6. 수익화 아이디어

> ⚠️ 프록시/VPN 트래픽 자체를 판매하는 모델은 법적·규제 리스크가 있다.
> 아래에서는 **"트래픽이 아니라 도구/소프트웨어를 파는 모델"** 을 우선 추천한다.
> 실행 전 관할 법령 및 전문가 검토 필요.

### 6.1 셀프호스팅 관리 패널 SaaS (최우선 추천)

**컨셉**: Hysteria 서버는 고객이 직접 운영하고, 우리는 그것을 관리하는 도구를 판매.
트래픽을 직접 중개하지 않으므로 리스크가 낮다.

**기능**
- 실시간 대시보드 (`/traffic`, `/online` 폴링 → 차트)
- 유저 관리 (생성/정지/삭제, 트래픽 쿼터, 만료일)
- 멀티 노드 통합 관리
- 구독 링크 / QR 코드 발급 (`hysteria share` 활용)
- 알림 (쿼터 초과, 노드 다운, 이상 트래픽)
- 월간 사용량 리포트
- 권한 관리 (관리자 / 서브관리자 / 조회 전용)

**가격 모델 예시**

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | $0 | 노드 1개, 유저 5명 |
| Pro | $9/월 | 노드 5개, 유저 100명, 알림 |
| Business | $29/월 | 무제한, 화이트라벨, API |
| Enterprise | 문의 | 온프레미스 + 지원 |

**스택**: React + TypeScript + Tailwind + shadcn/ui + Recharts /
Laravel(PHP) 또는 Next.js API Routes / MySQL / Stripe·Paddle·토스페이먼츠 /
Docker Compose 원클릭 설치

**장점**: 기존 React·PHP 역량 활용, Go 미수정, 트래픽 비용 0, 법적 리스크 낮음,
오픈소스 공개로 마케팅 가능
**리스크**: 기존 경쟁 패널 존재 → UI 품질·멀티노드 편의성·문서 품질로 차별화 필요

### 6.2 게임 / 스트리밍 가속기

"VPN"이 아닌 **"지연 감소 가속기"** 포지셔닝. Brutal 혼잡 제어와 UDP 릴레이가 적합하다.

- 게임 트래픽 자동 감지 (`sniff` + `acl`)
- 지역별 노드 선택, 실시간 핑 비교 표시
- 스플릿 터널링 (게임만 우회, 나머지는 직통)
- Electron/Tauri + React 데스크톱 앱
- 가격: 무료 체험 후 월 ₩9,900 / 연 ₩79,000 수준

핵심 기술 기반은 `extras/outbounds/acl/` 규칙 엔진.
**리스크**: 게임사 이용약관 확인 필요 (지역 우회는 제재 사유가 될 수 있음).

### 6.3 기업용 원격접속 솔루션 (B2B)

개인 대상 대비 단가가 크게 높다.

| 항목 | 가격대 |
|---|---|
| 구축 서비스 (설계·설치·교육·문서) | 1회 200~500만원 |
| 운영 관리 (모니터링·업데이트·장애 대응) | 월 30~100만원 |
| 관리 패널 라이선스 | 월 20~50만원 |

**세일즈 포인트**: 상용 VPN 라이선스 대비 저렴 / `realm` 홀펀칭으로 공인 IP 없는
지사·공장 연결 / `trafficStats` 감사 로그 / `acl` 부서별 접근 제어 / 온프레미스 설치

**리스크**: 영업 난이도, 장애 시 책임 소지 → 계약서·SLA 정비 필요.

### 6.4 모바일 / 데스크톱 클라이언트 앱

`core` 모듈을 라이브러리로 사용.

- Android: `gomobile`로 `.aar` 빌드 → Kotlin/Flutter UI
- iOS: Network Extension + `gomobile`
- Desktop: Tauri(Rust) 또는 Electron + React로 바이너리 제어

수익: 앱 내 프리미엄 기능(멀티노드, 자동 전환, 위젯) / 무료 티어 광고.
앱은 도구만 제공하고 서버는 사용자가 준비 → 법적으로 깔끔하다.
**리스크**: App Store의 VPN 앱 심사 요건이 까다롭다(사업자 정보, 개인정보처리방침 등).

### 6.5 교육 콘텐츠 / 정보 상품

가장 안전하고 즉시 시작 가능한 경로.

- YouTube: "QUIC 프로토콜", "Go로 프록시 만들기"
- 기술 블로그 시리즈 → 애드센스 + 제휴
- 인프런/유데미 강의 (₩55,000~)
- 전자책: 셀프호스팅 인프라 구축 가이드
- VPS 제휴 링크 (건당 $25~35 수준)

6.1의 마케팅 채널로도 활용 가능 (콘텐츠 독자 → 패널 고객 전환).

### 6.6 오픈소스 + 스폰서십

GitHub Sponsors, Buy Me a Coffee, 듀얼 라이선스(오픈소스 무료 / 상업용 유료),
코어 무료 + 유료 애드온. 수익화 속도는 느리지만 평판 자산이 축적된다.

### 6.7 권장 로드맵

```
STEP 1 (1~2개월)  리스크 0, 검증
  - 관리 패널 MVP를 React + PHP로 제작, 오픈소스 공개
  - 개발 과정을 블로그/영상으로 기록
  → GitHub 스타 및 초기 사용자 확보

STEP 2 (3~6개월)  수익화 시작
  - Pro 기능 추가 (멀티노드, 알림, 화이트라벨)
  - 결제 연동 (Stripe/Paddle)
  → 구독 매출 발생

STEP 3 (6개월~)   확장
  - B2B 구축 서비스로 단가 상승
  - 데스크톱/모바일 앱으로 생태계 완성
  - 강의/전자책으로 부수입
```

**선정 이유**: 기존 스킬(React/PHP) 재사용 → 학습 비용 최소 /
법적 리스크가 낮은 것부터 순차 진행 / 오픈소스 공개로 마케팅 비용 절감 /
실패해도 포트폴리오로 남음.

---

## 7. 참고 링크

| 항목 | 링크 |
|---|---|
| 이 저장소 (포크) | https://github.com/bmshin94/hysteria |
| 원본 저장소 (업스트림) | https://github.com/apernet/hysteria |
| 공식 문서 | https://v2.hysteria.network/ |
| 공식 문서 (중국어) | https://v2.hysteria.network/zh/ |
| Hysteria 1.x (레거시) | https://v1.hysteria.network/ |
| 릴리스 | https://github.com/apernet/hysteria/releases |
| Discussions | https://github.com/apernet/hysteria/discussions |
| Telegram | https://t.me/hysteria_github |
| 서버 설치 스크립트 | https://get.hy2.sh/ |
| Mimic (UDP→TCP 위장) | https://github.com/hack3ric/mimic |
| QUIC (RFC 9000) | https://datatracker.ietf.org/doc/html/rfc9000 |
| QUIC Datagram (RFC 9221) | https://datatracker.ietf.org/doc/rfc9221/ |
| HTTP/3 (RFC 9114) | https://datatracker.ietf.org/doc/rfc9114/ |

### 저장소 내 문서

- [README.md](../README.md) — 프로젝트 개요
- [PROTOCOL.md](../PROTOCOL.md) — 프로토콜 스펙
- [LICENSE.md](../LICENSE.md) — MIT 라이선스
- [CHANGELOG.md](../CHANGELOG.md) — 변경 이력 (공식 사이트 링크)
