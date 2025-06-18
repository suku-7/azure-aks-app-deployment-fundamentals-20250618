# Model
## kubernetes-deployment-labels-lab-20250618
https://labs.msaez.io/#/189596125/storming/modelforops2

Kubernetes 기본 애플리케이션 배포 및 관리 (Labels, Annotations, Rollout)
- 이 실습은 쿠버네티스에서 애플리케이션을 배포하고 Labels, Annotations를 활용해 객체를 효율적으로 관리하는 방법을 다룹니다.
- Deployment의 롤아웃(Rollout) 및 롤백(Rollback) 기능을 통해 애플리케이션 버전을 안전하게 관리하며, ReplicaSet, Namespace, Service와 같은 핵심 개념을 함께 이해합니다.
- 클라우드 환경에서 안정적인 서비스 운영을 위한 쿠버네티스 기본기를 다지는 데 중점을 둡니다.



## 실습 단계별 상세 설명

0. 환경 초기화
```
# 환경 초기화 스크립트 실행 (init.sh 스크립트가 존재한다고 가정)
./init.sh
```
1. Java SDK 설치 (선택 사항)
```
# 필요한 경우 Java Development Kit(SDK) 설치 또는 업그레이드
sdk install java
```
2. Azure 로그인 및 Kubernetes 클러스터 연결
```
# Azure 계정에 로그인
az login --use-device-code
# (콘솔 안내에 따라 웹 브라우저에서 인증 진행 후 터미널에 '1' 입력 후 Enter)

# AKS 클러스터 자격 증명 설정
az aks get-credentials --resource-group a071098-rsrcgrp --name a071098-aks
```
3. 이전 프로젝트 리소스 삭제 (클러스터 초기화)
```
# 현재 실행 중인 모든 Kubernetes 객체 확인
kubectl get all

# 모든 Deployment 및 Service 삭제
kubectl delete deploy,svc --all

# 모든 리소스가 삭제되었는지 확인
kubectl get all

# (선택 사항: Kafka StatefulSet 및 관련 Pod/PVC 삭제)
# Kafka StatefulSet 삭제 (에러 무시)
kubectl delete statefulset my-kafka 2>/dev/null || true
# Kafka Client Pod 삭제 (에러 무시)
kubectl delete pod my-kafka-client 2>/dev/null || true

# Persistent Volume Claims (PVC) 목록 확인
kubectl get pvc
# Kafka에 사용된 PVC 삭제 (이름은 다를 수 있으니 'data-my-kafka-0'를 확인 후 삭제, 에러 무시)
kubectl delete pvc data-my-kafka-0 2>/dev/null || true
```
4. 홈페이지 Deployment 배포 (v1)
```
# 'home'이라는 이름의 Deployment를 YAML 정의를 통해 배포
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home
spec:
  replicas: 1
  selector:
    matchLabels:
      app: home
  template:
    metadata:
      labels:
        app: home
    spec:
      containers:
        - name: welcome # 이 컨테이너 이름이 중요합니다!
          image: apexacme/welcome:v1
          ports:
            - containerPort: 80
EOF
```
5. 배포된 객체 확인 및 레이블 조회
```
# 현재 클러스터에 배포된 모든 Kubernetes 객체 상태 확인
kubectl get all

# 모든 Pod들의 레이블 상세 조회
kubectl get pod --show-labels
```
6. 레이블을 통한 객체 조회 (동일성 비교)
```
# 'app' 레이블 값이 'home'인 Pod들 조회
kubectl get pods -l app=home

# '--selector' 옵션을 이용해 'app' 레이블 값이 'home'인 Pod들 조회
kubectl get pods --selector app=home

# 'app=home' 레이블을 가진 Pod 삭제 (Deployment가 즉시 재생성함)
kubectl delete pod -l app=home
```
7. 레이블을 통한 객체 조회 (집합 기반 선택)
```
# 'app' 레이블 값이 'home' 또는 'home1'인 Pod들 조회
kubectl get pods --selector 'app in (home, home1)'

# 'env' 레이블 값이 'home' 또는 'home1'이고, 'app' 레이블 값이 'home' 또는 'home1'인 Pod들 조회
kubectl get po --selector 'env in(home, home1), app in (home, home1)'
```
8. Deployment에 레이블 추가 및 삭제
```
# 현재 Deployment 목록 확인
kubectl get deploy

# 'home' Deployment에 'app=home' 레이블 추가 (이미 있다면 덮어쓰기)
kubectl label deploy home app=home --overwrite

# 'home' Deployment에 'env=home' 레이블 추가
kubectl label deploy home env=home

# 레이블이 추가된 Deployment 상세 조회
kubectl get deployment --show-labels

# 'app=home' 레이블을 가진 Deployment 삭제 (테스트용)
kubectl delete deploy --selector app=home

# 모든 Kubernetes 객체 상태 재확인
kubectl get all
```
9. 홈페이지 v1 재배포 및 배포 주석 추가
```
# 'home' Deployment를 v1 이미지로 다시 생성
kubectl create deploy home --image=apexacme/welcome:v1

# 배포된 'home' Deployment의 상세 정보 확인 (특히 CONTAINERS 컬럼 확인)
kubectl get deploy -o wide

# v1 배포에 대한 변경 주석 추가
kubectl annotate deploy home kubernetes.io/change-cause="v1 is The first deploy of My Homepage."
```
10. 홈페이지 v2로 업그레이드 및 주석 추가
```
# 'home' Deployment의 'welcome' 컨테이너 이미지를 v2로 업데이트
# 주의: 컨테이너 이름 'welcome'을 정확히 사용해야 합니다.
kubectl set image deploy home welcome=apexacme/welcome:v2

# 업데이트된 'home' Deployment의 상세 정보 확인
kubectl get deploy -o wide

# v2 배포에 대한 변경 주석 추가
kubectl annotate deploy home kubernetes.io/change-cause="v2 is The 2nd version of My Homepage."

# 배포 히스토리 확인
kubectl rollout history deploy home
```
11. 이전 버전으로 롤백
```
# 'home' Deployment를 가장 최근 이전 버전으로 롤백
kubectl rollout undo deploy home

# 롤백 후 'home' Deployment의 상세 정보 확인
kubectl get deploy -o wide

# (선택 사항: 특정 리비전으로 롤백)
# 예: 배포 이력에서 특정 버전(리비전 5)으로 롤백
# kubectl rollout undo deploy home --to-revision 5
```
