# AI플랫폼 5주차 실습
## 데이터·ML 파이프라인 — Kubeflow Pipelines

> **실습 전제**  
> Docker Desktop, `kubectl`, Kind가 이미 설치되어 있고 **Kind Kubernetes Cluster까지 생성되어 정상 동작 중**이라고 가정함.  
> 따라서 Docker Desktop 설치, Kind 설치, Kind Cluster 생성은 본 실습에서 제외함.

---

# 1. 실습 목표

- 기존 Kind Kubernetes Cluster 상태 확인
- 현재 `kubectl` Context가 Kind Cluster를 가리키는지 확인
- Kubeflow Community Distribution 설치
- Kubeflow Central Dashboard 접속
- Kubeflow Notebook Server 생성
- KFP SDK 설치
- Component / Pipeline / DAG / Artifact 개념 확인
- 전처리 → 학습 → 평가 Pipeline 작성
- Pipeline을 KFP v2 IR YAML로 컴파일
- Kubeflow Pipelines UI에서 Run 실행
- DAG / Log / Artifact / Metrics / Experiment 확인
- KFP Task와 Kubernetes Pod의 관계 확인

---

# 2. 4주차와 5주차의 연결

4주차에서는 Kubernetes Object를 직접 생성하고 확인함.

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

5주차에서는 여러 ML 작업을 Pipeline으로 연결함.

```text
preprocess
    │
    ▼
  train
    │
    ▼
evaluate
```

Pipeline의 각 Task는 Kubernetes 환경에서 Container 기반으로 실행됨.

```text
Pipeline
   ├─ preprocess Task → Container / Pod
   ├─ train Task      → Container / Pod
   └─ evaluate Task   → Container / Pod
```

### 핵심 연결

```text
3주차: Container
        ↓
4주차: Kubernetes Pod / Controller
        ↓
5주차: Kubeflow Pipeline Task
```

---

# 3. 실습 버전 기준

본 자료는 2026년 9월 기준 다음 버전을 기준으로 구성함.

- Kubeflow Community Distribution: `26.03.1`
- Kubeflow Pipelines: `2.16.1`
- KFP SDK: `2.16.1`
- Kustomize: `5.8.x` 계열
- Kubernetes: Kubeflow 26.03.1 기준 `1.35+`

> 현재 Kind Cluster의 Kubernetes Server Version이 지원 범위보다 낮으면 Kubeflow 설치가 실패할 수 있으므로 반드시 확인함.

---

# 4. KFP v2에서 주의할 점

KFP v2는 다음 구조로 Pipeline을 처리함.

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

> 과거 KFP v1 설명에서 많이 등장하던 **Argo Workflow YAML을 직접 작성·업로드하는 방식으로 이해하면 안 됨**.  
> 본 실습에서는 KFP v2의 IR YAML을 사용함.

---

# 5. 이미 준비되어 있는 환경

```text
Windows 10/11
   │
   └─ Docker Desktop
        │
        └─ Docker Engine
             │
             └─ Kind
                  │
                  └─ Kubernetes Cluster
```

이미 설치되어 있다고 가정하는 도구:

- Docker Desktop
- Docker CLI
- Kind
- kubectl

이번 실습에서 추가로 필요한 도구:

- Git
- Kustomize
- Kubeflow
- KFP SDK

---

# 6. 실습 1 — Docker Desktop 상태 확인

Windows Terminal에서 PowerShell 실행.

```powershell
docker version
```

추가 확인:

```powershell
docker info
```

### 확인사항

- Docker Client 정보 출력
- Docker Server 정보 출력
- Docker Desktop 실행 상태 정상

---

# 7. 실습 2 — 기존 Kind Cluster 확인

Kind 버전 확인.

```powershell
kind version
```

생성되어 있는 Cluster 확인.

```powershell
kind get clusters
```

예:

```text
kind
```

또는:

```text
kubeflow
```

> Cluster 이름은 실습 환경마다 다를 수 있음.

---

# 8. 실습 3 — kubectl Context 확인

현재 Context:

```powershell
kubectl config current-context
```

전체 Context:

```powershell
kubectl config get-contexts
```

예:

```text
CURRENT   NAME
*         kind-kind
          docker-desktop
```

Kind Context는 일반적으로 다음 형식임.

```text
kind-<CLUSTER_NAME>
```

필요한 경우 Context 변경.

```powershell
kubectl config use-context kind-kind
```

또는 Cluster 이름이 `kubeflow`인 경우:

```powershell
kubectl config use-context kind-kubeflow
```

### 중요

Kubeflow 설치 전에 반드시 **현재 Context가 실습용 Kind Cluster인지 확인**함.

---

# 9. 실습 4 — Kubernetes Cluster 상태 확인

Node 확인.

```powershell
kubectl get nodes
```

상세 확인.

```powershell
kubectl get nodes -o wide
```

정상 상태:

```text
STATUS = Ready
```

---

# 10. 실습 5 — Kubernetes 버전 확인

```powershell
kubectl version
```

또는:

```powershell
kubectl version -o yaml
```

### 확인사항

- Client Version: `kubectl` 버전
- Server Version: 실제 Kind Kubernetes 버전

Kubeflow Community Distribution 26.03.1은 Kubernetes `1.35+` 기준으로 제공됨.

---

# 11. Docker Desktop Resource 확인

Kubeflow 전체 설치는 일반 Kubernetes 기초 실습보다 많은 자원을 사용함.

권장:

```text
CPU    : 8 Core 이상
Memory : 16 GB 이상
Disk   : 충분한 여유공간
```

Docker Desktop에서 Resource 설정 확인.

```text
Docker Desktop
   ↓
Settings
   ↓
Resources
```

자원이 부족할 경우 다음 문제가 발생할 수 있음.

```text
Pending
Evicted
CrashLoopBackOff
```

---

# 12. 실습 6 — Git 확인

```powershell
git --version
```

Git이 설치되어 있지 않다면 Git for Windows 설치 후 진행.

---

# 13. 실습 7 — Kustomize 확인

```powershell
kustomize version
```

권장 예:

```text
v5.8.1
```

Kustomize가 없는 경우 공식 Release에서 Windows AMD64 버전을 설치.

```text
https://github.com/kubernetes-sigs/kustomize/releases
```

예:

```text
kustomize_v5.8.1_windows_amd64.zip
```

압축을 해제한 `kustomize.exe`를 PATH에 포함된 디렉터리에 배치한 후 다시 확인.

```powershell
kustomize version
```

---

# 14. 실습 8 — Kubeflow Community Distribution 다운로드

작업 디렉터리 이동.

예:

```powershell
cd C:\workspace
```

Kubeflow 26.03.1 다운로드.

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

# 15. 설치 직전 최종 확인

```powershell
kubectl config current-context
kubectl get nodes
```

### 체크

- Kind Cluster Context인가?
- Node가 `Ready`인가?
- Kubernetes Server Version이 호환되는가?
- Docker Desktop Resource가 충분한가?

문제가 없다면 Kubeflow 설치 진행.

---

# 16. 실습 9 — Kubeflow 설치

```bash
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do
  echo "Retrying to apply resources"
  sleep 20
done
```

### 참고

Kubeflow는 CRD, Controller, Webhook 등 많은 Resource를 생성함.

CRD가 준비되기 전에 Custom Resource가 적용되면 처음에는 실패할 수 있으므로 반복 적용 방식 사용.

---

# 17. 실습 10 — Namespace 확인

```bash
kubectl get namespaces
```

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

# 18. 실습 11 — Pod 확인

```powershell
kubectl get pods -A
```

Kubeflow Namespace:

```powershell
kubectl get pods -n kubeflow
```

실시간 확인:

```powershell
kubectl get pods -n kubeflow -w
```

정상 상태 예:

```text
Running
Completed
```

문제 상태 예:

```text
Pending
CrashLoopBackOff
ImagePullBackOff
Error
```

---

# 19. 실습 12 — Kubeflow Dashboard 접속

Istio Ingress Gateway 확인:

```bash
kubectl get svc -n istio-system
```

Port Forward:

```bash
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

브라우저 접속:

```text
http://localhost:8080
```

기본 실습 계정:

```text
ID: user@example.com
PW: 12341234
```

> 운영환경에서는 기본 계정과 비밀번호를 반드시 변경해야 함.

---

# 20. Kubeflow 주요 화면 확인

Central Dashboard에서 다음 메뉴 확인.

- Notebooks
- Pipelines
- Experiments
- Runs
- Artifacts
- Executions
- Volumes
- Katib Experiments
- KServe 관련 메뉴

설치 구성에 따라 일부 메뉴 명칭이나 위치가 달라질 수 있음.

---

# 21. 실습 13 — Notebook Server 생성

Dashboard:

```text
Notebooks
   ↓
New Notebook / New Server
```

이름:

```text
kfp-lab
```

권장 설정 예:

```text
CPU: 1~2
Memory: 2~4 GiB
Workspace Volume: 10 GiB
```

Python/Jupyter 기반 Notebook Image 선택.

생성 후:

```text
CONNECT
```

선택.

---

# 22. Notebook과 Kubernetes의 관계

```text
Kubeflow Dashboard
       │
       ▼
Notebook Resource
       │
       ▼
Notebook Controller
       │
       ▼
Pod
       │
       ▼
JupyterLab Container
```

4주차의 Object / Controller 개념과 동일함.

Notebook Pod 확인:

```bash
kubectl get pods -A | grep kfp-lab
```

---

# 23. 실습 14 — KFP SDK 설치

JupyterLab Terminal에서:

```bash
python --version
pip show kfp
```

필요 시 설치:

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

# 24. 실습 15 — Notebook 생성

새 Notebook 파일:

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

# 21. Component란?

Component는 Pipeline의 최소 실행 단위.

```text
Pipeline

preprocess
    │
    ▼
  train
    │
    ▼
evaluate
```

KFP v2에서는 Python 함수를 `@dsl.component`로 정의할 수 있음.

---

# 22. Artifact Type import

```python
from kfp.dsl import Dataset, Model, Metrics, Input, Output
```

| Type | 의미 |
|---|---|
| Dataset | 데이터셋 |
| Model | 학습된 모델 |
| Metrics | 평가 지표 |
| Artifact | 일반 산출물 |

---

# 23. 실습 15 — 전처리 Component

외부 `sample.csv` 의존성을 없애고 Iris Dataset을 Component 내부에서 생성함.

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

# 24. preprocess Component 분석

`@dsl.component`

```text
Python Function
      ↓
KFP Component
```

`base_image`

```python
base_image="python:3.11-slim"
```

Component를 실행할 Container Base Image.

`packages_to_install`

```python
packages_to_install=[
    "pandas==2.3.2",
    "scikit-learn==1.7.1",
]
```

실행에 필요한 Python Package 설치.

`Output[Dataset]`

```python
output_data: Output[Dataset]
```

Dataset Artifact 생성.

저장:

```python
df.to_csv(output_data.path, index=False)
```

---

# 25. 3주차와 연결

3주차:

```text
Dockerfile
   ↓
Container Image
```

5주차:

```text
KFP Component
   ├─ base_image
   ├─ Python Function
   └─ dependencies
          ↓
     Container Task
```

실제 운영에서는 필요한 라이브러리를 미리 포함한 Custom Image 사용이 일반적임.

본 실습에서는 이해를 위해 `packages_to_install` 사용.

---

# 26. 실습 16 — 학습 Component

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

# 27. Input / Output / Parameter

Input Artifact:

```python
input_data: Input[Dataset]
```

Output Artifact:

```python
output_model: Output[Model]
```

Parameter:

```python
c_value: float = 1.0
```

| 구분 | Parameter | Artifact |
|---|---|---|
| 대상 | 숫자, 문자열, Boolean | Dataset, Model, File |
| 예 | `C=1.0` | `model.pkl` |
| 전달 | 값 자체 | 저장소의 산출물 |
| 용도 | 설정값 | 큰 데이터/파일 |

---

# 28. 실습 17 — 평가 Component

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

# 29. Metrics Artifact

단순 `print()`:

```text
Log에서 확인 가능
```

Metrics Artifact:

```python
metrics: Output[Metrics]
```

기록:

```python
metrics.log_metric("accuracy", accuracy)
```

구조:

```text
evaluate
   ├─ stdout log
   ├─ accuracy Parameter
   └─ Metrics Artifact
```

---

# 30. 지금까지의 Component

```text
preprocess
Input : 없음
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

# 31. 실습 18 — Pipeline 정의

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

# 32. Pipeline 코드 분석

1단계:

```python
prep = preprocess()
```

2단계:

```python
trained = train(
    input_data=prep.outputs["output_data"]
)
```

3단계:

```python
evaluate(
    input_model=trained.outputs["output_model"],
    input_data=prep.outputs["output_data"],
)
```

---

# 33. DAG가 자동으로 만들어지는 이유

단순한 Python 코드 작성 순서가 아니라 **데이터 의존관계**에 의해 실행 순서가 결정됨.

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

evaluate는 preprocess의 Dataset도 입력받음.

---

# 34. DAG 개념

DAG:

```text
Directed Acyclic Graph
```

한국어:

```text
방향성 비순환 그래프
```

Directed:

```text
A → B → C
```

Acyclic:

```text
순환 구조 없음
```

---

# 35. 실습 19 — Pipeline 컴파일

```python
from kfp import compiler

compiler.Compiler().compile(
    pipeline_func=simple_pipeline,
    package_path="simple_pipeline.yaml",
)
```

생성 파일:

```text
simple_pipeline.yaml
```

---

# 36. KFP v2 IR YAML

```text
Python DSL
   ↓
KFP Compiler
   ↓
IR YAML
   ↓
KFP Backend
   ↓
Kubernetes Runtime
```

4주차에서는 Deployment YAML을 직접 작성했지만, 5주차에서는 Python Pipeline을 Compiler가 IR YAML로 변환함.

---

# 37. YAML 확인

Notebook Terminal:

```bash
head -n 40 simple_pipeline.yaml
```

또는 Python:

```python
with open("simple_pipeline.yaml", "r") as f:
    for _ in range(40):
        line = f.readline()
        if not line:
            break
        print(line, end="")
```

---

# 38. 4주차와 YAML 비교

4주차:

```text
사용자
  ↓
deployment.yaml 직접 작성
  ↓
kubectl apply
```

5주차:

```text
사용자
  ↓
Python Pipeline
  ↓
KFP Compiler
  ↓
pipeline.yaml
  ↓
KFP Backend
```

---

# 39. 실습 20 — Pipeline YAML 다운로드

JupyterLab File Browser에서:

```text
simple_pipeline.yaml
```

다운로드.

---

# 40. 실습 21 — Pipeline 업로드

Kubeflow Dashboard:

```text
Pipelines
  ↓
Upload Pipeline
```

Pipeline 이름:

```text
simple-ml-pipeline
```

파일:

```text
simple_pipeline.yaml
```

업로드.

---

# 41. 실습 22 — Experiment 생성

Dashboard:

```text
Experiments
   ↓
Create Experiment
```

이름:

```text
iris-experiment
```

구조:

```text
Experiment
   ├─ Run 1
   ├─ Run 2
   └─ Run 3
```

---

# 42. Pipeline / Run / Experiment

| 개념 | 의미 |
|---|---|
| Pipeline | 실행할 ML Workflow 정의 |
| Run | Pipeline 1회 실행 |
| Experiment | 여러 Run을 묶는 그룹 |

예:

```text
simple-ml-pipeline
       ↓
iris-experiment
   ├─ Run(C=0.1)
   ├─ Run(C=1.0)
   └─ Run(C=10.0)
```

---

# 43. 실습 23 — 첫 Run 실행

Pipeline 선택:

```text
Create Run
```

Run 이름:

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

# 44. DAG 시각화 확인

Run 화면에서:

```text
preprocess
    ↓
  train
    ↓
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

# 45. Kubernetes에서 실행 확인

Host Terminal:

```powershell
kubectl get pods -A
```

사용자 Profile Namespace를 알고 있다면:

```bash
kubectl get pods -n <PROFILE_NAMESPACE>
```

실시간:

```bash
kubectl get pods -A -w
```

### 학습 포인트

KFP Component는 실제 Kubernetes 환경에서 Container 기반 Task로 실행됨.

---

# 46. Component와 Task의 차이

Component:

```text
작업의 정의
```

Task:

```text
Pipeline 안에서 Component를 호출해 생성한 실행 단위
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

# 47. 실습 24 — preprocess Log 확인

Run DAG에서 `preprocess` 선택.

예:

```text
전처리 완료
Dataset shape: (150, 5)
```

확인사항:

- 독립적인 Task 실행
- Dataset 생성
- 단계별 Log 확인

---

# 48. 실습 25 — train Log 확인

`train` 선택.

예:

```text
모델 학습 완료
C = 1.0
```

---

# 49. 실습 26 — evaluate Log 확인

`evaluate` 선택.

예:

```text
Accuracy: 0.xxxx
```

정확한 값은 실행 환경과 Library 버전에 따라 달라질 수 있음.

---

# 50. 실패한 Pipeline 디버깅

Task 실패 시:

1. Logs 확인
2. Input 확인
3. Output 확인
4. Artifact 확인
5. Kubernetes Pod 확인
6. Events 확인

Kubernetes:

```bash
kubectl get pods -A
kubectl describe pod <POD_NAME> -n <NAMESPACE>
kubectl logs <POD_NAME> -n <NAMESPACE>
```

---

# 51. 실습 27 — Artifact 확인

각 Task의 Artifact 확인.

```text
preprocess → Dataset
train      → Model
evaluate   → Metrics
```

전체 관계:

```text
preprocess
   └─ Dataset
       ↓
      train
       └─ Model
           ↓
        evaluate
           └─ Metrics
```

---

# 52. Artifact란?

Artifact는 Pipeline 실행 중 생성되는 데이터 또는 파일 형태 산출물.

예:

- Dataset
- CSV
- Model
- Evaluation Report
- Metrics
- TensorBoard Log

Parameter와 구분해야 함.

---

# 53. Pipeline Root와 Artifact Storage

개념:

```text
Pipeline
 ├─ Task A → Dataset
 ├─ Task B → Model
 └─ Task C → Metrics
               ↓
        Artifact Storage
```

실제 저장소는 배포 환경에 따라 다음과 같이 구성 가능.

- MinIO
- S3-compatible Storage
- Cloud Object Storage

---

# 54. ML Metadata

ML Metadata는 실행과 Artifact 관계를 기록함.

```text
Dataset A
   ↓
preprocess execution
   ↓
Dataset B
   ↓
train execution
   ↓
Model A
   ↓
evaluate execution
   ↓
Metrics
```

---

# 55. ML Lineage

Lineage:

```text
데이터와 모델이 어떤 과정을 거쳐 생성되었는지에 대한 계보
```

예:

```text
Model v3
   ↓
Train Run #25
   ↓
Dataset Version #7
   ↓
Preprocess Execution
```

재현성과 추적성을 높임.

---

# 56. 실습 28 — Artifacts / Executions 메뉴 확인

Dashboard에서:

```text
Artifacts
```

확인:

- Dataset
- Model
- Metrics

또한:

```text
Executions
```

확인:

- 각 Task의 실행 이력
- Artifact 생성 관계

---

# 57. 실습 29 — 두 번째 Run

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

# 58. 실습 30 — 세 번째 Run

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

# 59. Experiment에서 비교

```text
iris-experiment

├─ iris-run-c01  C=0.1
├─ iris-run-c1   C=1.0
└─ iris-run-c10  C=10.0
```

비교 항목:

- Parameter
- 실행 시간
- Task 상태
- Accuracy
- Metrics Artifact
- Log

---

# 60. 5주차와 6주차 연결

5주차:

```text
사람이 C 값을 지정
  ├─ 0.1
  ├─ 1.0
  └─ 10.0
```

6주차 Katib:

```text
Search Space
    ↓
Katib
    ├─ Trial 1
    ├─ Trial 2
    ├─ Trial 3
    └─ ...
```

Hyperparameter 탐색 자동화.

---

# 61. KServe 연결

현재:

```text
preprocess
   ↓
 train
   ↓
evaluate
```

6주차 이후:

```text
preprocess
   ↓
 train
   ↓
evaluate
   ↓
model deploy
   ↓
KServe
```

---

# 62. Pipeline의 4대 이점

## 재현성

```text
동일 Pipeline 정의
      ↓
동일한 DAG
      ↓
동일한 실행 단계
```

## 자동화

```text
Run 시작
  ↓
preprocess
  ↓
train
  ↓
evaluate
```

## 재사용성

한 Component를 여러 Pipeline에서 재사용 가능.

## 추적성

```text
Run
 ├─ Input
 ├─ Parameter
 ├─ Artifact
 ├─ Log
 └─ Metrics
```

---

# 63. Component / Pipeline / Run / Experiment 정리

```text
Component
   ↓
Pipeline
   ↓
Run
   ↓
Experiment
```

더 정확히는:

```text
여러 Component → 하나의 Pipeline
하나의 Pipeline → 여러 Run
여러 Run → 하나의 Experiment에서 그룹화 가능
```

---

# 64. KFP 전체 구조

```text
사용자
  ↓
Python SDK
  ↓
@dsl.component / @dsl.pipeline
  ↓
KFP Compiler
  ↓
IR YAML
  ↓
KFP Backend
  ↓
Kubernetes
  ↓
Container Task

실행 결과
  ├─ Run
  ├─ Artifact
  ├─ Metrics
  └─ Metadata / Lineage
```

---

# 65. 3주차·4주차·5주차 연결

3주차:

```text
Application
   ↓
Container Image
   ↓
Container
```

4주차:

```text
Container
   ↓
Pod
   ↓
Deployment / Service
```

5주차:

```text
Container 기반 ML Task
   ↓
Component
   ↓
Pipeline
   ↓
DAG
```

---

# 66. 실습 31 — KFP 관련 Kubernetes Resource 확인

```bash
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow | grep pipeline
kubectl get svc -n kubeflow | grep pipeline
kubectl get deploy -n kubeflow | grep pipeline
```

### 학습 포인트

Kubeflow Pipelines 자체도 Kubernetes Deployment, Pod, Service 등의 조합으로 실행됨.

---

# 67. 실습 32 — Pipeline 실행 중 Pod 관찰

Run 시작 직후:

```bash
kubectl get pods -A -w
```

관찰:

```text
Task 시작
   ↓
Pod 생성
   ↓
Container 실행
   ↓
작업 수행
   ↓
Artifact 생성
   ↓
Task 완료
```

---

# 68. 실습 33 — 장애 실습

전처리 코드를 일부러 잘못 변경.

예:

```python
df = pd.read_csv("/not-exist.csv")
```

컴파일 후 실행.

예상:

```text
preprocess → Failed
train      → 실행되지 않음
evaluate   → 실행되지 않음
```

이유:

```text
preprocess
   X
 train
   ↓
evaluate
```

DAG 의존관계가 있기 때문.

---

# 69. 장애 로그 확인

실패 Task의 Logs에서:

```text
FileNotFoundError
```

확인.

### 학습 포인트

Pipeline은 실패 위치와 로그를 단계별로 추적할 수 있음.

---

# 70. 실습 34 — 정상 코드로 복구

다시:

```python
iris = load_iris(as_frame=True)
```

컴파일:

```python
compiler.Compiler().compile(
    pipeline_func=simple_pipeline,
    package_path="simple_pipeline.yaml",
)
```

---

# 71. 전체 실습 코드

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

    model = LogisticRegression(C=c_value, max_iter=300)
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

# 72. 자주 사용하는 명령어

| 목적 | 명령 |
|---|---|
| Namespace | `kubectl get ns` |
| 전체 Pod | `kubectl get pods -A` |
| Kubeflow Pod | `kubectl get pods -n kubeflow` |
| Service | `kubectl get svc -n kubeflow` |
| Pipeline Pod 검색 | `kubectl get pods -n kubeflow | Select-String pipeline` |
| Pod 실시간 확인 | `kubectl get pods -A -w` |
| Pod 상세 | `kubectl describe pod POD -n NAMESPACE` |
| Pod Log | `kubectl logs POD -n NAMESPACE` |
| KFP SDK 버전 | `python -c "import kfp; print(kfp.__version__)"` |

---

# 73. 오류 1 — Pod Pending

```bash
kubectl get pods -A
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

가능 원인:

- CPU 부족
- Memory 부족
- PVC Pending
- StorageClass 없음
- Scheduling 조건 불충족

---

# 74. 오류 2 — ImagePullBackOff

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

가능 원인:

- Image 이름 오류
- Registry 접근 실패
- Docker Hub Rate Limit
- Private Registry 인증 실패
- CPU Architecture 불일치

---

# 75. 오류 3 — CrashLoopBackOff

```bash
kubectl logs <POD_NAME> -n <NAMESPACE>
kubectl logs <POD_NAME> -n <NAMESPACE> --previous
```

가능 원인:

- Application 오류
- 설정 오류
- Secret/Config 누락
- Database 연결 실패

---

# 76. 오류 4 — ModuleNotFoundError

예:

```text
ModuleNotFoundError: No module named 'pandas'
```

해결:

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas"]
)
```

또는 필요한 Package를 포함한 Custom Image 사용.

---

# 77. 오류 5 — Artifact 파일 오류

Output:

```python
output_data: Output[Dataset]
df.to_csv(output_data.path, index=False)
```

Input:

```python
input_data: Input[Dataset]
pd.read_csv(input_data.path)
```

Artifact의 실제 저장 URI를 코드에 직접 하드코딩하지 않음.

---

# 78. 초급 실습에서 UI 업로드 방식을 사용하는 이유

Notebook에서 KFP API를 직접 호출할 경우:

- ServiceAccount Token
- Kubeflow 인증
- Namespace
- API Endpoint

구성이 필요할 수 있음.

따라서 초급 실습에서는:

```text
Notebook
  ↓
YAML Compile
  ↓
YAML Download
  ↓
Kubeflow UI Upload
  ↓
Create Run
```

방식을 사용함.

---

# 79. 실습 종료

본 실습에서는 **기존에 준비된 Kind Cluster를 다음 주차에도 재사용**하므로 Cluster를 삭제하지 않음.

Dashboard Port Forward를 실행한 Terminal에서:

```text
Ctrl + C
```

로 Port Forward만 종료.

Cluster 상태 확인:

```powershell
kind get clusters
kubectl get nodes
```

> 주의  
> `kind delete cluster`는 본 실습 종료 절차에 포함하지 않음.  
> 6주차 Katib·KServe 실습에서도 동일 Kubeflow Cluster를 계속 사용함.

---

# 80. 최종 확인 문제

## 문제 1

Component와 Pipeline의 차이는?

### 정답

Component는 독립 작업 단위이고 Pipeline은 여러 Component를 연결한 Workflow임.

---

## 문제 2

Pipeline 실행 흐름을 표현하는 그래프 구조는?

### 정답

DAG

---

## 문제 3

다음 중 Parameter에 적합한 것은?

```text
A. 3 GB Dataset
B. model.pkl
C. learning_rate=0.01
D. image dataset
```

### 정답

C

---

## 문제 4

Dataset과 Model을 Component 사이에서 전달할 때 사용하는 개념은?

### 정답

Artifact

---

## 문제 5

Pipeline 1회 실행은?

### 정답

Run

---

## 문제 6

여러 Run을 묶는 단위는?

### 정답

Experiment

---

## 문제 7

모델이 어떤 데이터와 실행으로 만들어졌는지 추적하는 개념은?

### 정답

ML Lineage

---

## 문제 8

KFP v2 Pipeline 컴파일 결과는?

### 정답

KFP v2 IR YAML

---

## 문제 9

KFP v2는 반드시 Argo Workflows YAML로 변환되는가?

### 정답

아님. KFP v2는 Argo Workflows에 종속되지 않는 generic IR YAML을 사용함.

---

# 81. 오늘 실습의 핵심

```text
① Kubeflow 설치
      ↓
② Notebook 생성
      ↓
③ Component 작성
   ├─ preprocess
   ├─ train
   └─ evaluate
      ↓
④ Artifact 연결
   ├─ Dataset
   ├─ Model
   └─ Metrics
      ↓
⑤ Pipeline 정의
      ↓
⑥ DAG 구성
      ↓
⑦ Compiler
      ↓
⑧ IR YAML
      ↓
⑨ Run 실행
      ↓
⑩ Log / Artifact / Metrics 확인
      ↓
⑪ Experiment에서 여러 Run 비교
```

---

# 82. 반드시 기억할 한 문장

> **ML Pipeline은 데이터 전처리, 학습, 평가 등의 작업을 DAG로 연결하고 자동 실행함으로써 재현성, 자동화, 재사용성, 추적성을 확보하는 기술임.**

---

# 83. 6주차 예고

```text
5주차

preprocess
   ↓
 train
   ↓
evaluate
```

```text
6주차

Katib
  ↓
Hyperparameter Tuning
  ↓
train
  ↓
evaluate
  ↓
KServe
  ↓
Model Serving
```

---

# 84. 강의 내용과 실습 대응표

| 강의 내용 | 실습 |
|---|---|
| Pipeline 필요성 | 전처리→학습→평가 자동 실행 |
| DAG | KFP Run DAG |
| 재현성 | 동일 Pipeline 반복 실행 |
| 자동화 | 데이터 의존관계 기반 실행 |
| 재사용성 | Component 함수 분리 |
| 추적성 | Run/Artifact/Metrics 확인 |
| Component | preprocess/train/evaluate |
| Pipeline | `simple_pipeline()` |
| Run | `iris-run-*` |
| Experiment | `iris-experiment` |
| Artifact | Dataset/Model/Metrics |
| ML Metadata | Artifacts/Executions |
| Notebook | Jupyter 개발환경 |
| Kubernetes 연결 | Task Pod 확인 |
| Container 연결 | `base_image` |
| 6주차 연결 | Katib/KServe |

---

# 85. 참고문헌 및 공식 문서

- Kubeflow Documentation  
  https://www.kubeflow.org/docs/

- Installing Kubeflow  
  https://www.kubeflow.org/docs/started/installing-kubeflow/

- Kubeflow Community Distribution  
  https://github.com/kubeflow/community-distribution

- Kubeflow Notebooks  
  https://www.kubeflow.org/docs/components/notebooks/

- Kubeflow Notebook Quickstart  
  https://www.kubeflow.org/docs/components/notebooks/quickstart-guide/

- Kubeflow Pipelines  
  https://www.kubeflow.org/docs/components/pipelines/

- KFP User Guides  
  https://www.kubeflow.org/docs/components/pipelines/user-guides/

- KFP Components  
  https://www.kubeflow.org/docs/components/pipelines/user-guides/components/

- KFP Core Functions  
  https://www.kubeflow.org/docs/components/pipelines/user-guides/core-functions/

- KFP Version Compatibility  
  https://www.kubeflow.org/docs/components/pipelines/reference/version-compatibility/

- KFP SDK  
  https://pypi.org/project/kfp/
