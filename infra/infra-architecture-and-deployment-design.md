# Emoselfie 인프라 아키텍처 및 배포 설계

기준일: 2026-09-14  
상태: 제출용 초안. 코드로 확인한 설계와 실행 검증을 구분한다.  
범위: 로컬 k3d와 운영용 k3s 매니페스트, Docker 이미지 빌드 및 추적, 상태 판정, 자원 및 관제 설계.

## 1. 전체 아키텍처

![기존 k3d 아키텍처 구성도](./k3d-architecture.svg)

그림은 기존 로컬 3노드 설계다. 그림의 `localhost:80`은 설계 예시이며, 이번 실습의 기존 `emoselfie-full` 클러스터는 `http://localhost:8082`를 사용한다. 그림의 Sentinel 화살표는 master 탐색 관계를 뜻한다. 실제 데이터 명령은 backend가 탐색한 Redis master에 직접 보낸다.

| 구성 요소 | 역할 |
|---|---|
| k3d serverlb / Traefik | 호스트 포트를 클러스터에 연결하고 Ingress 규칙으로 web Service에 전달 |
| web Deployment / Service | nginx가 FE 정적 파일을 제공하고 API를 동일 오리진으로 프록시 |
| backend Deployment / Service | FastAPI, Socket.IO, 감정 추론 실행. Service 포트와 컨테이너 포트는 8000 |
| PostgreSQL StatefulSet | 영속 데이터 저장. 현재 단일 인스턴스 |
| Redis / Sentinel StatefulSet | 실시간 조정용 Redis와 master 탐색·장애 감시 |
| redis-media Deployment | 결과 이미지 캐시 |
| prepare-models initContainer / models PVC | 모델 다운로드·SHA256 검증 후 backend에 읽기 전용 모델 제공 |
| migrate Job | Alembic DB 스키마 변경. 신규 실행은 DB 변경 승인을 전제로 함 |

요청 흐름: 브라우저 → serverlb → Traefik Ingress → web Service → nginx. 정적 파일은 nginx가 응답하고 `/api/`, `/socket.io/`, `/media/`, `/health/`는 backend Service로 전달한다. FE·BE의 오리진을 통일해 쿠키와 Origin 검사에 필요한 Host·scheme을 유지한다.

로컬 namespace는 `local`, 운영은 `emoselfie`다. Kustomize의 namespace 지정은 객체의 소속을 설정하며 Namespace 객체 자체를 생성하지는 않는다. 생성 절차는 배포 README에 있다.

## 2. Docker 이미지 빌드와 버전 관리

### 2.1 빌드 방식

| 이미지 | 소스 | 빌드 방식 |
|---|---|---|
| backend runtime | BE Dockerfile의 `runtime` | builder에서 `uv sync --locked --no-dev --extra inference`, runtime으로 가상환경·앱 복사. UID 10001로 실행 |
| model preparation | BE Dockerfile의 `models` | 모델 준비 스크립트와 다운로드 의존성만 포함. 가중치는 시작 시 PVC에 준비 |
| web runtime | FE Dockerfile의 `runtime` | `pnpm install --frozen-lockfile`, `pnpm build` 후 dist를 nginx 이미지로 복사 |

BE·FE 각각의 GitHub Actions는 품질 검사 성공 후 이미지를 빌드한다. PR에서는 빌드만 하고, `main` push에서는 OIDC로 AWS에 인증하여 ECR에 게시하도록 설정되어 있다. CI 대상 플랫폼은 `linux/amd64`다. 로컬은 현재 머신 아키텍처로 빌드하며, 로컬 가이드에는 arm64 의존성 분기가 설명되어 있다. Dockerfile의 과거 amd64 전용 주석과 현재 로컬 가이드가 다르므로 주석만으로 로컬 빌드 불가를 판단하지 않는다.

로컬 빌드 명령은 상위 프로젝트 디렉터리에서 실행한다. 아래는 설명용이며 이번 문서 작성이 빌드 성공 증빙을 대신하지 않는다.

```bash
docker build --target runtime -t emoselfie-backend:local ./emoselfie-BE
docker build --target models -t emoselfie-models:local ./emoselfie-BE
docker build --target runtime -t emoselfie-web:local ./emoselfie-FE
```

### 2.2 SHA 태그 규칙과 소스 연결

운영 CI 태그는 `sha-` 뒤에 해당 저장소의 `GITHUB_SHA` 앞 12자리를 붙인다. 형식은 `^sha-[0-9a-f]{12}$`이며 숫자만 12자리가 아니라 소문자 16진수 12자리다. 예: `sha-012345abcdef`는 형식 예시이고 실제 배포 증빙은 아니다.

1. BE 커밋의 전체 SHA → 앞 12자리 → backend와 models 이미지의 동일 태그.
2. FE 커밋의 전체 SHA → 앞 12자리 → web 이미지 태그. BE 태그와 같을 필요는 없다.
3. Ansible이 `backend_tag`, `frontend_tag` 형식을 검사한다.
4. runtime Kustomize overlay가 ECR 저장소명과 태그를 주입한다.
5. 최종 Deployment와 initContainer의 `image`에 해당 이미지 참조가 반영된다.

추적 흐름은 `저장소 + 전체 소스 SHA → CI 실행 → 이미지 URI:태그 → 배포 Pod의 imageID(digest)`다. 태그 문자열만으로 내용의 불변성을 증명할 수는 없다. 제출 시 실제 digest와 CI 기록을 함께 보관한다. 미커밋 변경이 있는 로컬 이미지는 커밋 SHA만으로 재현할 수 없다.

`overlays/local`의 `local` 태그는 반복 개발용으로 소스 커밋을 특정하지 않는다. `overlays/prod`의 `amd64-v1`/`v1`은 기본값이며, Ansible 배포에서 SHA 태그로 덮어쓰는 구조다. CI 정의가 있다는 사실과 실제 CI 게시 성공은 별도다.

| 제출 기록 | BE runtime / models | FE web |
|---|---|---|
| 전체 소스 SHA |  |  |
| 이미지 URI 및 SHA 태그 |  |  |
| 실행 이미지 digest |  |  |
| CI 실행 URL |  |  |

## 3. 상태 확인과 probe 판정 기준

매니페스트는 검사 경로·주기·실패 처리 방법을 정한다. 정상 응답을 만드는 내부 조건은 BE의 `app/api/health.py`와 `app/core/resources.py`에서 확인해야 한다.

| 엔드포인트 | 정상 판정 | 실패 판정 |
|---|---|---|
| `/health/live` | 핸들러가 응답하면 HTTP 200, `status: ok` | 프로세스·서버가 응답하지 않거나 HTTP 검사 실패. DB·Redis·추론 정확도는 검사하지 않음 |
| `/health/ready` | 아래 4개 checks가 모두 참이면 HTTP 200, `status: ready` | 하나라도 거짓이면 HTTP 503, `status: not_ready` |

readiness checks:

- `database`: 제한 시간 안에 PostgreSQL 연결 및 `SELECT 1` 성공.
- `redis`: 조정용 Redis 클라이언트의 PING 성공.
- `mediaRedis`: 이미지 캐시 Redis의 PING 성공.
- `inference`: `resources.initialized`가 참이고 classifier가 존재. 이 응답만으로 실제 이미지 추론의 품질·지연까지 검증되지는 않는다.

| probe | 경로 / 포트 | 주기 | 연속 실패 임계값 | 실패 시 동작 |
|---|---|---|---|---|
| startup | `/health/ready`:8000 | 5초 | 24회 | 시작에 약 120초의 검사 예산 부여. 임계값 도달 시 컨테이너 재시작 |
| readiness | `/health/ready`:8000 | 5초 | 3회(기본값) | Pod를 미준비로 표시하여 Service의 일반 트래픽 대상에서 제외. 이 검사만으로 재시작하지 않음 |
| liveness | `/health/live`:8000 | 10초 | 3회(기본값) | 컨테이너 재시작 |

세 probe 모두 별도 지정이 없는 `timeoutSeconds`는 1초, `successThreshold`는 1회다. startup 성공 전에는 readiness·liveness 검사가 시작되지 않는다. 위 시간은 검사 간격 기준 설명이며 정확한 장애 복구 시간을 보장하지 않는다. HTTP probe의 성공 범위는 200 이상 400 미만이며 응답 JSON을 직접 해석하지 않는다. 이 앱은 정상 200 / 준비 실패 503으로 그 판단을 전달한다. [Kubernetes probe 문서](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

요구사항의 `/healthz`, `/readyz`와 실제 경로가 다르다. 동등 기능 인정 또는 별칭 경로 추가 여부는 미결정이다. 이번 문서 작성에서는 API를 변경하지 않는다.

## 4. CPU·Memory Request/Limit

| 대상 컨테이너 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| backend | 1 CPU | 1 CPU | 2Gi | 2Gi |
| prepare-models initContainer | 미지정 | 미지정 | 미지정 | 미지정 |

Request는 스케줄링에 사용하는 자원 요청량이며 실제 사용량과 다르다. CPU Limit은 실행 CPU 시간 제한, Memory Limit은 메모리 상한이다. CPU 사용량의 `1000m`은 1 CPU, 메모리 `2Gi`는 2048Mi다.

| 설정 근거 작성란 | 내용 |
|---|---|
| CPU Request 1 설정 근거 |  |
| CPU Limit 1 설정 근거 |  |
| Memory Request 2Gi 설정 근거 |  |
| Memory Limit 2Gi 설정 근거 |  |
| 실측 조건·결과 및 여유율 |  |

사용자 요청에 따라 설정 근거는 빈칸으로 둔다. backend 컨테이너에서 Request와 Limit이 같다는 사실만으로 Pod의 Guaranteed QoS를 단정하지 않는다. 현재 initContainer에는 설정이 없다. 모든 컨테이너의 조건과 실제 `status.qosClass`를 확인해야 한다. [Kubernetes QoS 문서](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

## 5. ConfigMap·Secret·다중 Replica·HPA

| 항목 | 현재 설계 | 확인할 사항 |
|---|---|---|
| ConfigMap | `backend-env`에 모델 경로·추론 동시성·Redis 주소, `nginx-conf`에 프록시 설정 분리. overlay에서 환경별 값 병합 | backend의 `envFrom` 참조와 web의 볼륨 참조 |
| Secret | backend가 `emoselfie-secrets`를 `envFrom`으로 사용. 로컬은 개발용 generator, 운영은 별도 생성 | 실제 값 대신 객체명·키 목록·참조 관계만 증빙 |
| 다중 Replica | backend는 base 1, local 3, prod 2. web은 base 3 | base만 읽지 말고 최종 overlay 및 실제 Ready 수 확인 |
| 노드 분산 | backend의 preferred pod anti-affinity로 노드별 분산 선호 | 강제 분산이 아니므로 `get pods -o wide`로 실제 배치 확인 |
| HPA | 현재 저장소에서 매니페스트 미확인 | 선택 구현이며 강사 협의 전 도입하지 않음 |
| Grafana | 현재 저장소에서 구성 미확인 | 선택 구현. `kubectl top`과 Grafana 대시보드는 다른 증빙 |

ConfigMap generator의 내용 변경은 이름 해시와 Pod 참조를 바꿔 rollout을 유발할 수 있다. 운영처럼 고정 이름 Secret을 직접 갱신하는 경우 이미 실행 중인 환경 변수는 자동 갱신되지 않으므로 별도 재배포 절차가 필요하다.

Socket.IO polling은 같은 세션 요청의 backend 일관성이 필요하다. prod에는 backend Service의 `sessionAffinity: ClientIP` 패치가 있고 local에는 같은 패치가 없다. 또한 Service가 보는 source IP는 nginx Pod일 수 있으므로 ClientIP만으로 사용자 단위 고정·균등 분산을 보장한다고 쓰지 않는다. 다중 replica 상태에서 실제 연결 유지·게임 진행을 별도 검증해야 한다. PostgreSQL 단일 인스턴스 등 다른 단일 장애점도 남는다.

HPA를 채택하면 메트릭 공급원, 대상 Deployment, 최소·최대 replica, 목표 사용률, scale-down 안정화와 부하 테스트를 함께 설계해야 한다. 수치는 아직 정하지 않는다. CPU 사용률 방식은 Request 대비 사용량을 이용하므로 자원 설정 근거와 연결해야 한다. 증설할 노드 여유와 Socket.IO 연결 유지도 검토 대상이다.

## 6. 직접 실행하는 상태·로그·사용량 실습

2026-09-14 실행 준비 결과: Docker Desktop을 시작해 기존 `emoselfie-full` 컨테이너의 기동을 확인했으나, Kubernetes API가 ServiceUnavailable 및 이후 EOF/시간 초과로 조회되지 않았다. 서버 로그에는 etcd peer ID 불일치가 반복되었다. 근본 원인은 미확정이며 서비스 정상 기동은 검증하지 못했다. 아래 명령은 클러스터 복구 후 실행한다. 기존 데이터 초기화·재배포·migration은 수행하지 않았다.

기존 로컬 클러스터를 명시적으로 지정한다. 아래 `k` 함수는 현재 터미널에만 적용되며 기본 kubeconfig context를 바꾸지 않는다. Secret 내용·사용자 이미지·세션 토큰은 캡처에 넣지 않는다.

```bash
k() { kubectl --context k3d-emoselfie-full -n local "$@"; }
date '+%Y-%m-%d %H:%M:%S %Z'
k get nodes
k get pods -o wide
k get deployments,statefulsets,jobs
k get svc,ingress
```

판정: 노드 Ready, backend READY와 AVAILABLE이 목표 replica와 일치, 일반 서비스 Pod의 Ready가 충족되어야 한다. migrate Job의 `Completed`는 정상 종료이므로 서비스 Pod의 `Running`과 구분한다. Pending·CrashLoopBackOff·지속 증가하는 재시작은 조사 대상이다.

### 6.1 외부 요청과 상태 응답

```bash
curl -i --max-time 10 http://localhost:8082/
curl -i --max-time 10 http://localhost:8082/health/live
curl -i --max-time 10 http://localhost:8082/health/ready
k get endpointslices -l kubernetes.io/service-name=backend
```

판정: 첫 요청의 FE HTML, live의 200·ok, ready의 200·4개 checks 모두 true를 각각 확인한다. 외부 요청 하나의 성공은 모든 backend replica의 정상 상태를 증명하지 않으므로 Pod Ready 및 EndpointSlice와 함께 판단한다.

### 6.2 로그와 이벤트

```bash
k logs -l app=backend -c backend --tail=50 --prefix --timestamps
k logs -l app=backend -c prepare-models --tail=20 --prefix --timestamps
k get events --sort-by=.metadata.creationTimestamp
# 지속 관찰은 별도 터미널에서 실행. Ctrl+C는 로그 관찰만 종료한다.
k logs -l app=backend -c backend -f --tail=20 --prefix --max-log-requests=5
```

판정: initContainer의 모델 준비 완료, backend 시작 및 요청 처리 기록을 확인한다. 오류가 보이면 같은 시각 이벤트·Pod 상태와 비교한다. 로그에 민감 정보가 포함되면 캡처 전 가린다.

### 6.3 CPU·Memory와 설정 비교

```bash
k top nodes
k top pods --containers
k get pods -l app=backend -o custom-columns='NAME:.metadata.name,QOS:.status.qosClass,CPU_REQ:.spec.containers[0].resources.requests.cpu,CPU_LIMIT:.spec.containers[0].resources.limits.cpu,MEM_REQ:.spec.containers[0].resources.requests.memory,MEM_LIMIT:.spec.containers[0].resources.limits.memory'
```

브라우저에서 실제 요청을 수행하기 전과 수행 중에 `top`을 반복해 시각·요청 조건과 함께 기록한다. 단일 순간 측정만으로 Limit 적정성을 결론내리지 않는다. `Metrics API not available` 또는 값 누락 시 사용량 0으로 기록하지 않는다. 먼저 다음 명령으로 공급원을 확인한다.

```bash
k get apiservice v1beta1.metrics.k8s.io
kubectl --context k3d-emoselfie-full -n kube-system get deployment metrics-server
```

관제 흐름: kubelet의 Pod·컨테이너 메트릭 → metrics-server → Metrics API → `kubectl top`. 상태·이벤트는 Kubernetes API로, 컨테이너 표준 출력 로그는 `kubectl logs`로 조회한다. 장기 보관·Grafana 시계열 수집 구성은 이 조회 경로와 별도다.

### 6.4 선택 구현과 실행 이미지 확인

```bash
k get configmaps
k get secrets
k get deploy backend web
k get hpa
k get deploy backend -o jsonpath='{.spec.template.spec.containers[0].envFrom}{"\n"}'
k get deploy backend -o jsonpath='{.spec.template.spec.containers[0].startupProbe}{"\n"}{.spec.template.spec.containers[0].readinessProbe}{"\n"}{.spec.template.spec.containers[0].livenessProbe}{"\n"}'
k get pods -l app=backend -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .status.initContainerStatuses[*]}{.name}{" "}{.image}{" "}{.imageID}{"\n"}{end}{range .status.containerStatuses[*]}{.name}{" "}{.image}{" "}{.imageID}{"\n"}{end}{end}'
```

Secret一覧の取得は値を表示しない。`get secret -o yaml`や`printenv`を提出証拠に使わない。HPAの `No resources found` は未構成の確認結果であって失敗ではない。

| 実測記録 | 結果 | 証拠ファイル |
|---|---|---|
| 計測日時・環境・ソース SHA |  |  |
| Pod Ready・再起動回数・ノード分散 |  |  |
| 外部接続・live・ready |  |  |
| 通常時 / リクエスト時 CPU・Memory |  |  |
| 起動・リクエスト処理ログ |  |  |
| ConfigMap・Secret参照・Replica・HPA |  |  |

## 7. 根拠ファイル

- [기존 아키텍처 설명](./k3d-architecture.md)
- [Kubernetes README](../../emoselfie-INFRA/k8s/README.md)
- [backend Deployment・Service](../../emoselfie-INFRA/k8s/base/backend.yaml)
- [共通 ConfigMap](../../emoselfie-INFRA/k8s/base/kustomization.yaml)
- [local overlay](../../emoselfie-INFRA/k8s/overlays/local/kustomization.yaml) / [prod overlay](../../emoselfie-INFRA/k8s/overlays/prod/kustomization.yaml)
- [BE Dockerfile](../../emoselfie-BE/Dockerfile) / [BE CI](../../emoselfie-BE/.github/workflows/ci.yml)
- [FE Dockerfile](../../emoselfie-FE/Dockerfile) / [FE CI](../../emoselfie-FE/.github/workflows/ci.yml)
- [Ansible runtime overlay](../../emoselfie-INFRA/ansible/templates/runtime-kustomization.yaml.j2)
- [상태 엔드포인트](../../emoselfie-BE/app/api/health.py) / [의존 서비스 확인](../../emoselfie-BE/app/core/resources.py)

다른 저장소로 향하는 상대 링크는 BE·FE·INFRA·DOCS를 같은 상위 디렉터리에 배치한 구성을 전제로 한다.
