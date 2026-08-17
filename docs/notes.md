# The Path a request takes

## PART 1 - Kubernetes가 하는 실제로 하는 일

Kubernetes는 "지금 상태가 어때야 하는지" 를 계속 선언하면, 컨트롤러들이 그 선언과 실제 상태의 차이를 끊임없이 관찰하고 좁혀나가는 시스템이다. 이 "선언 -> 관찰 -> 조정" 의 반복(reconciliation loop)이 이 프로젝트에 있는 모든 YAML 파일의 근본 동작 원리이다.

클러스트는 크게 두 부분이다. 컨트롤 플레인(API 서버, 상태 저장소 etcd, 스케줄러, 컨트롤러 메니저)이 "무엇이 어떻게 떠야 하는지" 결정하고, 노드들(kubelet, 컨네이터 런타임, kube-proxy)이 그 결정을 실제로 실행한다. 우리가 `kubectl apply`로 YAML을 넣으면, API 서버가 그걸 etcd에 저장하고, 관련 컨트롤러들이 "이 선언이랑 지금 상태가 다르다"는 것을 알아채서 행동을 취하는 식이다.

### Pod - 가장 작은 배포 단위

Pod는 컨테이너 하나가 아니라 "함께 뜨고 죽는 컨테이너들의 묶음"이다. 같은 Pod 안의 컨테이너들은 네트워크 네임스페이스를 공유해서, 서로를 localhost 로 부를 수 있고 IP도 하나만 배정 받는다. Insighton의 서비스들은 전부 컨테이너 1개짜리 Pod지만, 이 구조 덕분에 나중에 사이드카를 붙이기가 쉽다.

### Deployment - 상태를 선언하고 맡기는 방식

`infra/ai/deployment.yaml` 같은 파일에 `replicas: 2`라고 써두면, Deployment 컨트롤러가 내부적으로 ReplicaSet을 만들고, 지금 이 라벨 (`app: insighton-ai`)을 가진 Pod가 2개 떠 있어야 하는데 실제도 몇 개 인지를 계속해서 확인함. Pod가 죽으면 즉시 새로 만들고, 이미지 태그가 바뀌면 새 ReplicaSet을 만들고 점진적으로 옛것에서 새것으로 트래픽을 옮긴다(rolling update).

"컨테이너를 띄워라" 라고 명령한 적이 없고, 이 상태여야 한다고 선언만 했을 뿐임. 그 뒤는 컨트롤러의 관찰-조정 루프가 알아서 한다.

### StatefulSet - Pod가 서로 달라야 할 때

Deployment의 전제는 "모든 Pod는 완전히 교체 가능하다"이다. 근데 core와 ruleEngine은 이 전제가 깨진다. 두 인스턴스가 서로 다른 설정(어떤 게이트웨이/큐를 담당항지)을 가져야 한다. 그래서 이 둘만 `kind: StatefulSet`이다.

StatefulSet은 세 가지를 보장한다.

- Pod 이름이 `insighton-core-0,`, `insighton-core-1`처럼 고정된 서수를 가진다.
- 항상 0번부터 순서대로 뜨고 역순으로 내려간다.
- 재시작해도 같은 서수를 유지한다. 

우리는 이 서수를 `RuleEngineInstanceEnvironmentPostProcessor` / `InstanceIndexEnvironmentPostProcessor`가 `$HOSTNAME`에서 읽어다가, 0번은 짝수 큐 + engine-a, 1번은 홀수 큐 + engine-b로 자동 배정하는데 사용한다.

> SatefulSet은 `spec.serviceName`으로 headless Service를 하나 요구함. 그 DNS 기능(개별 Pod를 이름으로 직접 찾는 것)을 실제로 사용하지 않지만, API 요구사항으로 채우기 위해 `insighton-core-headless`, `insighton-ruleengine-headless`를 각각 만들어 두었다.

### Service - 안 바뀌는 주소

Pod IP는 Pod가 재시작될 때마다 바뀐다. Service는 그 위에 절대 안 바뀌는 가상 IP(ClusterIP)를 하나 씌워주는 오브젝트이다. Service를 만들면 `kube-proxy`가 클러스터의 모든 노드에 iptables(또는 IPVS) 규칙을 심어둔다. "ClusterIP 로 오는 패킷의 목적지 주소를, 지금 살아있는 Pod IP 중 하나로 바꿔치기(DNAT) 해라" 라는 규칙이다. 즉, 실제로 트래픽을 전달하는 프록시 프로세스가 어딘가로 떠서 패킷을 중계하는 것이 아니라, 커널 레벨에서 패킷의 목적지 주소 자체가 바뀌는 것임.

~~~ str
																	 
                        패킷 도착	  																	 목적지 재작성
dst: 10.x (ClusterIP) ---------->				 				Node								------------> dst: 10.244.2.7 (Pod)
																	(kube-proxy iptables DNAT 규칙 적용) 

~~~

ClusterIP로 향하던 패킷이 노드를 지나며 목적지 주소가 실제 Pod IP로 재작성된다. - 별도 프록시 프로세서가 중계하는 것이 아니라 커널의 netfilter거 하는 일이다.

Service가 어떤 Pod를 대상으로 삼을지는 **라벨 셀렉터**로 정한다. `infra/core/service.yaml`의 `selector: {app: insighton-core}`는 그 라벨을 가진 Pod라면 뭐든 --Service와 Pod 사이에는 직접적인 참조가 없고, 라벨이라는 느슨한 연결만 있다. 그래서 Pod가 죽고 새로 뜨며 이름이 바뀌어도 Service는 신경 쓸 필요가 없다.

### Headless Service - 가상 IP를 일부러 안 만드는 경우

`clusterIP: None`은 방금 설명한 DNAT 단계 자체를 건너뛴다. 대신 DNS 조회 시 StatefulSet과 묶이면 `insighton-core-0`, `insighton-core-headless...` 처럼 Pod별 개별 DNS 이름까지 생기는데, 여기에서는 이 기능 자체를 사용하지 않는다 - StatefulSet이 `serviceName` 을 강제로 요구해서 채워둔 것이다.

### Namespace - 격리가 아니라 구독

`infra/namespace.yaml`이 정의하는 insighton Namespace는 같은 클러스터, 같은 노드, 같은 네트워크를 공유하는 하나의 구획일 뿐이다. 별도의 클러스터나 VM처럼 진짜로 격리된 것이 아니다. 리소스 이름이 Namespace 단위로 스코핑되고 DNS 이름에도 `.insighton.svc.cluster.local` 처럼 네임스페이스가 들어간다.

(그래서 `insighton-core` 라는 이름을 다른 네임스페이스에서도 사용할 수 있다)

### Secret이 정말 지켜주는가?

k8s Secret은 ConfigMap과 메커니즘이 거의 동일하다. 둘 다 그냥 key-value 데이터를 env var나 파일로 마운트 해주는 오브젝트이다. 차이는 Secret의 값은 base6로 인코딩되어 저장된다는 것 뿐이다. 그래서 insighton-core-secret 같은 Secret이 실제로 비밀이려면, Secret 오브젝트를 get/describe 할 수 있는 사람을 RBAC로 제한하고, etcd 자체를 저장 시점 암호화(encryption at rest)해야 한다 - 둘 다 Secret 오브젝트 자체가 보장해주는 것이 아니라 클러스터 운영자가 별도로 설정해야 하는 부분이다.



## PART 2 insighton-k8s-manifasts의 실제 오브젝트 그래프

위 원리들이 실제로 어떻게 조합되었는지, `infra/`밑 폴더 하나하나를 그대로 본다. ArgoCD는 이 폴더 하나 당 Application 하나(`infra/ingress` 만 예외 - 3개 파일을 한 Application이 같이 봄)를 만들어서, git의 내용과 클러스터의 실제 상태를 계속 비교하고 다르면 다시 적용한다. 이것도 결국 **PART 1**에서 언급한 "선언-관찰-조정" 루프를 git까지 한 단계 더 확장한 것일 뿐이다. 단지 선어의 원본이 `kubectl` 명령이 아니라 git 커밋이 된 것이다.

| 서비스      | 종류                     | Secret                       | 비고                                 |
| :---------- | :----------------------- | :--------------------------- | :----------------------------------- |
| ai          | Deployment               | insighton-ai-secret          | DB, Redis, RabbitMQ, Gemini API      |
| auth        | Deployment               | insighton-auth-secret        | DB, Redis, JWT 키, OAuth, 메일       |
| core        | **StatefulSet**          | insighton-core-secret        | 서수 기반 slot-index 자동 주입       |
| ruleengine  | **StatefulSet**          | insighton-ruleengine-secret  | 서수 기반 engine-id/큐분담 자동 주입 |
| gateway     | Deployment               | —                            | JWT 공개키는 properties에 직접 내장  |
| front       | Deployment               | —                            | 외부 상태 없음, 순수 BFF             |
| influxdb    | Deployment (replicas: 1) | insighton-influxdb-secret    | PVC로 영속 저장                      |
| zipkin      | Deployment (replicas: 1) | —                            | 인메모리 저장, 무상태                |
| cloudflared | Deployment               | insighton-cloudflared-secret | PART 4 참고                          |



## PART 3 - ingress-nginx: Ingress는 규칙일 뿐, 동작은 컨트롤러가 한다

`kind: Ingress`는 그 자체로는 아무 일도 하지 않는다. k8s API가 표준화한건 "이 host/path로 오면 이 Service로 보내라"는 최소한의 스펙뿐이고, 그걸 실제로 실행하는 건 별도로 설치한 Ingress 컨트롤러의 몫이다. 이 프로젝트에서 사용하고 있는 `ingrss-nginx`-진짜 NGINX 프로세스 + k8s API를 계속 지켜보다가 Ingress/Service/Secret 이 바뀌면 nginx.conf를 새로 만들어 리로드하는 컨트롤러 루프 조합이다.

이게 중요한 이유는 - NGINX 전용 기능들은 Ingress 스펙 안에 없다는 뜻이다. host/path/backend는 표준이지만 rate limit이나 Basic Auth 같은 nginx라는 특정 구현체만 아는 개념이라, k8s API 자체엔 그걸 넣을 자리가 없다. 그래서 `nginx.ingress.kubernetes.io/...` 같은 **annotion**을 붙이는 것이다 - 다른 컨트롤러(Traefik, HAProxy 등)를 사용하면 이 annotation들은 전부 무시된다.

#### 실제로 사용한 어노테이션

`ssl-redirect`: HTTP로 들어오면 같은 경로로 301 https 리다이렉트

`limit-rps`: IP당 초당 요청 수 상한 (leaky-bucket 방식 rate limit)

`limit-connections`: IP당 동시 연결 수 상한

`proxy-buffer-size`: 백엔드 응답 레더를 담을 버퍼 크기 - 작으면 헤더가 큰 응답(JWT 등)에서 "upsteam sent too big header" 에러

`proxy-connect/send/read-timeout`: nginx -> 백엔드 구간의 연결/전송/응답 대기 타임아웃(초)

`auth-type / auth-secret / auth-realm`: HTTP Basic Auth

#### Basic Auth가 실제로 하는 일

브라우저가 `influxdb.insighton.store`에 처음에 접속하면, 컨트롤러는 `401 + Unauthorized` + `WWW-Authenticate: Basic` 헤더로 응답한다. 브라우저는 이걸 보고 로그인 창을 띄우고, 사용자가 아이디/비번을 입력하면 `Authorization: Basic base64(아이디:비번)` 헤더를 담아 같은 요청을 다시 보낸다. 컨트롤러는 그 값을 `insighton-team-basic-auth` Secret의 auth 키와 대조한다.

`auth`라는 이름이 고정인 이유는 k8s가 정한게 아니라, ingress-nginx 컨트롤러 코드 자체에 그 이름이 하드코딩되어 있기 때문이다 - TLS Secret이 `tls.crt`/`tls.key`를 요구하는 것과 같은 종류의 "고정된 계약"이다.

#### NodePort - 표준 포트가 아닌 이유

`ingress-nginx-controller` Service는 NodePort 타입이다. 이건 "80/443이 아니라 30000~32767 사이의 임의 포트를 클러스터의 모든 노드에 똑같이 열어준다"는 뜻이다. 실제로 어느 노드가 컨트롤러 Pod를 갖고 있는지와 무관하게, 어떤 노드가 컨트롤러의 Pod를 갖고 있는지와 무관하게, 어떤 노드의 IP:NodePort로 접속해서 kube-proxy가 그 노드에서 다시 한번 DNAT을 태워서 진짜 Pod까지 연결해준다.

## PART 4 - cloudflared

NordPort 문제(표준 포트가 안 열림, 공인 IP 관리, 방화벽 설정)는 전부 "외부에서 안으로 들어오는 연결을 어떻게 받을까"라는 전제에서 나온다. Cloudflare Tunnel은 이 전제 자체를 뒤집는다.

일반적인 서버는 포트를 열어두고 기다린다(listen). cloudflared는 반대로, 클러스터 안에서 바깥으로 Cloudflare의 가장 가까운 엣지 데이터센터에 연결을 건다(dial). 한 번 걸리면 그 연결은 끊기지 않고 계속 유지되고(persistent), 그 위에 여러 
