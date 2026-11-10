# monitoring-lab

VirtualBox VM 3대(ubuntu-server, web02, web03)를 **Prometheus + Grafana**로 모니터링하고, 장애가 나면 **디스코드로 알림**을 받는 실습 프로젝트

## 구조
```
💻 노트북 브라우저 ── http://192.168.56.101:3000 ──> Grafana (그래프)
                                                       │
🖥️ ubuntu-server 192.168.56.101                         ▼
   [Prometheus] ── 15초마다 수집 ──> 각 서버의 exporter (CPU·메모리·디스크·Nginx)
        │ 이상 감지
        ▼
   [Alertmanager] ──> 📱 디스코드 #alerts
```

| 부품 | 하는 일 |
|---|---|
| exporter | 서버 상태를 숫자(지표)로 내놓음 |
| Prometheus | 15초마다 숫자를 모아 저장 |
| Grafana | 숫자를 그래프로 |
| Alertmanager | 기준을 넘으면 알림 전송 |

## 서버
| VM | 이름 | IP | 역할 |
|---|---|---|---|
| 1 | ubuntu-server | 192.168.56.101 | 모니터링 서버 (Docker) + 관리 대상 |
| 2 | web02 | 192.168.56.102 | 관리 대상 |
| 3 | web03 | 192.168.56.104 | 관리 대상 |

## 진행
| Phase | 내용 | 상태 |
|---|---|---|
| 1 | Prometheus + Grafana (Docker Compose) | 🔄 |
| 2 | 서버 3대에 exporter (Ansible) + 방화벽 | ⬜ |
| 3 | Grafana 대시보드 | ⬜ |
| 4 | Alertmanager → 디스코드, 장애 훈련 | ⬜ |
| 5 | 정리 | ⬜ |

## 실행 (ubuntu-server)
```bash
git clone https://github.com/network-growth/monitoring-lab ~/monitoring-lab
cd ~/monitoring-lab
cp .env.example .env && nano .env    # Grafana 비밀번호 변경
docker compose up -d
```
| 화면 | 주소 |
|---|---|
| Grafana | http://192.168.56.101:3000 |
| Prometheus | http://192.168.56.101:9090 |
