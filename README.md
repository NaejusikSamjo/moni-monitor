# MONI 모니터링 서버

## 기술 스택

| 구성 요소 | 역할 |
|-----------|------|
| Grafana Alloy | 로그 수집(Promtail 대체) + Eureka SD 메트릭 스크래핑 |
| Prometheus | 메트릭 저장 (remote_write 수신) |
| Grafana Loki | 로그 저장 |
| Grafana Tempo | 분산 추적 저장 (OTLP gRPC) |
| Grafana | 시각화 (대시보드 7개) |
| Alertmanager | 알림 발송 |
| Discord Webhook | Slack 대신 Discord로 알림 |

## 전체 구조

```
service-ec2                monitor-ec2
┌────────────────┐         ┌──────────────────────────┐
│ Spring Boot    │─OTLP──▶ │ Tempo :4317              │
│ (with OTel)    │         │                          │
│                │         │ Prometheus :9090 ◀─────┐ │
│ Alloy sidecar  │─remote_write──────────────────────┘ │
│ (docker-compose│         │ Loki :3100 ◀───────────┐ │
│   .alloy.yml)  │─logs──▶ │                        │ │
│                │         │ Alloy (내장) ──logs────▶│ │
│ Eureka :8761   │◀─SD─────│        ──metrics─▶Prom │ │
└────────────────┘         │                          │
                           │ Grafana :3000 (시각화)   │
infra-ec2                  │ Alertmanager :9093        │
┌────────────────┐         └──────────────────────────┘
│ PostgreSQL     │
│ Redis          │
│ Kafka          │
└────────────────┘
```

## 기동 방법

### 로컬 개발 환경

```bash
# 1. 환경변수 설정
cp .env.local.example .env
# LOG_PATH=../moni/logs  ← moni 서비스 로그 경로

# 2. 모니터링 스택 기동
docker compose up -d

# 3. Grafana 접속 (로컬: admin / admin)
open http://localhost:3000
```

### 운영 환경 (3개 EC2)

```bash
# monitor-ec2 에서:
cp .env.prod.example .env.prod
# 필수 값 채우기: AWS, S3, OKTA, SERVICE_PRIVATE_IP 등

docker compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env.prod up -d
```

## PROMETHEUS Scrape 정책

| job | 주기 | 수집 대상 | 목적 |
|-----|------|-----------|------|
| `moni-health` | 30초 | `/actuator/prometheus` (up만) | 서비스 생존 여부 |
| `moni-signals` | 60초 | HTTP 요청 + HikariCP | 에러율 + 레이턴시 |
| `moni-jvm` | 300초 | JVM + CPU | Heap / GC / Thread |
| `moni-services` | 60초 | 전체 메트릭 | 상세 분석 / 대시보드 |
| `monitor-stack` | 30초 | Alloy/Loki/Tempo/Grafana 자체 | 모니터링 시스템 감시 |

## 대시보드

| 대시보드 | uid | 설명 |
|---------|-----|------|
| Observability | `spring-boot-observability` | **메인 대시보드** — 서비스 health, HTTP 레이턴시/에러율, JVM, 로그, 트레이스 |
| Trace & Log Explorer | `moni-trace-log` | 서비스맵 + Span 레이턴시 + 로그 드릴다운 |
| Infrastructure | `moni-infra` | Node CPU/메모리 + Kafka Lag + PostgreSQL 상태 |

## VPC 3 EC2 통신 구성

배포 환경에서 같은 VPC Private Subnet 내 3개 EC2 통신:

| 방향 | 프로토콜:포트 | 보안 그룹 규칙 |
|------|-------------|---------------|
| service-ec2 → monitor-ec2 | TCP:4317 | OTLP gRPC (Tempo) |
| service-ec2 → monitor-ec2 | TCP:9090 | Prometheus remote_write |
| service-ec2 → monitor-ec2 | TCP:3100 | Loki push (Alloy sidecar) |
| monitor-ec2 → service-ec2 | TCP:8761 | Eureka SD (서비스 발견) |
| monitor-ec2 → service-ec2 | TCP:8080~19097 | /actuator/prometheus 스크래핑 |
| 외부 → monitor-ec2 | TCP:443/80 | Grafana (Nginx + HTTPS) |

### service-ec2 .env 핵심 설정

```env
# service-ec2 서비스 .env
OTEL_EXPORTER_OTLP_ENDPOINT=http://10.0.1.30:4317   # monitor-ec2 private IP
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_TRACES_EXPORTER=otlp
```

## docker-compose.alloy.yml 기동 시점

`docker-compose.alloy.yml`은 **service-ec2 전용** 사이드카 Alloy입니다.

| 환경 | 사용 여부 |
|------|----------|
| 로컬 개발 | ❌ 사용 안 함 (docker-compose.yml의 alloy로 충분) |
| service-ec2 (운영) | ✅ 서비스 기동 후 실행 |
| monitor-ec2 (운영) | ❌ 사용 안 함 (docker-compose.yml에 alloy 내장) |

기동 순서 (service-ec2):
1. `docker-compose.infra.yml up -d` (DB, Redis, Kafka)
2. `docker-compose.yml up -d` (Spring Boot 서비스)
3. `docker-compose.alloy.yml --env-file .env.alloy up -d` (Alloy 사이드카)

## S3 운영 이전

운영 환경에서 Loki와 Tempo 데이터를 S3에 저장합니다:

```bash
# S3 버킷 생성 (ap-northeast-2)
aws s3 mb s3://moni-loki-logs
aws s3 mb s3://moni-tempo-traces

# IAM 정책: PutObject/GetObject/DeleteObject/ListBucket 권한 필요
# docker-compose.prod.yml 이 loki.prod.yaml, tempo.prod.yaml 을 사용
```
