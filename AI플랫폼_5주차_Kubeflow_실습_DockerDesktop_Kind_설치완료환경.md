# AI플랫폼 5주차 실습
## 데이터·ML 파이프라인 — Kubeflow Pipelines

> **실습 전제**  
> 본 실습은 **Docker Desktop에서 Kubernetes 클러스터가 이미 생성되어 있고 정상 동작 중인 상태**에서 시작함.  
> Kind, Minikube 등 별도의 Kubernetes 클러스터는 생성하지 않음.

---

# 1. 실습 목표

- Docker Desktop Kubernetes 클러스터 상태 확인
- Kubeflow Community Distribution 설치
- Kubeflow Central Dashboard 접속
- Kubeflow Notebook 생성
- KFP SDK 설치
- `Component`, `Pipeline`, `DAG`, `Artifact`, `Run`, `Experiment` 개념 확인

---

# 2. 실습 환경

이미 다음 환경이 구성되어 있다고 가정함.

```text
Windows 10/11
   │
   └─ Docker Desktop
         │
         └─ Kubernetes Cluster
               │
               └─ kubectl
```

본 실습에서 추가로 사용하는 도구:

- Git
- Kustomize
- Kubeflow Community Distribution
- Kubeflow Pipelines SDK

---

# 3. 실습 버전 기준

본 자료는 **2026년 9월 기준** 다음 버전을 사용함.

- Kubeflow Community Distribution: `26.03.1`
- Kubernetes: `1.35+`
- Kustomize: `5.8.1`
- Kubeflow Pipelines Runtime: `2.16.1`
- KFP SDK: `2.16.1`

> 주의  
> Docker Desktop Kubernetes Server Version이 너무 낮으면 Kubeflow 설치가 정상적으로 되지 않을 수 있음.

---

# 4. 4주차와 5주차의 연결

## 4주차

Kubernetes Object를 직접 배포함.

```text
kubectl
   │
   ▼
Deployment
   │
   ▼
ReplicaSet
   │
   ▼
Pod
```

## 5주차

ML 작업을 Pipeline으로 구성함.

```text
preprocess
    │
    ▼
  train
    │
    ▼
evaluate
```

각 Pipeline Task는 Kubernetes 환경에서 Container 기반 작업으로 실행됨.

```text
KFP Pipeline
    │
    ├─ preprocess Task
    │      └─ Container / Pod
    │
    ├─ train Task
    │      └─ Container / Pod
    │
    └─ evaluate Task
           └─ Container / Pod
```

---

# 5. KFP v2 구조

```text
Python DSL
   │
   ▼
KFP Compiler
   │
   ▼
KFP v2 IR YAML
   │
   ▼
KFP Backend
   │
   ▼
Kubernetes Runtime
```

> 과거 KFP v1의 Argo Workflow YAML 방식과 구분할 것.

---

# 6. 실습 1 — Docker Desktop 확인

Windows Terminal 실행.

```terminal
docker version
```

추가 확인.

```terminal
docker info
```

### 확인사항

- Docker Desktop 실행 여부
- Docker Server 정보 정상 출력 여부

---

# 7. 실습 2 — 현재 Kubernetes Context 확인

```terminal
kubectl config current-context
```

Docker Desktop Kubernetes를 사용하고 있다면 일반적으로 다음 Context가 나타남.

```text
docker-desktop
```

전체 Context 확인.

```terminal
kubectl config get-contexts
```

필요한 경우 Docker Desktop Context로 변경.

```terminal
kubectl config use-context docker-desktop
```

---

# 8. 실습 3 — Kubernetes Node 확인

```terminal
kubectl get nodes
```

예:

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   ...   v1.xx.x
```

상세 확인.

```terminal
kubectl get nodes -o wide
```

### 확인사항

`STATUS`가 `Ready`인지 확인.

---

# 9. 실습 4 — Kubernetes Server Version 확인

```terminal
kubectl version
```

또는:

```terminal
kubectl version -o yaml
```

확인할 항목:

```text
Client Version
Server Version
```

Kubeflow 26.03.1 실습에서는 Kubernetes Server Version `1.35+` 사용.

---

# 10. 실습 5 — 기본 StorageClass 확인

Kubeflow Notebook과 Pipeline은 PVC를 사용할 수 있으므로 StorageClass 확인.

```terminal
kubectl get storageclass
```

축약형:

```terminal
kubectl get sc
```

확인사항:

- 기본 StorageClass 존재 여부
- `(default)` 표시 여부

실제 StorageClass 이름은 Docker Desktop 버전에 따라 다를 수 있음.

---

# 11. 실습 6 — 현재 Kubernetes 상태 점검

```terminal
kubectl get pods -A
```

다음 상태가 다수 존재하면 먼저 클러스터 상태 확인.

```text
CrashLoopBackOff
ImagePullBackOff
Pending
Error
```

---

# 12. Docker Desktop Resource 확인

Kubeflow 전체 설치 권장 자원:

```text
CPU: 8 Core 이상
Memory: 16 GB 이상
Disk: 충분한 여유공간
```

Docker Desktop의 Resources 설정에서 확인.

> 자원이 부족하면 Pod가 `Pending`, `Evicted`, `CrashLoopBackOff` 상태가 될 수 있음.

---

# 13. 실습 7 — Git 확인

```terminal
git --version
```

Git이 없다면 Git for Windows 설치 필요.

---

# 14. 실습 8 — Kustomize 확인

```terminal
kustomize version
```
```
# Kustomize 5.8.1 설치

curl -s \
  "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" \
  | bash -s -- 5.8.1 /tmp

sudo mv /tmp/kustomize /usr/local/bin/kustomize
sudo chmod +x /usr/local/bin/kustomize
```

권장 버전:

```text
v5.8.1
```

Kustomize가 없다면 공식 Release에서 Windows용 실행 파일 설치.

```text
https://github.com/kubernetes-sigs/kustomize/releases
```

---

# 15. 실습 9 — Kubeflow Community Distribution 다운로드

작업 디렉터리 이동.

```terminal
cd ~
```

다운로드.

```terminal
git clone --branch 26.03.1 https://github.com/kubeflow/community-distribution.git
```

이동.

```terminal
cd community-distribution
```

버전 확인.

```terminal
git describe --tags
```

예상:

```text
26.03.1
```

---

# 16. 설치 전 Context 재확인

```terminal
kubectl config current-context
kubectl get nodes
```

Docker Desktop Kubernetes를 사용할 경우 현재 Context가 `docker-desktop`인지 확인.

---

# 17. 실습 10 — Kubeflow 설치

현재 Docker Desktop Kubernetes Cluster에 Kubeflow 전체 Manifest 적용.

```terminal
while ! kustomize build example | \
kubectl apply --server-side --force-conflicts -f -; do
  echo "Retrying to apply resources"
  sleep 20
done
```

---

# 18. 첫 실행에서 오류가 발생할 수 있는 이유

Kubeflow 설치에는 다음 순서가 필요함.

```text
CRD 생성
   │
   ▼
CRD 등록
   │
   ▼
Custom Resource 생성
```

처음 실행 시 CRD가 아직 준비되지 않아 일부 Resource 생성이 실패할 수 있음.

이 경우 같은 명령을 다시 실행.

```terminal
kustomize build example | kubectl apply --server-side --force-conflicts -f -
```

필요하면 여러 번 반복.

---

# 19. 왜 같은 apply 명령을 반복해도 되는가?

```text
Manifest
   │
   ▼
Desired State
   │
   ▼
kubectl apply
   │
   ▼
Actual State
```

`kubectl apply`는 선언적 방식으로 동작하므로 이미 존재하는 Resource는 갱신되고, 없는 Resource는 생성됨.

---

# 20. 실습 11 — Namespace 확인

```terminal
kubectl get namespaces
```

Kubeflow 관련 Namespace 확인.

예:

```text
kubeflow
kubeflow-system
istio-system
cert-manager
auth
...
```

---

# 21. 실습 12 — 전체 Pod 확인

```terminal
kubectl get pods -A
```

Kubeflow Namespace만 확인.

```terminal
kubectl get pods -n kubeflow
```

실시간 확인.

```terminal
kubectl get pods -n kubeflow -w
```

종료:

```text
Ctrl + C
```

---

# 22. Pod 상태 이해

정상:

```text
Running
Completed
```

설치 과정에서 나타날 수 있음:

```text
Pending
ContainerCreating
PodInitializing
```

문제 가능성이 높은 상태:

```text
CrashLoopBackOff
ImagePullBackOff
Error
```

---

# 23. 실습 13 — Kubeflow 주요 Kubernetes Object 확인

Deployment:

```terminal
kubectl get deploy -n kubeflow
```

Service:

```terminal
kubectl get svc -n kubeflow
```

Pipeline 관련 Pod:

```terminal
kubectl get pods -n kubeflow | Select-String pipeline
```

### 학습 포인트

```text
Kubeflow
  ├─ Deployment
  ├─ Pod
  ├─ Service
  ├─ ConfigMap
  ├─ Secret
  └─ Custom Resource
```

Kubeflow 자체도 Kubernetes Object의 조합으로 구성됨.

---

# 24. 실습 14 — Istio Ingress 확인

```terminal
kubectl get svc -n istio-system
```

`istio-ingressgateway` Service 확인.

---

# 25. 실습 15 — Kubeflow Dashboard 접속

Port Forward 실행.

```terminal
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

해당 Terminal은 그대로 유지.

브라우저:

```text
http://localhost:8080
```

---

# 26. Kubeflow 로그인

기본 실습 계정:

```text
ID: user@example.com
PW: 12341234
```

> 실습용 기본 계정이며 실제 운영 환경에서는 변경 필요.

---

# 27. Kubeflow Dashboard 확인

다음 메뉴 확인.

- Notebooks
- Pipelines
- Experiments
- Runs
- Artifacts
- Executions
- Volumes
- Katib 관련 메뉴
- KServe 관련 메뉴

---

# 28. 실습 16 — Notebook 생성

Kubeflow Dashboard:

```text
Notebooks
   ↓
New Notebook
```

또는 UI 버전에 따라 `New Server`.

Notebook 이름:

```text
kfp-lab
```

---

# 29. Notebook Resource 설정

예:

```text
CPU: 1~2
Memory: 2~4 GiB
Workspace Volume: 10 GiB
```

Python/Jupyter 기반 Notebook Image 선택.

Notebook이 Ready가 되면 `CONNECT` 선택.

---

# 30. Notebook과 Kubernetes 관계

```text
Kubeflow Dashboard
       │
       ▼
Notebook Object
       │
       ▼
Notebook Controller
       │
       ▼
Pod
       │
       ▼
Jupyter Container
```

---

# 31. 실습 17 — Notebook Pod 확인

Windows Terminal 새 탭에서:

```terminal
kubectl get pods -A
```

Notebook 이름 검색.

```terminal
kubectl get pods -A | Select-String kfp-lab
```

---



# 32. 주요 오류 대응

## 잘못된 Context

```terminal
kubectl config current-context
kubectl config use-context docker-desktop
kubectl get nodes
```

## Kubernetes 버전 불일치

```terminal
kubectl version
```

Kubeflow 26.03.1은 Kubernetes 1.35+ 기준으로 사용.

## Pod Pending

```terminal
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

확인:

- CPU
- Memory
- PVC
- StorageClass
- Scheduling

## PVC Pending

```terminal
kubectl get pvc -A
kubectl get sc
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
```

## ImagePullBackOff

```terminal
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

## CrashLoopBackOff

```terminal
kubectl logs <POD_NAME> -n <NAMESPACE>
kubectl logs <POD_NAME> -n <NAMESPACE> --previous
```

---


# 부록 — 수업 진행 권장 순서

```text
1. docker version
      ↓
2. kubectl config current-context
      ↓
3. kubectl get nodes
      ↓
4. kubectl version
      ↓
5. kubectl get sc
      ↓
6. Kubeflow 다운로드
      ↓
7. kustomize build example | kubectl apply
      ↓
8. kubectl get pods -A
      ↓
9. Dashboard 접속
      ↓
10. Notebook 생성
      
```
