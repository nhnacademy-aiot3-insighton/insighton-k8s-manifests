# insighton-k8s-manifests

InsightOn 운영 클러스터의 Kubernetes 매니페스트 저장소. 이 저장소의 git 상태가 곧 클러스터가 도달해야 하는 상태(desired state)이며, ArgoCD가 이를 실제 클러스터와 지속적으로 비교/동기화한다(GitOps).

## 이 저장소가 배포 파이프라인에서 하는 역할

이미지 빌드는 여기서 하지 않는다. 각 서비스 저장소 → [insighton-infra](../insighton-infra)의 재사용 워크플로우가 GHCR에 이미지를 푸시한 뒤, 마지막 `update-manifest` 단계가 **이 저장소**를 clone해서 해당 서비스 `deployment.yaml`의 이미지 태그를 커밋 SHA로 치환하고 push한다.

```
<service-repo> push
  → insighton-infra 재사용 워크플로우 (build-and-test → sonar-scan → build-and-push)
  → insighton-k8s-manifests의 infra/<service>/deployment.yaml 이미지 태그 갱신 커밋 (본 저장소)
  → ArgoCD가 변경 감지 → 클러스터에 반영 (실제 롤아웃)
```

즉 이 저장소에 대한 수동 편집은 이미지 태그 외의 인프라 변경(리소스 조정, 새 서비스 추가, ingress/모니터링 설정 변경 등)에만 하면 되고, ArgoCD는 `infra/` 하위 폴더 하나당 Application 하나를 두고 감시한다(`infra/ingress`만 예외로 3개 파일을 한 Application이 함께 관리).

## Namespace 구성

| Namespace | 용도 |
| --- | --- |
| `insighton` | 애플리케이션 서비스 전체 (gateway, auth, core, ai, ruleengine, front, config, influxdb, zipkin, redis-telemetry, cloudflared, ingress) |
| `insighton-logging` | 관측성 스택 (Grafana, Loki, Alloy) — 별도 네임스페이스로 분리 |

## 서비스 오브젝트 (`infra/`, namespace: `insighton`)

| 서비스 | 종류 | Secret | 비고 |
| --- | --- | --- | --- |
| gateway | Deployment (2 replicas) | — | JWT 공개키는 properties에 직접 내장 |
| front | Deployment (2 replicas) | insighton-front-secret | 순수 BFF, TELEMETRY_REDIS만 사용 |
| auth | Deployment | insighton-auth-secret | DB, Redis, JWT 키, OAuth, 메일 |
| **core** | **StatefulSet** (2 replicas) | insighton-core-secret | 서수(ordinal) 기반 인스턴스 슬롯 자동 주입 |
| **ruleengine** | **StatefulSet** (2 replicas) | insighton-ruleengine-secret | 서수 기반 engine-id/큐 분담 자동 주입 |
| ai | Deployment (2 replicas) | insighton-ai-secret | DB, Redis, RabbitMQ, Gemini API |
| config | Deployment (2 replicas) | config-secrets | Spring Cloud Config, GitHub 백엔드 저장소 연동 |
| influxdb | Deployment (1 replica) | insighton-influxdb-secret | PVC로 영속 저장 |
| redis-telemetry | Deployment | — | 텔레메트리 캐시용 Redis |
| zipkin | Deployment (1 replica) | — | 인메모리 저장, 무상태 |
| cloudflared | Deployment (2 replicas) | insighton-cloudflared-secret | 클러스터 → Cloudflare 엣지로의 아웃바운드 터널 |

`core`와 `ruleengine`만 StatefulSet인 이유: 두 서비스는 인스턴스마다 담당 범위가 달라야 한다(core는 슬롯 인덱스, ruleengine은 짝/홀수 큐 + engine-a/b 분담). `$HOSTNAME`의 서수(`insighton-core-0`, `-1` …)를 `RuleEngineInstanceEnvironmentPostProcessor` / `InstanceIndexEnvironmentPostProcessor`가 읽어 자동 배정한다. StatefulSet은 `serviceName`으로 headless Service(`clusterIP: None`)를 요구하므로 `insighton-core-headless`, `insighton-ruleengine-headless`가 있지만, Pod별 DNS 조회 기능 자체는 사용하지 않고 API 요구사항 충족용으로만 존재한다.

모든 서비스 컨테이너는 `automountServiceAccountToken: false`, `imagePullSecrets: insighton-ghcr-secret`(GHCR private pull), liveness/readiness(+core/ruleengine는 startup) probe, `preStop: sleep 5`(graceful shutdown과 맞춤)를 공통으로 갖는다.

## 관측성 스택 (`infra/{alloy,loki,grafana}`, namespace: `insighton-logging`)

- **Alloy** — 각 노드에서 컨테이너 로그를 수집하는 DaemonSet, Loki로 전송 (Promtail을 대체함, [`_deprecated/`](#_deprecated) 참고)
- **Loki** — 로그 저장소 (PVC로 영속화)
- **Grafana** — 대시보드/데이터소스가 ConfigMap으로 프로비저닝되어 배포 시 자동 구성됨. `k8s-viewer-rbac.yaml`로 `insighton` 네임스페이스 리소스 조회 권한을 부여받음
- **Zipkin**(`insighton` ns) — 분산 트레이싱, 각 서비스가 `management.tracing.export.zipkin.endpoint`로 전송

## Ingress / 외부 노출 (`infra/ingress/`, `infra/cloudflared/`)

| Host | Path | 대상 | 비고 |
| --- | --- | --- | --- |
| `insighton.store`, `www.insighton.store` | `/api` | insighton-gateway | |
| `insighton.store`, `www.insighton.store` | `/` | insighton-front | |
| `insighton.store` | `/grafana` | grafana (insighton-logging) | |
| `insighton.store` | `/zipkin` | zipkin | Basic Auth (`insighton-team-basic-auth`) |
| `influxdb.insighton.store` | `/` | insighton-influxdb | |

인바운드는 표준 80/443이 아닌 NodePort로 여는 대신, **Cloudflare Tunnel**(`cloudflared`)이 클러스터 내부에서 Cloudflare 엣지로 아웃바운드 연결을 걸어(dial) 두고 그 위로 트래픽을 받아온다. 그 덕분에 클러스터/방화벽에 별도 인바운드 포트를 열 필요가 없다.

```
Browser → Cloudflare → cloudflared → ingress-nginx → Service → Pod
```

`nginx.ingress.kubernetes.io/*` 어노테이션(rate limit, proxy timeout/buffer, Basic Auth)은 Ingress 스펙 표준이 아니라 `ingress-nginx` 컨트롤러 전용 확장이라, 다른 컨트롤러로 바꾸면 무시된다.

## 사전 준비 (클러스터에 이미 존재해야 하는 리소스)

이 저장소가 직접 만들지 않는, 클러스터에 수동/별도 절차로 준비되어 있어야 하는 것들:

- `insighton`, `insighton-logging` 네임스페이스 (단, `namespace.yaml`은 `insighton`만 정의)
- `insighton-ghcr-secret` (GHCR pull용 imagePullSecret)
- 서비스별 Secret (`insighton-*-secret`, `config-secrets`) — DB/Redis/RabbitMQ/외부 API 키 등
- `insighton-tls` (Ingress TLS), `insighton-team-basic-auth` (zipkin Basic Auth)
- `insighton-cloudflared-secret`의 `TUNNEL_TOKEN`
- `ingress-nginx` 컨트롤러, ArgoCD 자체 설치

## `_deprecated/`

더 이상 클러스터에 배포하지 않는 매니페스트를 보관 (ArgoCD 감시 대상인 `infra/` 밖에 위치).

- `_deprecated/promtail/` — 로그 수집 DaemonSet. Alloy로 대체되어 제거됨

## `docs/notes.md`

이 저장소의 각 오브젝트가 왜 이렇게 구성되었는지(Pod/Deployment/StatefulSet/Service/Headless Service/Namespace/Secret의 실제 동작 원리부터, ingress-nginx 어노테이션과 cloudflared 터널링 방식까지)를 정리한 학습/레퍼런스 문서. 신규 합류자는 이 문서를 먼저 읽는 것을 권장.
