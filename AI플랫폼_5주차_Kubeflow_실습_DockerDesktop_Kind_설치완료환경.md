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
- 전처리 → 학습 → 평가 ML Pipeline 작성
- Pipeline을 KFP v2 IR YAML로 컴파일
- Kubeflow Pipelines UI에서 Pipeline 실행
- DAG, Log, Artifact, Metrics 확인
- 여러 Run을 Experiment에서 비교
- KFP Task와 Kubernetes Pod의 관계 확인

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

```powershell
docker version
```

추가 확인.

```powershell
docker info
```

### 확인사항

- Docker Desktop 실행 여부
- Docker Server 정보 정상 출력 여부

---

# 7. 실습 2 — 현재 Kubernetes Context 확인

```powershell
kubectl config current-context
```

Docker Desktop Kubernetes를 사용하고 있다면 일반적으로 다음 Context가 나타남.

```text
docker-desktop
```

전체 Context 확인.

```powershell
kubectl config get-contexts
```

필요한 경우 Docker Desktop Context로 변경.

```powershell
kubectl config use-context docker-desktop
```

---

# 8. 실습 3 — Kubernetes Node 확인

```powershell
kubectl get nodes
```

예:

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   ...   v1.xx.x
```

상세 확인.

```powershell
kubectl get nodes -o wide
```

### 확인사항

`STATUS`가 `Ready`인지 확인.

---

# 9. 실습 4 — Kubernetes Server Version 확인

```powershell
kubectl version
```

또는:

```powershell
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

```powershell
kubectl get storageclass
```

축약형:

```powershell
kubectl get sc
```

확인사항:

- 기본 StorageClass 존재 여부
- `(default)` 표시 여부

실제 StorageClass 이름은 Docker Desktop 버전에 따라 다를 수 있음.

---

# 11. 실습 6 — 현재 Kubernetes 상태 점검

```powershell
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

```powershell
git --version
```

Git이 없다면 Git for Windows 설치 필요.

---

# 14. 실습 8 — Kustomize 확인

```powershell
kustomize version
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

```powershell
cd C:\workspace
```

다운로드.

```powershell
git clone --branch 26.03.1 https://github.com/kubeflow/community-distribution.git
```

이동.

```powershell
cd community-distribution
```

버전 확인.

```powershell
git describe --tags
```

예상:

```text
26.03.1
```

---

# 16. 설치 전 Context 재확인

```powershell
kubectl config current-context
kubectl get nodes
```

Docker Desktop Kubernetes를 사용할 경우 현재 Context가 `docker-desktop`인지 확인.

---

# 17. 실습 10 — Kubeflow 설치

현재 Docker Desktop Kubernetes Cluster에 Kubeflow 전체 Manifest 적용.

```powershell
kustomize build example | kubectl apply --server-side --force-conflicts -f -
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

```powershell
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

```powershell
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

```powershell
kubectl get pods -A
```

Kubeflow Namespace만 확인.

```powershell
kubectl get pods -n kubeflow
```

실시간 확인.

```powershell
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

```powershell
kubectl get deploy -n kubeflow
```

Service:

```powershell
kubectl get svc -n kubeflow
```

Pipeline 관련 Pod:

```powershell
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

```powershell
kubectl get svc -n istio-system
```

`istio-ingressgateway` Service 확인.

---

# 25. 실습 15 — Kubeflow Dashboard 접속

Port Forward 실행.

```powershell
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

```powershell
kubectl get pods -A
```

Notebook 이름 검색.

```powershell
kubectl get pods -A | Select-String kfp-lab
```

---

# 32. 실습 18 — KFP SDK 설치

JupyterLab Terminal에서:

```bash
python --version
pip show kfp
```

설치:

```bash
pip install "kfp==2.16.1"
```

확인:

```bash
python -c "import kfp; print(kfp.__version__)"
```

예상:

```text
2.16.1
```

---

# 33. 실습 19 — Pipeline Notebook 생성

새 Notebook:

```text
simple_ml_pipeline.ipynb
```

첫 Cell:

```python
import kfp
from kfp import dsl
from kfp import compiler

print("KFP Version:", kfp.__version__)
```

---

# 34. Component란?

Component는 Pipeline의 최소 작업 단위.

```text
Pipeline
   │
   ├─ preprocess
   ├─ train
   └─ evaluate
```

---

# 35. Artifact Type 불러오기

```python
from kfp.dsl import Dataset, Model, Metrics, Input, Output
```

| Type | 의미 |
|---|---|
| Dataset | 데이터셋 |
| Model | 학습된 모델 |
| Metrics | 평가 지표 |
| Artifact | 일반적인 산출물 |

---

# 36. 실습 20 — 전처리 Component 작성

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.3.2",
        "scikit-learn==1.7.1",
    ],
)
def preprocess(output_data: Output[Dataset]):
    import pandas as pd
    from sklearn.datasets import load_iris

    iris = load_iris(as_frame=True)
    df = iris.frame

    df = df.rename(columns={"target": "label"})
    df = df.dropna()

    df.to_csv(output_data.path, index=False)

    print("전처리 완료")
    print("Dataset shape:", df.shape)
    print(df.head())
```

---

# 37. preprocess Component 분석

Component 선언:

```python
@dsl.component
```

Base Image:

```python
base_image="python:3.11-slim"
```

필요 Package:

```python
packages_to_install=[
    "pandas==2.3.2",
    "scikit-learn==1.7.1",
]
```

Output Artifact:

```python
output_data: Output[Dataset]
```

저장:

```python
df.to_csv(output_data.path, index=False)
```

---

# 38. 3주차 Container와 연결

```text
3주차

Dockerfile
   │
   ▼
Container Image
   │
   ▼
Container
```

```text
5주차

KFP Component
   ├─ base_image
   ├─ Python Code
   └─ dependencies
          │
          ▼
     Container Task
```

---

# 39. 실습 21 — 학습 Component 작성

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.3.2",
        "scikit-learn==1.7.1",
        "joblib==1.5.2",
    ],
)
def train(
    input_data: Input[Dataset],
    output_model: Output[Model],
    c_value: float = 1.0,
):
    import pandas as pd
    import joblib
    from sklearn.linear_model import LogisticRegression

    df = pd.read_csv(input_data.path)

    X = df.drop("label", axis=1)
    y = df["label"]

    model = LogisticRegression(
        C=c_value,
        max_iter=300,
    )

    model.fit(X, y)

    joblib.dump(model, output_model.path)

    print("모델 학습 완료")
    print("C =", c_value)
```

---

# 40. Parameter와 Artifact

Parameter:

```python
c_value: float = 1.0
```

Artifact:

```python
input_data: Input[Dataset]
output_model: Output[Model]
```

| 구분 | Parameter | Artifact |
|---|---|---|
| 예 | `C=1.0` | Dataset, Model |
| 크기 | 작은 값 | 상대적으로 큰 파일 |
| 전달 | 값 자체 | Artifact Storage |
| 용도 | 설정값 | 데이터/모델 |

---

# 41. 실습 22 — 평가 Component 작성

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.3.2",
        "scikit-learn==1.7.1",
        "joblib==1.5.2",
    ],
)
def evaluate(
    input_model: Input[Model],
    input_data: Input[Dataset],
    metrics: Output[Metrics],
) -> float:
    import pandas as pd
    import joblib

    model = joblib.load(input_model.path)
    df = pd.read_csv(input_data.path)

    X = df.drop("label", axis=1)
    y = df["label"]

    accuracy = float(model.score(X, y))

    metrics.log_metric("accuracy", accuracy)

    print(f"Accuracy: {accuracy:.4f}")

    return accuracy
```

---

# 42. Metrics Artifact

단순 출력:

```python
print(accuracy)
```

Metrics Artifact:

```python
metrics.log_metric("accuracy", accuracy)
```

Metrics를 사용하면 평가 결과를 Pipeline Metadata와 연결 가능.

---

# 43. 현재 Component 구조

```text
preprocess
Output: Dataset
      │
      ▼
train
Input : Dataset
Output: Model
      │
      ▼
evaluate
Input : Dataset + Model
Output: Accuracy + Metrics
```

---

# 44. 실습 23 — Pipeline 정의

```python
@dsl.pipeline(
    name="simple-ml-pipeline",
    description="Iris 데이터 전처리-학습-평가 파이프라인",
)
def simple_pipeline(c_value: float = 1.0):

    prep = preprocess()

    trained = train(
        input_data=prep.outputs["output_data"],
        c_value=c_value,
    )

    evaluate(
        input_model=trained.outputs["output_model"],
        input_data=prep.outputs["output_data"],
    )
```

---

# 45. Pipeline 코드의 의미

```text
preprocess
    │
    │ Dataset
    ▼
  train
    │
    │ Model
    ▼
evaluate
```

데이터 의존관계에 따라 실행 순서가 결정됨.

---

# 46. DAG란?

```text
Directed Acyclic Graph
```

한국어:

```text
방향성 비순환 그래프
```

예:

```text
A → B → C
```

순환 구조 없음.

---

# 47. 실습 24 — Pipeline 컴파일

```python
compiler.Compiler().compile(
    pipeline_func=simple_pipeline,
    package_path="simple_pipeline.yaml",
)
```

생성:

```text
simple_pipeline.yaml
```

---

# 48. KFP v2 IR YAML 의미

```text
Python DSL
   │
   ▼
KFP Compiler
   │
   ▼
IR YAML
   │
   ▼
KFP Backend
```

4주차에서는 Deployment YAML을 직접 작성했지만 5주차에서는 Python Pipeline을 Compiler가 IR YAML로 변환함.

---

# 49. 실습 25 — YAML 확인

JupyterLab Terminal:

```bash
head -n 40 simple_pipeline.yaml
```

또는 Notebook:

```python
with open("simple_pipeline.yaml", "r") as f:
    for _ in range(40):
        line = f.readline()
        if not line:
            break
        print(line, end="")
```

---

# 50. 실습 26 — Pipeline YAML 다운로드

JupyterLab File Browser에서:

```text
simple_pipeline.yaml
```

선택 후 다운로드.

---

# 51. 실습 27 — Pipelines 메뉴 이동

Kubeflow Dashboard에서 `Pipelines` 선택.

---

# 52. 실습 28 — Pipeline 업로드

`Upload Pipeline` 선택.

Pipeline Name:

```text
simple-ml-pipeline
```

파일:

```text
simple_pipeline.yaml
```

---

# 53. Pipeline / Run / Experiment

| 개념 | 의미 |
|---|---|
| Component | 하나의 작업 정의 |
| Pipeline | 여러 Component를 연결한 Workflow |
| Run | Pipeline의 1회 실행 |
| Experiment | 여러 Run을 그룹화 |

---

# 54. 실습 29 — Experiment 생성

```text
Experiments
   ↓
Create Experiment
```

이름:

```text
iris-experiment
```

---

# 55. 실습 30 — 첫 Run 실행

Pipeline 선택 → `Create Run`.

Run Name:

```text
iris-run-c1
```

Experiment:

```text
iris-experiment
```

Parameter:

```text
c_value = 1.0
```

실행.

---

# 56. 실습 31 — DAG 시각화 확인

```text
preprocess
    │
    ▼
  train
    │
    ▼
evaluate
```

상태:

```text
Pending
  ↓
Running
  ↓
Succeeded
```

---

# 57. 실습 32 — Kubernetes Pod 확인

Windows Terminal에서:

```powershell
kubectl get pods -A
```

실시간:

```powershell
kubectl get pods -A -w
```

Pipeline 실행 시 Task Pod 생성 확인.

---

# 58. KFP Task와 Kubernetes 관계

```text
Pipeline
   │
   ▼
Task
   │
   ▼
Container
   │
   ▼
Kubernetes Pod
```

KFP는 Kubernetes를 대체하지 않고 Kubernetes 위에 ML Workflow 계층을 제공함.

---

# 59. Component와 Task 차이

Component:

```text
작업 정의
```

Task:

```text
Pipeline 안에서 Component를 호출한 실행 단위
```

예:

```python
@dsl.component
def train(...):
    ...
```

→ Component

```python
trained = train(...)
```

→ Task

---

# 60. 실습 33 — preprocess Log 확인

Run DAG에서 `preprocess` 선택.

예:

```text
전처리 완료
Dataset shape: (150, 5)
```

---

# 61. 실습 34 — train Log 확인

`train` 선택.

예:

```text
모델 학습 완료
C = 1.0
```

---

# 62. 실습 35 — evaluate Log 확인

`evaluate` 선택.

예:

```text
Accuracy: 0.xxxx
```

---

# 63. 실습 36 — Artifact 확인

```text
preprocess → Dataset
train      → Model
evaluate   → Metrics
```

전체 흐름:

```text
Dataset
   │
   ▼
 train
   │
   ▼
 Model
   │
   ▼
evaluate
   │
   ▼
Metrics
```

---

# 64. Artifact란?

Pipeline 실행 중 생성되는 데이터/파일 형태 산출물.

예:

- Dataset
- CSV
- Model
- Evaluation Report
- Metrics
- TensorBoard Log

---

# 65. ML Metadata와 Lineage

```text
Dataset
   │
   ▼
preprocess execution
   │
   ▼
Dataset
   │
   ▼
train execution
   │
   ▼
Model
   │
   ▼
evaluate execution
   │
   ▼
Metrics
```

Lineage는 어떤 데이터와 실행을 거쳐 모델이 생성되었는지를 나타내는 계보임.

---

# 66. 실습 37 — Artifacts / Executions 확인

Dashboard에서:

```text
Artifacts
```

확인:

- Dataset
- Model
- Metrics

다음:

```text
Executions
```

확인:

- Component / Task 실행 이력

---

# 67. 실습 38 — 두 번째 Run

Parameter:

```text
c_value = 0.1
```

Run:

```text
iris-run-c01
```

Experiment:

```text
iris-experiment
```

---

# 68. 실습 39 — 세 번째 Run

Parameter:

```text
c_value = 10.0
```

Run:

```text
iris-run-c10
```

Experiment:

```text
iris-experiment
```

---

# 69. 실습 40 — Experiment 비교

```text
iris-experiment

├─ iris-run-c01   C=0.1
├─ iris-run-c1    C=1.0
└─ iris-run-c10   C=10.0
```

비교:

- Parameter
- 실행 시간
- 성공/실패
- Accuracy
- Metrics
- Logs

---

# 70. 5주차와 Katib 연결

5주차:

```text
사람이 Parameter 변경
C=0.1
C=1.0
C=10.0
```

6주차:

```text
Search Space
    │
    ▼
Katib
    │
    ├─ Trial 1
    ├─ Trial 2
    ├─ Trial 3
    └─ ...
```

---

# 71. 5주차와 KServe 연결

```text
preprocess
   │
   ▼
train
   │
   ▼
evaluate
   │
   ▼
KServe
   │
   ▼
Model Serving
```

---

# 72. 실습 41 — Pipeline 실패 실습

전처리 코드를 일부러 변경.

```python
df = pd.read_csv("/not-exist.csv")
```

다시 컴파일.

```python
compiler.Compiler().compile(
    pipeline_func=simple_pipeline,
    package_path="simple_pipeline_error.yaml",
)
```

업로드 후 실행.

---

# 73. 실패 결과 확인

예상:

```text
preprocess → Failed
train      → 실행되지 않음
evaluate   → 실행되지 않음
```

DAG Data Dependency 때문에 앞 단계 실패 시 뒤 단계가 실행되지 않음.

---

# 74. 실습 42 — 실패 Log 확인

실패한 `preprocess` Task의 Logs 확인.

예:

```text
FileNotFoundError
```

---

# 75. Kubernetes에서도 오류 확인

```powershell
kubectl get pods -A
```

상세:

```powershell
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

Log:

```powershell
kubectl logs <POD_NAME> -n <NAMESPACE>
```

---

# 76. 실습 43 — 정상 코드 복구

정상 코드로 복구 후 재컴파일.

```python
compiler.Compiler().compile(
    pipeline_func=simple_pipeline,
    package_path="simple_pipeline.yaml",
)
```

---

# 77. Pipeline의 4대 이점

## 재현성

동일 Pipeline 정의로 동일 실행 흐름 유지.

## 자동화

```text
Run
 ↓
preprocess
 ↓
train
 ↓
evaluate
```

## 재사용성

Component를 다른 Pipeline에서도 재사용 가능.

## 추적성

```text
Run
 ├─ Parameter
 ├─ Log
 ├─ Artifact
 ├─ Metrics
 └─ Metadata
```

---

# 78. 전체 KFP 구조

```text
사용자
  │
  ▼
Python SDK
  │
  ▼
Component / Pipeline
  │
  ▼
KFP Compiler
  │
  ▼
IR YAML
  │
  ▼
KFP Backend
  │
  ▼
Docker Desktop Kubernetes
  │
  ▼
Container Task / Pod

실행 결과
  ├─ Run
  ├─ Log
  ├─ Artifact
  ├─ Metrics
  └─ Metadata
```

---

# 79. 3주차·4주차·5주차 연결

```text
3주차
Container
   │
   ▼
4주차
Kubernetes Pod
   │
   ▼
5주차
KFP Component / Task
   │
   ▼
6주차
Katib / KServe
```

---

# 80. 전체 Pipeline 코드

```python
import kfp
from kfp import dsl
from kfp import compiler
from kfp.dsl import Dataset, Model, Metrics, Input, Output

print("KFP Version:", kfp.__version__)
```

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.3.2",
        "scikit-learn==1.7.1",
    ],
)
def preprocess(output_data: Output[Dataset]):
    import pandas as pd
    from sklearn.datasets import load_iris

    iris = load_iris(as_frame=True)
    df = iris.frame
    df = df.rename(columns={"target": "label"})
    df = df.dropna()

    df.to_csv(output_data.path, index=False)

    print("전처리 완료")
    print("Dataset shape:", df.shape)
```

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.3.2",
        "scikit-learn==1.7.1",
        "joblib==1.5.2",
    ],
)
def train(
    input_data: Input[Dataset],
    output_model: Output[Model],
    c_value: float = 1.0,
):
    import pandas as pd
    import joblib
    from sklearn.linear_model import LogisticRegression

    df = pd.read_csv(input_data.path)

    X = df.drop("label", axis=1)
    y = df["label"]

    model = LogisticRegression(
        C=c_value,
        max_iter=300,
    )

    model.fit(X, y)

    joblib.dump(model, output_model.path)

    print("모델 학습 완료")
    print("C =", c_value)
```

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.3.2",
        "scikit-learn==1.7.1",
        "joblib==1.5.2",
    ],
)
def evaluate(
    input_model: Input[Model],
    input_data: Input[Dataset],
    metrics: Output[Metrics],
) -> float:
    import pandas as pd
    import joblib

    model = joblib.load(input_model.path)
    df = pd.read_csv(input_data.path)

    X = df.drop("label", axis=1)
    y = df["label"]

    accuracy = float(model.score(X, y))

    metrics.log_metric("accuracy", accuracy)

    print(f"Accuracy: {accuracy:.4f}")

    return accuracy
```

```python
@dsl.pipeline(
    name="simple-ml-pipeline",
    description="Iris 데이터 전처리-학습-평가 파이프라인",
)
def simple_pipeline(c_value: float = 1.0):

    prep = preprocess()

    trained = train(
        input_data=prep.outputs["output_data"],
        c_value=c_value,
    )

    evaluate(
        input_model=trained.outputs["output_model"],
        input_data=prep.outputs["output_data"],
    )
```

```python
compiler.Compiler().compile(
    pipeline_func=simple_pipeline,
    package_path="simple_pipeline.yaml",
)

print("Pipeline compile 완료")
```

---

# 81. 자주 사용하는 명령어

| 목적 | 명령 |
|---|---|
| Context 확인 | `kubectl config current-context` |
| Context 목록 | `kubectl config get-contexts` |
| Node 확인 | `kubectl get nodes` |
| Kubernetes 버전 | `kubectl version` |
| StorageClass | `kubectl get sc` |
| Namespace | `kubectl get ns` |
| 전체 Pod | `kubectl get pods -A` |
| Kubeflow Pod | `kubectl get pods -n kubeflow` |
| Kubeflow Service | `kubectl get svc -n kubeflow` |
| 실시간 Pod | `kubectl get pods -A -w` |
| Pod 상세 | `kubectl describe pod POD -n NAMESPACE` |
| Pod Log | `kubectl logs POD -n NAMESPACE` |
| KFP 버전 | `python -c "import kfp; print(kfp.__version__)"` |

---

# 82. 주요 오류 대응

## 잘못된 Context

```powershell
kubectl config current-context
kubectl config use-context docker-desktop
kubectl get nodes
```

## Kubernetes 버전 불일치

```powershell
kubectl version
```

Kubeflow 26.03.1은 Kubernetes 1.35+ 기준으로 사용.

## Pod Pending

```powershell
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

확인:

- CPU
- Memory
- PVC
- StorageClass
- Scheduling

## PVC Pending

```powershell
kubectl get pvc -A
kubectl get sc
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
```

## ImagePullBackOff

```powershell
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

## CrashLoopBackOff

```powershell
kubectl logs <POD_NAME> -n <NAMESPACE>
kubectl logs <POD_NAME> -n <NAMESPACE> --previous
```

---

# 83. 초급 실습에서 UI Upload 방식을 사용하는 이유

SDK에서 KFP API에 직접 연결하려면 환경에 따라 다음 구성이 필요함.

- Authentication
- ServiceAccount Token
- Namespace
- KFP API Endpoint

따라서 이번 실습에서는 다음 방식 사용.

```text
Notebook
   │
   ▼
Compile
   │
   ▼
IR YAML
   │
   ▼
Kubeflow UI Upload
   │
   ▼
Create Run
```

---

# 84. 최종 확인 문제

## 문제 1

이번 실습에서 별도의 Kind Cluster를 만드는가?

### 정답

아님. Docker Desktop에서 이미 생성된 Kubernetes Cluster를 그대로 사용함.

## 문제 2

Kubeflow 설치 전에 확인해야 할 것은?

### 정답

- 현재 kubectl Context
- Node 상태
- Kubernetes Server Version
- StorageClass
- Docker Desktop Resource

## 문제 3

Component와 Pipeline의 차이는?

### 정답

Component는 하나의 작업 정의이고 Pipeline은 여러 Component를 연결한 Workflow임.

## 문제 4

Pipeline 실행 흐름을 표현하는 그래프는?

### 정답

DAG.

## 문제 5

Dataset과 Model 파일을 Component 사이에서 전달할 때 사용하는 개념은?

### 정답

Artifact.

## 문제 6

Pipeline 1회 실행은?

### 정답

Run.

## 문제 7

여러 Run을 묶는 단위는?

### 정답

Experiment.

## 문제 8

KFP v2에서 Pipeline을 컴파일하면 무엇이 생성되는가?

### 정답

KFP v2 IR YAML.

## 문제 9

KFP Task와 Kubernetes의 관계는?

### 정답

Pipeline Task는 Kubernetes 환경에서 Container/Pod 기반 작업으로 실행됨.

---

# 85. 오늘 실습 핵심

```text
[이미 준비됨]

Docker Desktop
      │
      ▼
Kubernetes Cluster
      │
      ▼
kubectl

[이번 실습]

Kubeflow
      │
      ▼
Central Dashboard
      │
      ▼
Notebook
      │
      ▼
KFP Component
      │
      ├─ preprocess
      ├─ train
      └─ evaluate
      │
      ▼
Pipeline
      │
      ▼
DAG
      │
      ▼
IR YAML
      │
      ▼
Run
      │
      ├─ Log
      ├─ Artifact
      ├─ Metrics
      └─ Metadata
      │
      ▼
Experiment
```

---

# 86. 반드시 기억할 한 문장

> **Kubeflow Pipelines는 Kubernetes 위에서 데이터 전처리, 학습, 평가 등의 ML 작업을 Component로 정의하고 DAG로 연결하여 자동 실행·추적하는 ML Workflow Orchestration 도구임.**

---

# 87. 다음 주차 연결

```text
5주차

preprocess
   │
   ▼
train
   │
   ▼
evaluate
```

```text
6주차

Katib
   │
   ▼
Hyperparameter Tuning
   │
   ▼
train
   │
   ▼
evaluate
   │
   ▼
KServe
   │
   ▼
Model Serving
```

---

# 88. 강의 내용과 실습 대응표

| 강의 내용 | 실습 |
|---|---|
| Pipeline 필요성 | 전처리→학습→평가 자동화 |
| DAG | KFP Run DAG |
| 재현성 | 동일 Pipeline 반복 Run |
| 자동화 | 데이터 의존관계 기반 실행 |
| 재사용성 | Component 함수 분리 |
| 추적성 | Run / Artifact / Metrics / Log |
| Component | preprocess / train / evaluate |
| Pipeline | `simple_pipeline()` |
| Run | `iris-run-*` |
| Experiment | `iris-experiment` |
| Artifact | Dataset / Model / Metrics |
| ML Metadata | Artifacts / Executions |
| Notebook | Kubeflow Jupyter 환경 |
| Kubernetes 연결 | Task Pod 관찰 |
| Container 연결 | `base_image` |
| 6주차 연결 | Katib / KServe |

---

# 89. 참고문헌 및 공식 문서

- Kubeflow Documentation  
  https://www.kubeflow.org/docs/

- Installing Kubeflow  
  https://www.kubeflow.org/docs/started/installing-kubeflow/

- Kubeflow Community Distribution  
  https://github.com/kubeflow/community-distribution

- Kubeflow 26.03 Release  
  https://www.kubeflow.org/docs/kubeflow-distribution/releases/kubeflow-26.03/

- Kubeflow Pipelines  
  https://www.kubeflow.org/docs/components/pipelines/

- KFP Getting Started  
  https://www.kubeflow.org/docs/components/pipelines/getting-started/

- KFP Components  
  https://www.kubeflow.org/docs/components/pipelines/user-guides/components/

- KFP Artifacts  
  https://www.kubeflow.org/docs/components/pipelines/user-guides/data-handling/artifacts/

- KFP SDK 2.16.1  
  https://pypi.org/project/kfp/2.16.1/

- Kustomize Releases  
  https://github.com/kubernetes-sigs/kustomize/releases

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
      ↓
11. KFP SDK 설치
      ↓
12. preprocess Component
      ↓
13. train Component
      ↓
14. evaluate Component
      ↓
15. Pipeline 정의
      ↓
16. Compile
      ↓
17. YAML Upload
      ↓
18. Experiment 생성
      ↓
19. Run 실행
      ↓
20. DAG / Log / Artifact / Metrics 확인
```
