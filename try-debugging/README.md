# Pod 디버깅 실습 가이드

## 개요

이 가이드는 Kubernetes Pod에서 발생하는 문제를 진단하고 해결하는 실습을 다룹니다. 이미지 버전 오류로 인한 `ErrImagePull` 문제를 `kubectl describe`와 `kubectl edit` 명령어를 사용하여 해결하는 과정을 설명합니다.

## 사전 준비

### 1. 이전 실습 완료

이 실습을 진행하기 전에 다음 실습을 완료해야 합니다:

- [make-cluster 가이드](../make-cluster/README.md) - Kind 클러스터 생성
- [make-the-pod 가이드](../make-the-pod/README.md) - Pod 생성 실습

### 2. 환경 설정

```bash
# 프로젝트 루트에서 환경 설정 적용
source .bashrc

# kubectl alias 설정 (선택사항)
alias k=kubectl
```

## 실습 시나리오

### 문제 상황

`pod-destruction.yaml` 파일을 적용했을 때, Pod가 `ErrImagePull` 상태로 실패합니다. 이는 존재하지 않는 이미지 버전(`blux2/hello-server:1.1`)을 참조하기 때문입니다.

### 실습 파일

#### pod-destruction.yaml (문제가 있는 파일)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: hello-server
      image: blux2/hello-server:1.1  # 존재하지 않는 버전
      ports:
        - containerPort: 8080
```

#### myapp.yaml (정상 파일)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: hello-server
      image: blux2/hello-server:1.0  # 정상 버전
      ports:
        - containerPort: 8080
```

## 실습 단계

### 1. 문제가 있는 Pod 생성

```bash
# try-debugging 디렉토리로 이동
cd try-debugging

# 문제가 있는 Pod 생성
kubectl apply -f pod-destruction.yaml
```

**출력 예시:**

```
pod/myapp configured
```

### 2. Pod 상태 확인

```bash
# Pod 상태 확인
kubectl get pod myapp
```

**출력 예시:**

```
NAME    READY   STATUS         RESTARTS   AGE
myapp   0/1     ErrImagePull   0          19m
```

**상태 분석:**

- `READY`: `0/1` - 컨테이너가 준비되지 않음
- `STATUS`: `ErrImagePull` - 이미지를 가져오는 중 오류 발생
- `RESTARTS`: `0` - 아직 재시작되지 않음

### 3. Pod 상세 정보 확인

```bash
# Pod 상세 정보 확인
kubectl describe pod myapp
```

**주요 확인 사항:**

1. **Container 상태:**

```
Containers:
  hello-server:
    Image:          blux2/hello-server:1.1
    State:          Waiting
      Reason:       ImagePullBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    2
```

2. **Events 섹션:**

```
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  19m               default-scheduler  Successfully assigned default/myapp to kind-control-plane 
  Normal   Pulling    19m               kubelet            Pulling image "blux2/hello-server:1.0"
  Normal   Pulled     19m               kubelet            Successfully pulled image "blux2/hello-server:1.0" in 5.466s
  Normal   Created    19m               kubelet            Created container hello-server
  Normal   Started    19m               kubelet            Started container hello-server
  Normal   Killing    20s               kubelet            Container hello-server definition changed, will be restarted
  Normal   BackOff    17s               kubelet            Back-off pulling image "blux2/hello-server:1.1"
  Warning  Failed     17s               kubelet            Error: ImagePullBackOff
  Warning  Failed     0s (x2 over 17s)  kubelet            Failed to pull image "blux2/hello-server:1.1": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/blux2/hello-server:1.1": failed to resolve reference "docker.io/blux2/hello-server:1.1": docker.io/blux2/hello-server:1.1: not found
  Warning  Failed     0s (x2 over 17s)  kubelet            Error: ErrImagePull
```

**문제 분석:**

- `ImagePullBackOff`: 이미지 다운로드 실패 후 재시도 중
- `ErrImagePull`: 이미지를 찾을 수 없음 (`not found`)
- 이미지 버전 `1.1`이 존재하지 않음

### 4. 문제 원인 확인

```bash
# Reason 필드만 확인
kubectl describe pod myapp | grep Reason
```

**출력 예시:**

```
      Reason:       ImagePullBackOff
      Reason:       Error
  Type     Reason     Age                From               Message
```

### 5. Pod 수정 (kubectl edit 사용)

```bash
# Pod 편집 모드로 진입
kubectl edit pod myapp
```

**편집 내용:**

1. 편집기가 열리면 이미지 버전을 수정:
   - `image: blux2/hello-server:1.1` → `image: blux2/hello-server:1.0`

2. 파일 저장 후 종료 (vi의 경우: `:wq`, nano의 경우: `Ctrl+X` → `Y` → `Enter`)

**출력 예시:**

```
pod/myapp edited
```

**참고:** `kubectl edit`는 기본적으로 시스템 기본 편집기를 사용합니다. Windows Git Bash에서는 보통 vi 또는 nano가 사용됩니다.

### 6. 수정 후 Pod 상태 확인

```bash
# Pod 상태 확인
kubectl get pod myapp
```

**출력 예시:**

```
NAME    READY   STATUS    RESTARTS        AGE
myapp   1/1     Running   1 (3m53s ago)   22m
```

**상태 분석:**

- `READY`: `1/1` - 컨테이너가 정상적으로 준비됨
- `STATUS`: `Running` - Pod가 정상 실행 중
- `RESTARTS`: `1 (3m53s ago)` - 이미지 변경으로 인해 한 번 재시작됨

### 7. 정리 (Pod 삭제)

```bash
# Pod 삭제
kubectl delete -f pod-destruction.yaml
```

**출력 예시:**

```
pod "myapp" deleted
```

**확인:**

```bash
kubectl get pod
```

**출력 예시:**

```
No resources found in default namespace.
```

## 주요 명령어

### Pod 상태 확인

```bash
# Pod 목록 확인
kubectl get pods
kubectl get pod

# 특정 Pod 상세 정보
kubectl describe pod <pod-name>

# Pod 상태만 간단히 확인
kubectl get pod <pod-name>

# Reason 필드만 확인
kubectl describe pod <pod-name> | grep Reason
```

### Pod 수정

```bash
# Pod 편집 (인터랙티브 모드)
kubectl edit pod <pod-name>

# Pod YAML 출력 후 수정
kubectl get pod <pod-name> -o yaml > pod-fixed.yaml
# 파일 수정 후
kubectl apply -f pod-fixed.yaml
```

### Pod 이벤트 확인

```bash
# Pod 이벤트 확인
kubectl describe pod <pod-name>

# 모든 이벤트 확인 (시간순 정렬)
kubectl get events --sort-by='.lastTimestamp'

# 특정 Pod의 이벤트만 확인
kubectl get events --field-selector involvedObject.name=<pod-name>
```

### Pod 로그 확인

```bash
# Pod 로그 확인
kubectl logs <pod-name>

# 실시간 로그 확인
kubectl logs -f <pod-name>

# 특정 컨테이너 로그 확인 (멀티 컨테이너인 경우)
kubectl logs <pod-name> -c <container-name>
```

## 문제 해결 가이드

### 1. ErrImagePull / ImagePullBackOff

**증상:**

```
STATUS: ErrImagePull 또는 ImagePullBackOff
```

**원인:**

- 이미지가 존재하지 않음
- 이미지 이름 또는 태그 오류
- 이미지 레지스트리에 접근할 수 없음
- 인증 문제

**해결 방법:**

1. **이미지 이름 확인:**

```bash
kubectl describe pod <pod-name> | grep Image
```

2. **이미지 버전 수정:**

```bash
kubectl edit pod <pod-name>
# 또는
kubectl set image pod/<pod-name> <container-name>=<correct-image>
```

3. **이미지 존재 여부 확인:**

```bash
# Docker Hub에서 확인
docker pull <image-name>:<tag>

# 또는 kind 클러스터에 이미지 로드
kind load docker-image <image-name>:<tag>
```

### 2. Pod가 Pending 상태

**증상:**

```
STATUS: Pending
```

**원인:**

- 노드 리소스 부족
- 노드 선택자 불일치
- 볼륨 마운트 실패

**해결 방법:**

```bash
# Pod 상세 정보 확인
kubectl describe pod <pod-name>

# Events 섹션에서 원인 확인
kubectl describe pod <pod-name> | grep -A 10 Events
```

### 3. Pod가 CrashLoopBackOff 상태

**증상:**

```
STATUS: CrashLoopBackOff
```

**원인:**

- 컨테이너가 시작 후 즉시 종료됨
- 애플리케이션 오류
- 설정 오류

**해결 방법:**

```bash
# 로그 확인
kubectl logs <pod-name>

# 이전 컨테이너 로그 확인
kubectl logs <pod-name> --previous

# 상세 정보 확인
kubectl describe pod <pod-name>
```

### 4. kubectl edit 사용 시 주의사항

**중요:**

- `kubectl edit`로 Pod를 수정하면 Pod가 재시작됩니다
- 일부 필드는 수정할 수 없습니다 (예: `metadata.name`)
- Pod는 불변(immutable) 객체이므로, 실제로는 Pod를 삭제하고 새로 생성합니다

**권장 방법:**

- 개발/테스트 환경: `kubectl edit` 사용 가능
- 프로덕션 환경: YAML 파일 수정 후 `kubectl apply` 사용 권장

## kubectl edit 상세 설명

### 편집기 설정

**기본 편집기 확인:**

```bash
echo $EDITOR
```

**편집기 변경:**

```bash
# vi 사용
export EDITOR=vi

# nano 사용
export EDITOR=nano

# VS Code 사용 (Windows)
export EDITOR="code --wait"
```

### 편집 모드 종료

**vi 편집기:**

- 저장 후 종료: `:wq` 또는 `:x`
- 저장하지 않고 종료: `:q!`

**nano 편집기:**

- 저장 후 종료: `Ctrl+X` → `Y` → `Enter`
- 저장하지 않고 종료: `Ctrl+X` → `N`

### 편집 실패 시

편집을 취소하거나 저장하지 않고 종료하면:

```
error: pods "myapp" was not modified
```

Pod는 변경되지 않습니다.

## 실습 단계 요약

1. ✅ `pod-destruction.yaml` 적용 (이미지 버전 1.1 - 존재하지 않음)
2. ✅ `kubectl get pod myapp`으로 상태 확인 (`ErrImagePull`)
3. ✅ `kubectl describe pod myapp`으로 상세 정보 확인
4. ✅ Events 섹션에서 이미지 다운로드 실패 원인 확인
5. ✅ `kubectl edit pod myapp`으로 이미지 버전을 1.0으로 수정
6. ✅ `kubectl get pod myapp`으로 수정 후 상태 확인 (`Running`)
7. ✅ `kubectl delete -f pod-destruction.yaml`로 Pod 삭제

## 학습 포인트

### 1. Pod 상태 이해

- **Pending**: Pod가 스케줄링 대기 중
- **Running**: Pod가 실행 중
- **Succeeded**: Pod의 모든 컨테이너가 성공적으로 종료
- **Failed**: Pod의 컨테이너 중 하나 이상이 실패
- **Unknown**: Pod 상태를 확인할 수 없음

### 2. 이미지 관련 오류

- **ErrImagePull**: 이미지를 가져오는 중 오류 발생
- **ImagePullBackOff**: 이미지 다운로드 실패 후 재시도 중 (Back-off 상태)

### 3. kubectl describe의 중요성

`kubectl describe`는 Pod 문제를 진단하는 가장 중요한 명령어입니다:

- **Containers 섹션**: 컨테이너 상태 및 이미지 정보
- **Events 섹션**: Pod의 이벤트 히스토리 (시간순)
- **Conditions 섹션**: Pod의 조건 상태

### 4. kubectl edit의 활용

- 빠른 수정이 필요한 경우 유용
- YAML 파일이 없는 경우에도 수정 가능
- 주의: Pod는 불변 객체이므로 실제로는 재생성됨

## 다음 단계

- [Deployment 실습](../README.md) - Pod를 관리하는 더 고수준 리소스
- [Service 실습](../README.md) - Pod에 네트워크 접근 제공
- [ConfigMap과 Secret](../README.md) - 설정 및 민감 정보 관리

## 참고 자료

- [Kubernetes Pod 문서](https://kubernetes.io/docs/concepts/workloads/pods/)
- [kubectl describe 문서](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [kubectl edit 문서](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [Troubleshooting Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [kubectl 치트시트](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

