# Tailscale 저장소 전수조사 및 활용 분석 (한국어 정리)

> 이 문서는 `bmshin94/tailscale` 포크 저장소를 전수조사한 결과와,
> 그에 대한 Q&A·수익화 아이디어를 정리한 것입니다.

## 저장소 정보

| 항목 | 값 |
|---|---|
| 이 저장소 (포크) | https://github.com/bmshin94/tailscale |
| 원본 (업스트림) | https://github.com/tailscale/tailscale |
| 작업 브랜치 | `claude/serene-faraday-f84k2e` |
| 공식 사이트 | https://tailscale.com |
| 공식 API 문서 | https://tailscale.com/api |
| 다운로드 | https://tailscale.com/download |
| 패키지 저장소 | https://pkgs.tailscale.com |
| tsnet Go 문서 | https://pkg.go.dev/tailscale.com/tsnet |
| 오픈소스 정책 | https://tailscale.com/opensource/ |
| 이슈 트래커 | https://github.com/tailscale/tailscale/issues |

### 관련 Tailscale 저장소

- Android 앱 — https://github.com/tailscale/tailscale-android
- Synology 패키지 — https://github.com/tailscale/tailscale-synology
- QNAP 패키지 — https://github.com/tailscale/tailscale-qpkg
- Chocolatey 패키징 — https://github.com/tailscale/tailscale-chocolatey
- tsidp (OIDC IdP, 이 저장소에서 이전됨) — https://github.com/tailscale/tsidp
- Tailscale Go 포크 — https://github.com/tailscale/go

---

## 1. 전수조사 결과

### 한 줄 결론

**이것은 플러그인·스킬·MCP가 아니라, Tailscale이라는 VPN 제품의 본체 소스코드(Go)입니다.**

### 객관적 수치 (실측)

| 항목 | 값 |
|---|---|
| 주 언어 | Go — 2,442개 파일, **약 596,530줄** |
| 보조 언어 | TypeScript/React 54개(`.tsx` 37 + `.ts` 17), Swift 9개, Shell 30개 |
| 버전 (`VERSION.txt`) | **1.103.0** |
| 라이선스 | **BSD 3-Clause** (상업적 이용·수정·재배포 허용) |
| Go 요구 버전 | 1.27.1 |
| 최상위 디렉터리 | 67개 |
| 실행 바이너리 (`cmd/`) | **49개** |
| CI 워크플로우 | 25개 |

### 기술적 실체

WireGuard(최신 VPN 암호화 프로토콜) 위에 **제어 평면(control plane)** 을 씌운 제품.

- **기존 VPN**: 모든 트래픽이 중앙 VPN 서버 경유(허브 앤 스포크). 느리고 단일 장애점이며 포트포워딩·공인IP 필요.
- **Tailscale**: 기기끼리 **직접(P2P)** 암호화 터널. 중앙 서버는 "누가 누구와 연결 가능한지"와 공개키·주소만 알려주는 교환원 역할. 실제 데이터는 서버를 거치지 않음.

### 핵심 메커니즘 4가지 (소스 확인)

**① NAT 통과 — `net/`, `disco/`, `wgengine/magicsock/` (약 11만 줄)**

- `net/netcheck` — 내 네트워크의 NAT 유형 진단
- `net/stun` — 외부에서 보이는 공인 IP:포트 탐지
- `net/portmapper` — UPnP / NAT-PMP / PCP로 공유기에 자동 포트 매핑
- `disco/` — `TS💬`(0x54 53 f0 9f 92 ac) 매직넘버로 시작하는 자체 암호화 탐색 프로토콜. 양쪽이 동시에 패킷을 보내 홀펀칭

**② DERP 중계 — `derp/` (약 1.2만 줄)**

소스 주석 원문: *"Both sides between very aggressive NATs, firewalls, no IPv6, etc? Well, DERP. DERP is a last resort."*

직접 연결 실패 시 중계 서버 경유. **중계 서버도 내용을 볼 수 없음**(종단간 암호화 유지). `cmd/derper`로 자체 중계 서버 운영 가능.

**③ 노드 에이전트 — `ipn/` (약 6.6만 줄)**

`ipn/ipnlocal` 주석: *"the heart of the Tailscale node agent"*. 로그인 상태, 설정(Prefs), 프로필, netmap, DNS, 방화벽 규칙 전체 관리.

**④ 정책/보안 — `tka/`, `tailcfg/`**

- `tka/` (8,925줄) — **Tailnet Lock**. 제어 서버가 침해당해도 몰래 기기를 추가할 수 없도록 서명 체인을 요구하는 암호학적 장치
- `tailcfg/` — ACL 정책 데이터 모델

### 폴더 지도

#### 사용자가 직접 쓰는 것

| 경로 | 정체 |
|---|---|
| `cmd/tailscaled` | 데몬. 가상 네트워크 인터페이스 관리 |
| `cmd/tailscale` | CLI. `up`/`down`/`status`/`ip`/`ping`/`netcheck`/`ssh`/`serve`/`funnel`/`file`/`drive`/`cert`/`exit-node`/`switch`/`whois`/`whoami`/`tailnet-lock`/`dns`/`metrics`/`nc`/`service`/`update`/`bugreport`/`wait` 등 |
| `client/web` | 웹 UI — React + TypeScript + Tailwind + Vite (`.tsx` 37개) |
| `client/systray` | Linux 트레이 아이콘 앱 |

#### 개발자 / 서버 운영자용

| 경로 | 정체 |
|---|---|
| **`tsnet/`** (8,192줄) | 내 Go 프로그램에 Tailscale 노드 내장. gVisor 유저스페이스 TCP/IP 스택. root·데몬 불필요. 한 프로세스에 여러 노드 가능 |
| `client/local` | **LocalAPI** 클라이언트 — 로컬 데몬 제어, `WhoIs` 신원 확인 |
| `client/tailscale` | 제어 평면 **REST API** 클라이언트 (deprecated → `tailscale.com/client/tailscale/v2`는 별도 모듈) |
| `k8s-operator/` + `cmd/k8s-operator` | 쿠버네티스 오퍼레이터 (1.8만 줄). Service/Ingress를 tailnet에 노출 |
| `cmd/containerboot` | 컨테이너 래퍼. `TS_AUTHKEY`, `TS_HOSTNAME`, `TS_ROUTES`, `TS_CLIENT_ID`/`TS_CLIENT_SECRET`(OAuth), `TS_ID_TOKEN`/`TS_AUDIENCE`(워크로드 아이덴티티) 등 환경변수 기반 |
| `cmd/derper` | 자체 DERP 중계 서버 |
| `cmd/tsconnect` | 브라우저에서 동작하는 Tailscale (WASM) |
| `cmd/sniproxy` | tailnet → 인터넷 아웃바운드 SNI 프록시 |
| `cmd/natc` | 도메인별 트래픽을 특정 노드로 보내는 NAT 커넥터 (개발 중) |
| `cmd/tsidp` | OIDC 신원 공급자 — **이 저장소에서 개발 중단, https://github.com/tailscale/tsidp 로 이전** |
| `cmd/proxy-to-grafana`, `cmd/nginx-auth` | tailnet 신원으로 Grafana / nginx 인증 |
| `cmd/speedtest`, `cmd/stunc`, `cmd/stund` | 속도 측정 · STUN 도구 |
| `prober/`, `cmd/derpprobe` | 모니터링 / 프로빙 |

#### 기능 모듈 — `feature/` (48개)

`feature/README.md` 설계 철학 요약: *"클라이언트가 너무 커졌다. 몇 달러짜리 IoT 칩은 Taildrop, WebDAV, ACME, SSH가 필요 없다."*

→ `ts_omit_<기능>` 빌드 태그로 기능을 제외하고 컴파일 가능. 대상: `ssh`, `taildrop`, `drive`, `acme`, `tailnetlock`, `portlist`, `wakeonlan`, `tpm`, `syspolicy`, `relayserver`, `appconnectors`, `captiveportal`, `netlog` 등.

#### 기타

`tstest/`(3.5만 줄 — `natlab` 가상 네트워크 시뮬레이터 포함), `util/`(4.9만 줄), `gokrazy/`(Raspberry Pi 어플라이언스 이미지), `docs/windows/policy/`(Windows ADMX 그룹정책), `release/`(패키징), `tempfork/`(2.6만 줄 — 외부 라이브러리 임시 포크).

### 실전 활용 시나리오

1. 집 NAS / 홈랩 외부 접속 — 포트포워딩·DDNS·공인IP 불필요
2. 원격근무 — 사내 개발 서버 / DB 접속, P2P라서 기존 VPN보다 빠름
3. 파일 전송 — `tailscale file cp`(Taildrop), `tailscale drive`(WebDAV)
4. **Exit Node** — 특정 기기를 거쳐 인터넷. 해외에서 국내 IP, 공공 WiFi 보호
5. **Subnet Router** — 한 대만 설치해 프린터·IoT·레거시 장비가 속한 전체 네트워크 노출
6. **Tailscale SSH** — SSH 키 관리 폐지, ACL 중앙 관리, 세션 녹화(`sessionrecording/`)
7. **Serve / Funnel** — `localhost:3000`을 tailnet HTTPS 또는 인터넷에 공개(ngrok 대체, TLS 자동)
8. CI/CD — GitHub Actions 러너를 ephemeral 노드로 임시 연결
9. 쿠버네티스 — 클러스터 서비스 노출, kubectl을 tailnet 신원으로 인증
10. 앱 내장(tsnet) — 내 서비스를 "tailnet 전용"으로. 인터넷에 포트가 열리지 않음

### 주의할 점

- **제어 평면(계정·ACL 관리 서버)은 오픈소스가 아님.** 이 저장소는 클라이언트 + 중계 서버뿐. 완전 자체호스팅은 서드파티 `headscale` 필요
- iOS / Android / macOS GUI 래퍼는 이 저장소에 없음
- 포크(1.103.0)이므로 업스트림 최신과 차이 가능

---

## 2. 쉬운 설명 — 동작 원리

### 비유: 전 세계 어디서든 작동하는 가상 랜선

**예전 방식의 고통** — 집 NAS에 외부 접속하려면: 공유기 포트포워딩 → 공인IP 없으면 포기 → IP 변동 때문에 DDNS → 포트가 인터넷에 열려 무차별 공격 로그 → 불안해서 VPN 직접 구축 → 주말 소멸.

**Tailscale** — 노트북과 NAS에 각각 앱 설치 후 같은 계정으로 로그인. 끝. 이제 `nas:5000`이 전 세계 어디서든 열립니다. **포트는 인터넷에 열려 있지 않습니다.**

### 우체국 비유로 본 5단계

- 등장인물: **내 기기들** = 집집마다 있는 사람 / **제어 서버** = 전화 교환원 / **DERP** = 우체국(최후의 수단)

**1단계 — 열쇠 만들기.** 기기가 자기만의 공개키/비밀키 쌍을 생성. **비밀키는 그 기기를 절대 떠나지 않음.** 공개키만 교환원에게 전달. → 교환원(Tailscale 회사)은 당신의 암호 열쇠를 갖고 있지 않으므로, 교환원이 털려도 데이터를 볼 수 없습니다.

**2단계 — 명부 수신.** 교환원이 "내 계정의 기기 목록 + 공개키 + 현재 위치(IP:포트)"를 모든 기기에 배포. 소스에서는 이를 **netmap** 이라 부름.

**3단계 — 직접 연결(홀펀칭).** NAT는 "나가는 건 되고 들어오는 건 안 되는" 일방통행 문. Tailscale의 트릭은 **양쪽이 동시에 서로에게 전화 걸기**:

```
노트북 ──(나는 1.2.3.4:51820에서 보임)──→ 교환원
NAS    ──(나는 5.6.7.8:41641에서 보임)──→ 교환원
교환원 ──(상대 주소 전달)──────────────→ 양쪽

같은 순간에:
노트북 ─"안녕?"→ 5.6.7.8:41641
NAS    ─"안녕?"→ 1.2.3.4:51820
```

양쪽 공유기가 "우리 쪽에서 먼저 나갔으니 답장은 받아줘야지" 하며 길을 열어 **구멍이 뚫립니다.** 이제 직통. 이 "안녕?" 패킷 포맷이 `disco/disco.go`에 정의되어 있고, 앞 6바이트가 `TS💬`입니다.

**4단계 — 실패 시 우체국 경유.** 대칭형 NAT 등으로 홀펀칭이 안 되면 DERP 중계. 단 **우체국은 봉투를 열 수 없습니다** — 내용은 이미 양쪽 기기의 열쇠로 암호화됨.

**5단계 — 경로 전환.** DERP로 시작했다가 백그라운드에서 계속 직통을 시도해 성공하면 **끊김 없이** 전환. WiFi→LTE 전환에도 세션 유지. 이것이 `wgengine/magicsock`("매직소켓")의 역할.

### 주요 기능 생활 언어 설명

- **Exit Node** — 인터넷 트래픽 전체를 지정 기기 경유. 해외에서 국내 IP 쓰기, 카페 WiFi에서 트래픽 보호
- **Subnet Router** — 프린터·IP카메라·구형 서버에는 Tailscale을 설치할 수 없음. 같은 네트워크의 기기 한 대가 "내가 192.168.1.0/24 담당"이라 광고해 문지기 역할
- **Serve / Funnel** — `tailscale serve 3000`은 tailnet 내부에 HTTPS 공개(**TLS 인증서 자동 발급**, `feature/acme`). `tailscale funnel 3000`은 인터넷 전체 공개(443/8443/10000 포트)
- **Taildrop / Taildrive** — `tailscale file cp 사진.jpg 폰:`으로 OS 무관 직접 전송(클라우드 미경유). `tailscale drive`로 폴더를 WebDAV 공유
- **Tailscale SSH** — SSH 키 관리 폐지. ACL에 `"src": ["user@example.com"], "dst": ["tag:prod"]` 한 줄. 사람 퇴사 시 계정 하나 삭제로 모든 서버 접근 즉시 차단. 세션 녹화 가능
- **Tailnet Lock** — "Tailscale 회사 서버가 해킹당해 공격자가 몰래 기기를 추가하면?"을 막음. 기기 추가에 내 기기들의 암호학적 서명을 요구. `tka/`가 구현

### tsnet — 개발자에게 가장 강력한 부분

```go
s := &tsnet.Server{Hostname: "my-service", AuthKey: os.Getenv("TS_AUTHKEY")}
ln, _ := s.Listen("tcp", ":80")
http.Serve(ln, myHandler)
```

- root 권한 불필요, 데몬 설치 불필요
- 이 서비스는 **인터넷에서 보이지 않음**. tailnet 내부에만 존재
- 방화벽 규칙 없음, 열린 포트 없음
- `WhoIs(요청자)` 한 줄로 접속자 이메일·태그 확인 → **로그인 기능을 만들 필요가 없음**

일반적으로 사내 대시보드를 만들면 로그인·세션·비번재설정·2FA를 모두 구현해야 하지만, tsnet이면 그 코드가 **0줄**입니다. 네트워크 자체가 신원입니다.

### 폴더를 집으로 비유

| 폴더 | 집의 어느 부분 |
|---|---|
| `cmd/` (10만 줄) | 현관문들 — 49개 실행 프로그램 입구 |
| `ipn/` (6.6만 줄) | 거실/두뇌 — 상태·설정 중앙 관리 |
| `net/` (7.2만 줄) | 배관·전기 — NAT 통과, DNS, 소켓, 방화벽 |
| `wgengine/` (4만 줄) | 엔진룸 — WireGuard 터널 구동 |
| `derp/` | 우체국 — 중계 서버 |
| `tsnet/` | 셋방 — 남의 프로그램에 세 들어 사는 모드 |
| `feature/` (48개) | 옵션 가구 — 빼고 지을 수 있는 기능 |
| `tka/` | 금고 — Tailnet Lock 암호학 |
| `k8s-operator/` | 별관 — 쿠버네티스 전용 |
| `tstest/` (3.5만 줄) | 품질검사실 — 가상 네트워크 시뮬레이터 포함 |
| `util/` (4.9만 줄) | 공구함 |

---

## 3. Q&A

### Q1. 설치 및 사용법

#### 그냥 쓰려면 (99%의 경우)

```bash
# 리눅스
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up          # 출력 URL을 브라우저로 열어 로그인
tailscale ip -4            # 내 100.x.x.x 주소
tailscale status           # 기기 목록
```

macOS / Windows / iOS / Android는 https://tailscale.com/download 에서 설치 후 로그인.
(macOS는 App Store판과 Standalone판이 달라 `tailscale configure sysext` 등 전용 CLI가 있음)

```bash
# Docker
docker run -d --name tailscaled \
  -v /var/lib:/var/lib -v /dev/net/tun:/dev/net/tun \
  --network=host --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -e TS_AUTHKEY=tskey-auth-xxxxx \
  -e TS_ROUTES=192.168.1.0/24 \
  tailscale/tailscale
```
지원 환경변수 전체는 `cmd/containerboot/main.go` 상단 주석에 문서화되어 있습니다.

```bash
# 쿠버네티스
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm install tailscale-operator tailscale/tailscale-operator ...
```
이후 Service에 `loadBalancerClass: tailscale`, Ingress에 `ingressClassName: tailscale`.

#### 이 소스를 직접 빌드

```bash
# 전제: Go 1.27 이상
go install tailscale.com/cmd/tailscale{,d}

# 배포용 (버전·커밋 정보 삽입)
./build_dist.sh tailscale.com/cmd/tailscale
./build_dist.sh tailscale.com/cmd/tailscaled
```

이 포크에서:
```bash
go build ./cmd/tailscale ./cmd/tailscaled
make help        # 사용 가능한 모든 타겟
make check       # staticcheck + vet + depaware + 크로스컴파일 검증
make buildwasm   # 브라우저용 WASM
go test ./ipn/...
```

기능 제외 경량 빌드:
```bash
go build -tags ts_omit_ssh,ts_omit_taildrop,ts_omit_drive,ts_omit_acme ./cmd/tailscaled
```

#### 자주 쓰는 명령

```bash
tailscale up / down / status / ip / ping / netcheck
tailscale ssh user@hostname                      # SSH
tailscale file cp ./file.zip phone:              # Taildrop
tailscale drive share mydocs /home/me/doc        # Taildrive
tailscale serve 3000                             # tailnet에 HTTPS 공개
tailscale funnel 3000                            # 인터넷에 공개
tailscale set --exit-node=myhome                 # Exit Node 사용
tailscale up --advertise-routes=192.168.1.0/24   # Subnet Router
tailscale switch <account>                       # 계정 전환
tailscale cert my-host.tail-xxxx.ts.net          # TLS 인증서
tailscale whois 100.x.x.x                        # 이 IP가 누구인지
tailscale bugreport                              # 진단 ID
```

### Q2. 플러그인? 스킬? MCP?

**셋 다 아닙니다.**

| 분류 | 맞나? |
|---|---|
| 플러그인 (다른 앱 확장) | ❌ |
| 스킬 (Claude에게 작업법을 알려주는 지침 패키지) | ❌ |
| MCP 서버 (AI에 도구/데이터 노출 프로토콜) | ❌ |
| **독립 시스템 소프트웨어** (OS 네트워크 데몬 + CLI) | ✅ |
| **Go 라이브러리** (`tsnet`) | ✅ |

**혼동 원인:** 저장소에 `CLAUDE.md`가 있음. 그러나 이는 Claude Code 작업용으로 Curator-Agent가 자동 생성한 안내 문서이며(파일 하단에 명기), Tailscale 본체와 무관합니다. git 로그의 `PR #1: docs: add CLAUDE.md project guide`로 나중에 추가된 것이 확인됩니다.

**세 가지 얼굴:**
1. 시스템 데몬 + CLI (`cmd/tailscaled`, `cmd/tailscale`)
2. 임베더블 Go 라이브러리 (`tsnet`)
3. SDK / API 클라이언트 (`client/local` = LocalAPI, `client/tailscale` = 제어 평면 REST)

**MCP와 결합은 가능:** `client/local` + `client/tailscale`을 래핑해 `list_devices`, `get_status`, `authorize_device` 같은 MCP tool을 노출하는 **"Tailscale 제어 MCP 서버"를 직접 만들 수 있습니다.**

### Q3. API 토큰이 필요한가 — 6가지 인증 수단

| 하려는 일 | 필요한 것 |
|---|---|
| 내 노트북/폰에 VPN 설치 | **없음** (브라우저 OAuth 로그인) |
| 서버 / Docker 자동 등록 | **Auth Key** (`tskey-auth-...`) |
| 장기 운영 인프라 / CI | **OAuth Client** (`tskey-client-...`) — 권장 |
| 클라우드·K8s에서 비밀값 없이 | **Workload Identity Federation** |
| 기기·ACL을 코드로 관리 | **API Token** (`tskey-api-...`) 또는 OAuth Client |
| 내 PC 데몬 상태 조회 | **없음** (로컬 유닉스 소켓 권한만) |

**① 브라우저 로그인** — `tailscale up`. 토큰 개념이 등장하지 않음.

**② Auth Key** — 사람이 브라우저를 열 수 없는 환경(서버·Docker·CI·키오스크)용. 관리 콘솔 → Settings → Keys에서 발급. 종류: Reusable / **Ephemeral**(기기 종료 시 자동 삭제, CI 최적) / Pre-approved / Tagged. **최대 90일.**
```bash
sudo tailscale up --authkey=tskey-auth-xxxxx
export TS_AUTHKEY=tskey-auth-xxxxx   # tsnet, containerboot가 읽음
```

**③ OAuth Client** — **만료되지 않음**. 필요할 때 짧은 수명의 auth key를 자동 발급. 구현: `cmd/get-authkey/`, `feature/oauthkey/`.
```bash
export TS_CLIENT_ID=xxxxx
export TS_CLIENT_SECRET=tskey-client-xxxxx
```

**④ Workload Identity Federation** — 소스에서 발견한 가장 진보된 방식 (`wif/`, `feature/identityfederation/`). AWS/GCP/Azure/K8s가 발급하는 ID 토큰을 제어 서버에 제시해 교환. **비밀값을 코드·환경변수에 둘 필요 없음.**
```go
import _ "tailscale.com/feature/identityfederation"
// Server.ClientID + Server.IDToken 또는 Server.Audience
```
> AWS SDK 등 무거운 의존성 때문에 **기본으로 링크되지 않음.** 위처럼 명시적 import 필요.

**⑤ 제어 평면 REST API**
```bash
curl -u "tskey-api-xxxxx:" https://api.tailscale.com/api/v2/tailnet/-/devices
```
공식 문서: https://tailscale.com/api (`api.md`에 이 링크만 있음)

**⑥ LocalAPI** — `/var/run/tailscale/tailscaled.sock` 유닉스 소켓. 토큰 없음, 대신 root 또는 적절한 그룹 권한 필요.

**tsnet 인증 우선순위** (README 명시): `Server.AuthKey` → `TS_AUTHKEY` → `TS_AUTH_KEY` → OAuth client secret → Workload identity federation → 대화형 로그인 URL 출력. 이미 등록된 노드는 `TSNET_FORCE_LOGIN=1`이 없으면 auth key를 무시.

### Q4. AI 에이전트 구축에 도움이 되는가 → **매우 큰 도움**

**① 에이전트 샌드박스 네트워크 격리** — "에이전트가 인터넷에서 무슨 짓을 할지 어떻게 통제하나"에 대한 답. 에이전트 컨테이너를 tailnet에만 연결(인터넷 직결 차단), ACL로 "사내 DB와 사내 git만 접속 가능"을 선언적으로 정의, Exit Node + ACL로 아웃바운드 화이트리스트, `cmd/sniproxy`로 허용 도메인만 아웃바운드. Docker 네트워크 설정보다 상위 — **ACL이 코드(JSON)이고, 감사 로그가 남고, 기기가 아니라 신원 단위로 관리**됩니다.

**② MCP 서버를 안전하게 노출 — 킬러 유즈케이스**

현재 MCP 생태계의 실질적 문제: 원격 MCP 서버는 인증(OAuth 플로우, 토큰 저장·갱신·취소)을 직접 구현해야 함. tsnet + `WhoIs()`로 이 코드가 전부 사라집니다.

```go
s := &tsnet.Server{Hostname: "my-mcp-server", AuthKey: os.Getenv("TS_AUTHKEY")}
ln, _ := s.Listen("tcp", ":443")
lc, _ := s.LocalClient()

http.Serve(ln, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    who, err := lc.WhoIs(r.Context(), r.RemoteAddr)
    if err != nil { http.Error(w, "unauthorized", 401); return }
    // who.UserProfile.LoginName → 접속자 이메일
    // who.Node.Tags            → ["tag:agent-prod"]
    handleMCP(w, r, who)
}))
```

- 인터넷에 포트가 열리지 않음 → 스캔·무차별 공격 불가
- 로그인 화면·세션·토큰 저장 코드 **0줄**
- 신원이 WireGuard 키로 암호학적 보장
- 접근 취소 = 관리 콘솔에서 기기/사용자 삭제 → 즉시 반영

**③ 멀티 에이전트 메시** — 한 바이너리에 여러 독립 노드:

```go
for _, name := range []string{"researcher", "coder", "reviewer"} {
    srv := &tsnet.Server{
        Hostname: name,
        Dir: filepath.Join(baseDir, name),
        AuthKey: os.Getenv("TS_AUTHKEY"),
        Ephemeral: true,   // 종료 시 자동 정리
    }
    srv.Start()
}
```

각 에이전트가 **자기만의 네트워크 신원**을 가짐 → ACL로 "researcher는 읽기만, coder는 git 쓰기, reviewer는 CI만" 같은 **에이전트별 최소권한을 네트워크 레벨에서 강제**. 에이전트 간 통신이 자동으로 상호 인증·암호화(mTLS 직접 구현 불필요). 전체 접근 감사 로그. `Ephemeral: true`로 유령 노드 미축적.

**④ 분산 / 하이브리드** — 로컬 GPU 추론 + 클라우드 오케스트레이션을 tailnet으로 연결(공인IP 불필요). 사내 데이터는 밖으로 나가지 않고 에이전트가 사내로 들어옴(데이터 주권·컴플라이언스). 에지 디바이스도 같은 메시에(`gokrazy/`에 Pi 어플라이언스 빌드 존재).

**⑤ 개발 루프** — `tailscale funnel`로 로컬 에이전트가 즉시 외부 웹훅 수신(Slack/GitHub 웹훅 테스트, ngrok 대체). `tailscale serve`로 팀 데모 공유.

**한계 (솔직하게)**
- **Go 중심.** tsnet은 Go 라이브러리. Python 에이전트는 사이드카 컨테이너(`cmd/containerboot`)나 호스트 데몬 방식 필요
- **제어 평면 의존.** Tailscale 서버 다운 시 신규 연결 불가(기존 연결 유지). 완전 자립은 `headscale`
- **네트워크 레이어 보안일 뿐.** "DB 접속 가능"은 통제하지만 "접속해서 DROP TABLE"은 막지 못함. 애플리케이션 가드레일은 별도 필요

### Q5. 수익화 → 4장 참조

**핵심 전제:** BSD-3 라이선스는 상업적 이용을 허용 — 코드를 상업 제품에 넣어 판매해도 **합법**(저작권 고지·라이선스 사본 유지, Tailscale 상표 사용 불가).

### Q6. React나 PHP로 만들 수 있는가

#### (A) Tailscale 자체를 React/PHP로 재구현 → **사실상 불가능**

1. **커널 네트워크 스택**을 다룸 — TUN 인터페이스 생성, 라우팅 테이블, 방화벽(nftables/iptables/WFP). JS/PHP는 이 레이어 접근 불가
2. **고성능 UDP 처리** — 초당 수십만 패킷 암복호화. GC 있는 스크립트 언어로 불가
3. WireGuard 프로토콜 + Noise 핸드셰이크 구현 필요
4. **60만 줄** — 수년간 수십 명이 Go로 작성
5. **크로스플랫폼 네이티브** — Linux/Windows/macOS/iOS/Android/BSD/Plan9/WASM 각각의 네트워크 API (`.syso`, Swift, Windows ADMX까지 존재)

#### (B) Tailscale을 활용하는 것을 React/PHP로 → **얼마든지 가능** (현실적인 답)

**React** — 이 저장소 안에 **이미 React 코드가 있습니다**(`client/web/`: React + TypeScript + Tailwind + Vite + Yarn). Tailscale 팀 자신이 관리 UI를 React로 만들었습니다.

| 만들 것 | 방법 |
|---|---|
| 커스텀 Tailnet 관리 대시보드 | React → REST API v2 |
| 네트워크 토폴로지 시각화 | React + D3 / Cytoscape |
| ACL 비주얼 에디터 | React + Monaco Editor + ACL API |
| 기기 인벤토리 / 만료 알림 | React + `devices:read` |
| 접속 로그 분석 뷰어 | React + Network Flow Logs API |
| 온보딩 자동화 포털 | React + 백엔드가 Auth Key 발급 |
| 기존 웹앱에 tailnet SSO | 백엔드 `WhoIs` → React가 사용자 표시 |

⚠️ **React(브라우저 JS)는 직접 VPN 터널을 만들 수 없습니다.** 반드시 백엔드를 거쳐 API 호출. **API 토큰을 브라우저에 두면 안 됩니다.**
단, 예외: `cmd/tsconnect`는 Tailscale을 **WASM으로 컴파일**해 브라우저에서 실제로 tailnet에 접속합니다(`make buildwasm` 타겟 존재).

**PHP** — REST API 클라이언트로는 완벽히 가능:

```php
<?php
$ch = curl_init("https://api.tailscale.com/api/v2/tailnet/-/devices");
curl_setopt($ch, CURLOPT_USERPWD, "tskey-api-xxxxx:");
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$devices = json_decode(curl_exec($ch), true);
foreach ($devices['devices'] as $d) {
    echo "{$d['hostname']} — {$d['addresses'][0]} — 만료: {$d['expires']}\n";
}
```

| 만들 것 | 가능? |
|---|---|
| Laravel / Symfony 관리 패널 | ✅ |
| WordPress 플러그인 (기기 현황 위젯) | ✅ |
| 자산관리 + 만료 알림 cron | ✅ |
| 고객 셀프서비스 포털 | ✅ |
| PHP 웹앱에 tailnet 신원 인증 | ✅ nginx/Caddy + `cmd/nginx-auth` |
| PHP로 VPN 터널 직접 생성 | ❌ |

**PHP + `cmd/nginx-auth` 조합이 특히 실용적:** nginx `auth_request`로 "요청자가 tailnet의 누구인지"를 확인해 헤더로 PHP에 전달. **기존 PHP 레거시 앱에 로그인 코드 한 줄 없이 SSO를 붙일 수 있습니다.**

**추천 아키텍처**

```
┌─────────────────────────────────────────┐
│  React 프론트엔드 (대시보드 UI)          │
└──────────────┬──────────────────────────┘
               │ 내부 REST
┌──────────────▼──────────────────────────┐
│  백엔드 (Go 권장 / PHP·Node 가능)        │
│  · API 토큰 보관 (프론트 노출 금지)      │
│  · tailscale REST API v2 호출            │
│  · (Go면) tsnet 임베드 + WhoIs 인증      │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  Tailscale (데몬 또는 tsnet)             │
└─────────────────────────────────────────┘
```

Go를 쓸 수 있다면 Go 백엔드를 강력 추천 — tsnet 임베드로 백엔드 자체가 tailnet 노드가 되고 인증 코드가 사라집니다. PHP는 이 이점을 누릴 수 없습니다(REST API만).

### Q7. 유튜브 강의 영상 제작 가능성 → **매우 적합**

| 요소 | 평가 |
|---|---|
| 결과가 눈에 보임 | ⭐⭐⭐⭐⭐ "밖에서 집 NAS 접속 성공" |
| 검색 수요 | ⭐⭐⭐⭐ "포트포워딩", "NAS 외부접속", "홈랩", "ngrok 대체" |
| 진입 난이도 | ⭐⭐⭐⭐⭐ 설치 2분, 따라하다 포기할 확률 낮음 |
| 한국어 콘텐츠 공백 | ⭐⭐⭐⭐⭐ 영어권엔 많으나 **한국어 심화 콘텐츠 거의 없음** |
| 깊이 확장성 | ⭐⭐⭐⭐⭐ 입문 ~ 커널 네트워킹 |
| 제작 비용 | ⭐⭐⭐⭐⭐ 무료 요금제로 전부 촬영 가능 |

#### 12편 시리즈 기획안

**초급 (조회수 담당)**
1. 「포트포워딩 없이 집 NAS에 접속하는 법」 (8분) — 공유기 설정 화면을 보여주고 "이거 다 필요 없습니다" → 2분 만에 성공
2. 「공공 WiFi에서 안전하게 — Exit Node」 (10분) — 패킷 캡처로 암호화 전/후 비교
3. 「ngrok 월 1만원 그만 — Tailscale Funnel」 (9분) — 한 줄로 HTTPS 공개 URL, 인증서 자동 발급
4. 「AirDrop인데 안드로이드-윈도우도 됨 — Taildrop」 (7분)

**중급**
5. **「NAT 홀펀칭 완전정복」 (18분)** ⭐ 대표작 추천 — 화이트보드 애니메이션 + 실제 `tailscale netcheck`/`ping` 출력. 소스 `disco/disco.go`의 `TS💬` 매직넘버 노출로 신뢰도 급상승
6. 「Subnet Router — 프린터와 IoT를 VPN에 넣기」 (12분)
7. 「SSH 키 관리 폐지 — Tailscale SSH + ACL」 (15분) — 사용자 하나 삭제로 모든 서버 접근이 끊기는 장면이 강력
8. 「Docker / K8s에 Tailscale 붙이기」 (16분)

**고급 (전환율 담당)**
9. **「tsnet: 내 Go 앱을 VPN 노드로 — 로그인 코드 0줄」 (20분)** ⭐⭐ 가장 차별화. 한국어 콘텐츠 사실상 없음
10. 「MCP 서버를 안전하게 호스팅하기」 (22분) — AI 트렌드 + 실무 보안, 현재 수요 급증
11. 「React 대시보드 + Tailscale API 만들기」 (25분)
12. 「60만 줄 Go 코드 읽기 — Tailscale 아키텍처 투어」 (30분)

#### 제작 시 주의사항

**법적 / 윤리적**
- ✅ BSD-3 라이선스 → 코드를 화면에 보여주고 설명하는 것 전부 OK
- ⚠️ **상표** — Tailscale 로고/이름을 채널명·썸네일 메인에 사용 금지. "Tailscale 사용법" 같은 서술적 사용은 가능하나 "공식 강의"처럼 인증을 암시하면 안 됨
- ⚠️ **화면 가리기** — Auth Key, API Token, `100.x.x.x` 주소, 이메일, tailnet 도메인(`xxx.ts.net`)은 **반드시 블러**. 촬영용 더미 계정 권장
- ⚠️ **"VPN" 단어 주의** — 지역 우회/저작권 회피 맥락으로 설명하면 수익창출 제한 위험. **"사설 네트워크", "원격 접속", "홈랩"** 프레임 사용

**기술적**
- 버전이 빠르게 올라감(이 포크 1.103.0) → 설명란에 촬영 시점 버전 명기, 고정 댓글로 업데이트
- 터미널 16pt 이상, 다크 테마 (모바일 시청자)
- **에러 상황도 보여줄 것** — "방화벽 때문에 DERP로 떨어지는 경우" 등. 완벽하게만 되는 영상은 신뢰를 잃음

**수익 모델 연결** — 애드센스 + 멤버십 / 유료 강의(인프런·클래스101) 깔때기 / 제휴는 Tailscale 자체보다 **NAS 제조사·VPS 호스팅·홈랩 장비** / 템플릿·소스코드 묶음 판매

---

## 4. 수익화 아이디어 상세

### 전제: 라이선스와 상표

| 항목 | 내용 |
|---|---|
| 라이선스 | **BSD 3-Clause** — 상업적 이용, 수정, 재배포, 비공개 소스화 모두 허용 |
| 의무 | 저작권 고지 + 라이선스 사본 유지, "Tailscale 이름으로 보증/홍보하지 않음" |
| 상표 | **Tailscale 이름·로고는 별개 권리** — 제품명에 사용 불가 |

⚠️ **ToS 주의:** Tailscale의 **호스팅 서비스 재판매**(계정 공유·리셀)는 약관 위반 가능성. 반면 **코드 이용**과 **컨설팅·구축 서비스**는 문제없음. 이 구분이 중요합니다.

### 티어 1 — 자본 없이 당장 시작 가능

#### 1. 홈랩 / 중소기업 네트워크 구축 서비스 ⭐ 가장 현실적

- **무엇을**: "포트포워딩 없는 외부 접속 환경 구축" 대행
- **타겟**: 중소기업(사내 서버·CCTV·ERP 외부접속), 병원·학원, 소규모 쇼핑몰, 홈랩 입문자
- **왜 돈이 되나**: 전통적 VPN 구축은 수백만 원 + 유지보수 부담. Tailscale은 반나절. **지식 차익**
- **가격**: 기본 구축(5대 이하) 30~60만 원 / 중형(Subnet Router + ACL 설계 + 문서화) 150~400만 원 / **월 유지보수 10~30만 원 ← 핵심(반복 수익)**
- **필요**: Tailscale 숙련도만. 서버·자본 불필요
- **리스크**: 영업 채널. 블로그·유튜브(아이디어 2번)로 인바운드 생성이 정답

#### 2. 콘텐츠 + 유료 강의

| 경로 | 예상 |
|---|---|
| 유튜브 애드센스 | 월 10~100만 원 |
| 인프런·클래스101 | 강의당 3~8만 원 × 수강생 |
| 전자책 | 1.5~3만 원 |
| 멤버십·패트리온 | 월 5천~2만 원 × 구독자 |
| 템플릿(ACL 레시피, tsnet 보일러플레이트) | 2~10만 원 |

- **차별화**: 한국어로 **tsnet과 MCP 호스팅**을 다루는 콘텐츠가 사실상 없음. 초급은 경쟁 있으나 이쪽은 공백
- **현실**: 6개월~1년의 꾸준한 제작 필요. 빠른 돈은 아님. 다만 **1번의 영업 채널**로서의 가치가 콘텐츠 자체 수익보다 클 수 있음

#### 3. Tailscale 관리 대시보드 (SaaS) — React/PHP 활용

공식 콘솔의 약점: 기기 **만료 임박 알림** 미흡(키 만료 시 갑자기 연결 끊김), ACL을 JSON으로 수동 작성(GUI 없음), **여러 tailnet 통합 뷰 없음**(MSP에 치명적), 비용·사용량 리포트 빈약.

- **기능**: 만료 D-7 Slack/이메일 알림 / ACL 비주얼 에디터 / 토폴로지 그래프(직통 vs DERP) / 멀티 tailnet 통합 뷰(MSP) / 온보딩 포털 / 감사 로그 아카이빙·검색
- **가격**: 월 $15~50 / tailnet, MSP 플랜 월 $200~500
- **리스크**: ⚠️ Tailscale이 직접 추가하면 사업 소멸 → **MSP 멀티테넌시**처럼 본가가 당장 안 할 영역에 집중. 고객 API 토큰을 대신 보관 → **보안 설계가 사업의 생명**

### 티어 2 — 개발 역량 필요, 수익 잠재력 큼

#### 4. tsnet 기반 "제로 설정 사내 SaaS" ⭐⭐ 가장 유망

**핵심 통찰**: tsnet을 쓰면 **로그인 기능 구현이 불필요**. 일반적으로 회원가입·로그인·세션·비번재설정·2FA·SSO·권한관리는 전체 개발의 **30~40%** 를 차지합니다. tsnet이면 그 코드가 0줄이고, 더 강한 보안(인터넷에 포트 미개방)을 얻습니다.

| 제품 | 설명 | 가격 |
|---|---|---|
| 사내 비밀 관리자 | 인터넷 미노출 Vault 경량판 | 월 $10/인 |
| tailnet 전용 Git/위키 | 유출 경로 자체가 없음 | 월 $8/인 |
| 에어갭 CI/CD 러너 | ephemeral 노드로 자동 등록/삭제 | 월 $50~200 |
| DB 접근 게이트웨이 | 직접 접속 금지, 신원 경유 + 쿼리 감사 | 월 $100~500 |
| 원격 지원 도구 | TeamViewer 대체 | 월 $15/인 |
| 의료·법률·금융 폐쇄망 앱 | 컴플라이언스 요구 강함 | 연 수천만 원 |

**왜 유망한가**: "보안은 필요하나 SSO 구축 예산은 없는" 중소기업이 거대 시장. 규제 업종(의료·법률·금융)은 **"데이터가 인터넷에 노출되지 않음"을 증명**해야 하고, 이 아키텍처가 구조적으로 보장 → **단가를 높게 받을 수 있음**.

**필요 역량**: Go

#### 5. AI 에이전트 샌드박스 플랫폼 ⭐⭐ 타이밍이 좋음

- **문제**: 에이전트에게 코드 실행 권한을 주는 건 보편화됐으나 **"에이전트의 네트워크 행위 통제"** 는 미해결
- **해결**: Tailscale ACL + Exit Node + `cmd/sniproxy` + tsnet 멀티노드
- **제품**: 에이전트별 독립 신원 / 선언적 아웃바운드 화이트리스트 / 전체 접근 감사 로그 / 사내 전용 에이전트(데이터 주권) / 컴플라이언스 리포트 자동 생성
- **타겟**: 에이전트를 도입하려는데 보안팀이 막는 기업 — **실제로 일어나는 병목**
- **가격**: 월 $200~2,000 / 팀, 엔터프라이즈 연 계약
- **왜 지금**: 대부분이 프롬프트 가드레일과 도구 권한에 집중 중. **"네트워크 레벨 격리" 각도는 아직 비어 있음**
- **리스크**: 시장 형성 중. 너무 일찍 들어가면 고객이 없음 → 1번(컨설팅)과 병행해 현금흐름 확보

#### 6. MCP 서버 호스팅 서비스 ⭐ 틈새지만 수요 명확

- **문제**: 원격 MCP 서버는 인증을 직접 구현해야 하고, 인터넷 노출은 공격 표면
- **제품**: tsnet 기반 → 인터넷에 포트 0개 / `WhoIs()` 기반 사용자별 권한 / 호출 로그·감사 / 팀 공유(ACL) / 검증된 MCP 카탈로그 마켓플레이스
- **가격**: 월 $20~100 / 팀, 마켓플레이스 수수료 20~30%
- **리스크**: MCP 표준이 움직이는 중. 호스팅 레이어는 플랫폼이 흡수할 가능성 → **빠르게 진입, 빠르게 판단**

### 티어 3 — 장기, 자본·조직 필요

#### 7. 수직 산업 특화 패키지

| 산업 | 제품 | 왜 돈이 되나 |
|---|---|---|
| 의료 | 병원 PACS/EMR 원격접속 | 규제 컴플라이언스 → 단가 높고 가격 저항 적음 |
| 제조/스마트팩토리 | OT 네트워크 원격 모니터링 | 공장 설비는 인터넷 노출 불가. Subnet Router가 완벽히 맞음 |
| 리테일 체인 | 전국 지점 POS/CCTV 통합 | 지점별 VPN 장비 비용 제거 |
| 건설/현장 | 현장 사무소 임시 네트워크 | LTE 라우터 뒤에서도 작동(NAT 통과) |
| 교육 | 학교 폐쇄망 + 원격수업 | 공공 조달 시장 |

- **가격**: 구축 1,000만~1억 원 + 연 유지보수 20~30%
- **필요**: 도메인 지식 + 영업망. 혼자서는 어려워 파트너 필요

#### 8. 어플라이언스 하드웨어 판매

**근거**: `gokrazy/` 디렉터리 존재 + Makefile에 `tsapp-build-and-flash-pi`, `tsapp-qemu-pi`, `tsapp-push-pi` 타겟 — **라즈베리파이용 Tailscale 전용 OS 이미지 빌드 도구가 이미 포함**되어 있습니다.

- **제품**: "전원 꽂고 QR 찍으면 끝나는 VPN 박스" (Pi/미니PC + Subnet Router·Exit Node 사전 설정 이미지)
- **가격**: 하드웨어 15~30만 원 + 설정 10~20만 원 + 월 관리 3~10만 원
- **리스크**: 재고·물류·A/S 부담. ⚠️ **상표 주의** — 제품에 Tailscale 로고 사용 불가. "Tailscale 사전설치"라는 사실 서술은 가능

### 추천 실행 순서

```
[1~3개월]  유튜브/블로그 콘텐츠 시작 (2번)
              ↓ 인바운드 문의 발생
[2~6개월]  구축/컨설팅 수주 (1번)   ← 현금흐름 확보
              ↓ 고객의 실제 불만 수집
[4~9개월]  관리 대시보드 MVP (3번) — React/PHP 가능
              ↓ 기존 고객에게 먼저 판매 (검증된 수요)
[6~18개월] tsnet 기반 제품 (4번) 또는 AI 에이전트 샌드박스 (5번)
```

**논리**: ① 콘텐츠는 자본 0일 때 가장 효율적인 무료 영업 채널 ② 컨설팅은 즉시 현금 + 고객의 진짜 문제 파악 ③ 그 문제를 제품화하면 실패 확률 급감(검증된 수요) ④ SaaS는 마지막 — **수요 검증 전 제품 개발이 가장 흔한 실패 경로**

### 하나만 고른다면

**「1번 컨설팅 + 2번 콘텐츠」로 시작 → 「4번 tsnet 제품」으로 확장**

- 자본 0으로 시작 가능
- 콘텐츠가 영업을 대신
- 컨설팅으로 현금흐름 + 시장 학습 동시
- tsnet은 **진입장벽이 실질적**(Go + Tailscale 깊은 이해) → 경쟁자가 쉽게 따라오지 못함
- **"로그인 구현 0줄"은 설명하기 쉬운 가치 제안** → 영업이 쉬운 제품이 좋은 제품

### 하지 말아야 할 것

| 하지 말 것 | 이유 |
|---|---|
| Tailscale 계정/서비스 재판매 | ToS 위반 가능성 |
| 제품명에 Tailscale 사용 | 상표권 침해 |
| 코드 포크해 "우리 VPN"으로 판매 | 합법이나 60만 줄을 유지보수할 수 없음 |
| 고객 API 토큰 평문 보관 | 한 번 유출되면 사업 종료 |
| 본가가 곧 추가할 기능으로 SaaS | 기능 추가일에 사업 소멸 |

---

## 부록: 주요 참고 경로

| 알고 싶은 것 | 볼 파일 |
|---|---|
| 전체 개요 / 빌드 방법 | `README.md` |
| 앱에 Tailscale 내장 | `tsnet/tsnet.go`, `tsnet/README.md`, `tsnet/example/` |
| NAT 통과 원리 | `disco/disco.go`, `net/netcheck/`, `net/stun/`, `net/portmapper/`, `wgengine/magicsock/` |
| 중계 서버 프로토콜 | `derp/derp.go` |
| 노드 상태 관리 | `ipn/ipnlocal/local.go` |
| 모듈러 기능 설계 | `feature/README.md`, `feature/featuretags/featuretags.go` |
| 컨테이너 환경변수 전체 | `cmd/containerboot/main.go` (상단 주석) |
| 쿠버네티스 | `k8s-operator/`, `docs/k8s/README.md`, `docs/k8s/operator-architecture.md` |
| 로컬 데몬 제어 API | `client/local/local.go` |
| 제어 평면 REST API | `client/tailscale/`, 공식 문서 https://tailscale.com/api |
| 웹 UI (React) | `client/web/src/` |
| CLI 설계 철학 | `docs/cli.md` |
| Tailnet Lock 암호학 | `tka/tka.go` |
| 빌드 타겟 전체 | `Makefile` (`make help`) |
| 커밋 메시지 규칙 | `docs/commit-messages.md` |
