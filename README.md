# monitoring-lab

VirtualBox VM 3대(ubuntu-server, web02, web03)를 **Prometheus + Grafana**로 모니터링하고, 장애가 나면 **디스코드로 알림**을 받는 실습 프로젝트 (2026-11-10 ~ 11-11)

## 결과 요약
| 항목 | 결과 |
|---|---|
| 수집 대상 | 서버 3대 × (node + nginx) = 6개 + Prometheus 자신, 전부 UP |
| 대시보드 | `Lab Overview` (코드로 관리) + `Node Exporter Full` (1860) |
| 알림 규칙 | InstanceDown · NginxDown · HighCPU · DiskAlmostFull |
| 장애 훈련 | Nginx 중지 → 📱 FIRING/RESOLVED ✅ · 서버 전원 끄기 → 📱 FIRING/RESOLVED ✅ |

## 구조
```
💻 노트북 브라우저 ── http://192.168.56.101:3000 ──> Grafana (그래프)
                                                       │
🖥️ ubuntu-server 192.168.56.101 (Docker Compose)        ▼
   [Prometheus :9090] ── 15초마다 수집 ──> 서버 3대의 exporter
        │ 규칙에 걸림                        node_exporter  :9100 (CPU·메모리·디스크·네트워크)
        ▼                                    nginx_exporter :9113 (요청 수·접속 수)
   [Alertmanager :9093] ──> 📱 디스코드 #alerts
```

| 부품 | 하는 일 | 비유 |
|---|---|---|
| exporter | 서버 상태를 숫자(지표)로 내놓음 | 체온계 |
| Prometheus | 15초마다 숫자를 모아 저장, 알림 규칙 검사 | 간호사 기록지 |
| Grafana | 숫자를 그래프로 | 모니터 화면 |
| Alertmanager | 알림을 묶어서 디스코드로 전송 | 응급 호출벨 |

## 서버
| VM | 이름 | IP | 역할 |
|---|---|---|---|
| 1 | ubuntu-server | 192.168.56.101 | 모니터링 서버 (Docker) + 감시 대상 |
| 2 | web02 | 192.168.56.102 | 감시 대상 |
| 3 | web03 | 192.168.56.104 | 감시 대상 |

## 폴더
```
monitoring-lab/
├── docker-compose.yml          # Prometheus · Grafana · Alertmanager
├── .env.example                # Grafana 계정 양식 (.env는 Git 제외)
├── prometheus/
│   ├── prometheus.yml          # 수집 대상, Alertmanager 주소
│   └── rules/alerts.yml        # 알림 규칙 4개
├── alertmanager/alertmanager.yml   # 디스코드 전송 (웹후크는 secrets/ 파일, Git 제외)
├── grafana/provisioning/
│   ├── datasources/            # Prometheus 자동 연결
│   └── dashboards/json/        # Lab Overview 대시보드
└── ansible/                    # 서버 3대에 exporter 설치 + 방화벽
    ├── site.yml, inventory.ini
    └── roles/ node_exporter, nginx_exporter
```

---

## 진행 기록

### Phase 1. Prometheus + Grafana (Docker Compose)
- ubuntu-server에 Docker 설치 (`docker.io`, `docker-compose-v2`), `www`를 `docker` 그룹에 추가
- Grafana가 켜질 때 Prometheus 데이터 소스를 **자동 등록**(provisioning) → `Test` 성공
- Prometheus 타깃 `prometheus` 1/1 UP

<!-- 📸 docker compose ps / Grafana Data source Test / Prometheus targets -->

### Phase 2. 서버 3대에 exporter (Ansible)
```bash
cd ansible && ansible-playbook site.yml -K
# ubuntu-server, web02, web03 모두 failed=0
```
| 설계 | 이유 |
|---|---|
| ubuntu-server도 `localhost`가 아닌 **IP**로 수집 | Prometheus는 컨테이너 안에서 실행 → 거기서 `localhost`는 컨테이너 자신 |
| 9100·9113은 **Prometheus 서버에서만** 허용 (UFW) | 서버 상태 정보를 아무나 보지 못하게 |
| ubuntu-server만 `172.16.0.0/12` 추가 허용 | 컨테이너 → 같은 서버 접속은 출발지가 Docker 내부망 주소 |
| Nginx 상태 페이지는 `127.0.0.1:8080`에만 | 외부에 노출하지 않고 exporter만 읽음 |

- 결과: `node` 3/3 UP, `nginx` 3/3 UP

<!-- 📸 PLAY RECAP / targets 6개 UP -->

### Phase 3. Grafana 대시보드
| 단계 | 내용 |
|---|---|
| A | `Node Exporter Full`(ID 1860) Import |
| B | Explore에서 PromQL 직접 실행 (`up`, 메모리 %, `rate(nginx_http_requests_total[1m])`) |
| C | `Lab Overview` — JSON 파일로 저장소에 두고 Grafana가 자동으로 불러옴 (대시보드도 코드로 관리) |
| D | `curl` 3000회로 Nginx 요청 그래프, 무한 루프로 CPU 그래프가 솟는 것 확인 |

| Lab Overview 패널 | 쿼리 |
|---|---|
| 서버 / Nginx 상태 | `up{job="node"}`, `nginx_up` |
| 디스크·메모리 사용률 | `100 * (1 - 남은 양 / 전체)` |
| CPU 사용률 | `100 * (1 - avg by (host) (rate(node_cpu_seconds_total{mode="idle"}[1m])))` |
| Nginx 초당 요청 / 접속 수 | `rate(nginx_http_requests_total[1m])`, `nginx_connections_active` |
| 네트워크 | `rate(node_network_receive/transmit_bytes_total{device="enp0s8"}[1m])` |

<!-- 📸 Lab Overview / CPU 그래프 솟은 화면 -->

### Phase 4. 알림 + 장애 훈련
**알림 규칙** (`prometheus/rules/alerts.yml`, GitHub에서 직접 작성)

| 규칙 | 조건 | 대기(`for`) | 심각도 |
|---|---|---|---|
| InstanceDown | `up{job="node"} == 0` | 1m | critical |
| NginxDown | `nginx_up == 0` | 30s | critical |
| HighCPU | CPU > 80% | 1m | warning |
| DiskAlmostFull | 디스크 > 85% | 5m | warning |

- 규칙 문법 검사: `docker exec prometheus promtool check rules /etc/prometheus/rules/alerts.yml` → `SUCCESS: 4 rules found`
- 디스코드 웹후크는 **`secrets/discord_webhook` 파일**에 두고 `.gitignore`로 제외, 컨테이너에 읽기 전용으로 연결

**알림 상태 흐름**: 🟢 Inactive → 🟡 Pending (`for` 대기) → 🔴 Firing → Alertmanager → 📱 디스코드

**장애 훈련**
| 훈련 | 방법 | 대시보드 | 알림 |
|---|---|---|---|
| ① Nginx 중지 | web02 `systemctl stop nginx` | web02 Nginx DOWN | 📱 `[FIRING:1] NginxDown web02` → 복구 후 RESOLVED |
| ② CPU 폭주 | web03 모든 코어 무한 루프 | CPU 그래프 상승 | 노트북까지 멈춰 중단 (아래 Troubleshooting) |
| ③ 전원 끄기 | VirtualBox에서 web03 전원 끄기 | web03 DOWN | 📱 `InstanceDown web03` → 재시작 후 RESOLVED |

<!-- 📸 9090/alerts FIRING / 디스코드 FIRING·RESOLVED 메시지 -->

---

## 👀 관찰 포인트
1. UFW는 80·22만 열어뒀는데 **3000·9090·9093이 열림** — Docker가 UFW보다 앞에 방화벽(iptables) 규칙을 직접 넣기 때문. 반대로 일반 프로그램인 exporter(9100)는 UFW에 막혀 허용 규칙이 필요했음
2. 훈련 전부터 **ubuntu-server 디스크 80%** 가 대시보드에 주황색으로 표시 — Docker 이미지·수집 데이터 때문. 모니터링이 실제 문제를 먼저 보여줌
3. 서버 전원 끄기 알림은 Nginx 중지보다 늦게 옴 — 규칙의 `for`(1m vs 30s) + Alertmanager 묶기 시간 차이

## Troubleshooting
| 문제 | 원인 | 해결 |
|---|---|---|
| 설정을 바꿨는데 targets에 안 나옴 | Prometheus는 켜질 때만 설정을 읽음 | `curl -X POST localhost:9090/-/reload` |
| Grafana에서 Data source가 안 보임 | `Add new connection`(설치 가능한 목록)을 보고 있었음 | `Connections → Data sources` |
| nano 저장이 계속 실패 | 없는 `secrets/` 폴더에 저장 시도 | `mkdir -p secrets` 후 `read -rs`로 저장 (화면에 URL 노출도 방지) |
| 디스코드 알림의 Source 링크가 안 열림 | 링크가 컨테이너 내부 이름(`a56dae4ddb8f:9090`) | `--web.external-url=http://192.168.56.101:9090` |
| CPU 훈련 중 노트북 전체가 멈춤 | VM은 노트북의 실제 CPU를 나눠 씀 → 한 VM이 모든 코어를 쓰면 다른 VM·브라우저까지 느려짐 (시끄러운 이웃) | `timeout`으로 자동 종료 대기. 재시도 시 VM CPU를 1개로 줄이고 루프 1개만 |
| 쿼리 4개가 그래프 하나에 겹쳐 안 보임 | 단위가 다름 (바이트 vs %) | 눈 아이콘으로 하나씩 보기 |

## 실행 (ubuntu-server)
```bash
git clone https://github.com/network-growth/monitoring-lab ~/monitoring-lab
cd ~/monitoring-lab
cp .env.example .env && nano .env                 # Grafana 비밀번호
mkdir -p secrets && read -rs URL && printf '%s\n' "$URL" > secrets/discord_webhook && unset URL
chmod 755 secrets && chmod 644 secrets/discord_webhook
(cd ansible && ansible-playbook site.yml -K)      # 서버 3대에 exporter
docker compose up -d
```
| 화면 | 주소 |
|---|---|
| Grafana | http://192.168.56.101:3000 |
| Prometheus | http://192.168.56.101:9090 |
| Alertmanager | http://192.168.56.101:9093 |

## 다음에 해볼 것
| 주제 | 내용 |
|---|---|
| Blackbox exporter | 사용자 입장에서 웹 페이지 응답(200 OK, 응답 시간) 감시 |
| Loki | 서버 3대 로그를 Grafana에서 검색 (SSH 로그인 실패 등) |
| 보안 | Docker 포트 노출 정리, Grafana HTTPS |
| CI | push 시 `promtool`로 설정·규칙 자동 검사 |
