## Kubernetes Storage: PV와 PVC의 개념 및 동작 원리

쿠버네티스 환경에서 컨테이너 내부의 데이터는 컨테이너가 종료되면 사라지는 **휘발성**을 가집니다. 이를 해결하기 위해 영구 저장소인 **PV(Persistent Volume)**와 **PVC(Persistent Volume Claim)** 시스템을 사용하여 인프라와 애플리케이션의 결합도를 낮추고 데이터를 보존합니다.

---

### 1. PV (Persistent Volume)

PV는 클러스터 관리자가 미리 생성하거나 동적으로 할당된 **실제 물리 저장소 자원**입니다.

* **인프라 관점:** 실제 데이터가 저장되는 디스크(AWS EBS, NFS, 로컬 디스크 등)를 의미합니다.
* **독립적 생명주기:** PV는 이를 사용하는 파드(Pod)의 생명주기와 관계없이 독립적으로 존재합니다.
* **설정 요소:** 용량, 접근 모드(ReadWriteOnce 등), 회수 정책(Retain, Delete 등)을 포함합니다.

---

### 2. PVC (Persistent Volume Claim)

PVC는 사용자가 스토리지 자원을 사용하기 위해 제출하는 **요청서**입니다.

* **사용자 관점:** "최소 10GB 공간이 필요하고 읽기/쓰기가 가능해야 한다"는 요구사항을 정의합니다.
* **추상화:** 개발자는 실제 스토리지가 AWS EBS인지 온프레미스 NFS인지 알 필요 없이 PVC만 작성하면 됩니다.
* **바인딩:** 쿠버네티스 마스터가 PVC의 요구사항과 일치하는 PV를 찾아 두 객체를 서로 연결(Binding)합니다.

---

### 3. 주요 생명주기 및 흐름

1. **프로비저닝 (Provisioning):** * **정적(Static):** 관리자가 수동으로 PV를 생성합니다.
* **동적(Dynamic):** `StorageClass`를 통해 PVC가 요청될 때 자동으로 PV가 생성됩니다. (대규모 환경에서 권장)


2. **바인딩 (Binding):** PVC의 조건에 맞는 PV를 시스템이 찾아 매핑합니다. 1:1 관계입니다.
3. **사용 (Using):** 파드 정의서에 PVC 이름을 명시하여 볼륨으로 마운트합니다.
4. **회수 (Reclaiming):** PVC를 삭제했을 때 남은 PV를 어떻게 처리할지 결정합니다.
* **Retain:** 데이터와 PV를 그대로 보존 (수동 삭제 필요).
* **Delete:** 실제 물리 스토리지까지 삭제 (AWS EBS 등 클라우드 기본값).



---

### 4. 접근 모드 (Access Modes)

| 모드 | 설명 |
| --- | --- |
| **ReadWriteOnce (RWO)** | 단일 노드에서만 읽기/쓰기로 마운트 가능 |
| **ReadOnlyMany (ROX)** | 여러 노드에서 읽기 전용으로 마운트 가능 |
| **ReadWriteMany (RWX)** | 여러 노드에서 읽기/쓰기로 마운트 가능 (NFS 등 지원 필요) |

---

### 5. 비용 발생 주의 (AWS 연동 시)

AWS 환경에서 PV를 EBS(Elastic Block Store)로 생성할 경우, 파드 사용 여부와 관계없이 **EBS 볼륨 크기만큼 비용이 발생**합니다.

* **과금 요인:** 할당된 GiB당 월정액 비용이 청구됩니다.
* **주의사항:** PVC를 삭제하더라도 PV의 Reclaim Policy가 `Retain`으로 되어 있으면 실제 EBS 볼륨이 삭제되지 않아 계속 과금될 수 있습니다. 반드시 AWS 콘솔에서 EBS 볼륨 상태를 확인해야 합니다.

---

**Next Step: StorageClass를 활용한 동적 프로비저닝 실습**

* 실제 YAML 매니페스트를 작성하여 AWS EBS와 연동하는 과정을 진행해 볼까요?

---## PV 및 PVC YAML 설정과 상세 분석

쿠버네티스에서 스토리지를 정의하고 사용하기 위한 가장 기본적인 YAML 파일 구조입니다. AWS의 **EBS(Elastic Block Store)**를 백엔드 스토리지로 사용하는 가상의 시나리오를 바탕으로 각 라인별 의미를 상세히 분석합니다.

---

### 1. PersistentVolume (PV) 정의서

PV는 클러스터의 실제 저장 자원을 정의합니다.

```yaml
apiVersion: v1 # 쿠버네티스 API 버전 정의
kind: PersistentVolume # 리소스의 종류가 PV임을 명시
metadata: # 리소스의 메타데이터 시작
  name: ebs-pv-storage # PV의 이름을 설정 (클러스터 내 유일해야 함)
spec: # PV의 상세 사양 설정 시작
  capacity: # 저장 용량 설정
    storage: 10Gi # 10기가바이트(GiB) 용량 할당
  volumeMode: Filesystem # 볼륨 사용 방식을 파일시스템으로 설정 (기본값)
  accessModes: # 스토리지 접근 모드 설정
    - ReadWriteOnce # 하나의 노드에서만 읽기/쓰기가 가능하도록 설정
  persistentVolumeReclaimPolicy: Retain # PVC 삭제 후 PV 보존 정책 (Retain: 데이터 보존)
  storageClassName: manual # 매칭될 스토리지 클래스 이름 지정 (PVC와 일치해야 함)
  awsElasticBlockStore: # AWS EBS를 백엔드 스토리지로 사용함으로 명시
    volumeID: "<VOLUME_ID>" # 실제 AWS에서 생성된 EBS 볼륨 ID 입력
    fsType: ext4 # 파일시스템 형식을 ext4로 지정

```

---

### 2. PersistentVolumeClaim (PVC) 정의서

PVC는 파드가 사용할 스토리지를 요청하는 요청서입니다.

```yaml
apiVersion: v1 # 쿠버네티스 API 버전 정의
kind: PersistentVolumeClaim # 리소스의 종류가 PVC임을 명시
metadata: # 리소스의 메타데이터 시작
  name: ebs-pvc-claim # PVC의 이름을 설정 (파드에서 참조할 이름)
spec: # 스토리지 요청 사양 설정 시작
  accessModes: # 필요한 접근 모드 정의
    - ReadWriteOnce # PV의 접근 모드와 일치해야 바인딩 가능
  resources: # 필요한 자원 정의
    requests: # 최소 요구 자원 설정
      storage: 5Gi # 5기가바이트(GiB) 이상의 용량을 가진 PV를 요청
  storageClassName: manual # 요청할 스토리지 클래스 이름 (PV의 이름과 일치해야 함)

```

---

### 3. 주요 구성 요소 상세 설명

* **accessModes (접근 모드):**
* `ReadWriteOnce` (RWO): 단일 노드가 읽기/쓰기 가능.
* `ReadOnlyMany` (ROX): 여러 노드가 읽기 전용 가능.
* `ReadWriteMany` (RWX): 여러 노드가 읽기/쓰기 가능.


* **persistentVolumeReclaimPolicy (회수 정책):**
* `Retain`: PVC가 삭제되어도 PV와 실제 데이터는 삭제되지 않고 `Released` 상태가 되어 수동 정리가 필요합니다.
* `Delete`: PVC 삭제 시 연결된 PV와 실제 물리 스토리지(EBS 등)가 함께 자동 삭제됩니다.


* **storageClassName:**
* PV와 PVC를 논리적으로 묶어주는 역할을 합니다. 동적 프로비저닝을 사용할 때는 `StorageClass` 객체를 통해 자동으로 PV를 생성하게 됩니다.



---

### 4. AWS 과금 주의사항

* **EBS 비용:** `awsElasticBlockStore`를 통해 PV를 생성하면, 해당 볼륨이 실제 데이터 저장 여부와 상관없이 AWS 계정에 **EBS 사용 요금**이 매달 청구됩니다.
* **정리 필수:** 실습 종료 후에는 `kubectl delete pvc` 뿐만 아니라, 필요에 따라 `kubectl delete pv`를 수행하고, 최종적으로 **AWS 콘솔**에서 EBS 볼륨이 완전히 삭제되었는지 반드시 확인해야 과금 폭탄을 방지할 수 있습니다.

---

**Next Step: Pod에 PVC 연결 및 데이터 보존 확인**

* 위에서 만든 PVC를 실제 파드에 마운트하여 파일을 생성하고, 파드를 삭제 후 재생성해도 데이터가 남아있는지 확인하는 실습을 진행해 볼까요?

---
## PV와 PVC를 적용하여 스토리지를 구성하는 실전 프로세스

작성한 YAML 파일을 기반으로 실제 쿠버네티스 클러스터에 반영하고, 상태를 점검하며 파드에 연결하는 전체 과정을 단계별로 설명합니다.

---

### 1. PV 및 PVC 생성 및 상태 확인

먼저 정의한 스토리지를 클러스터에 등록하고, 두 객체가 정상적으로 연결(Bound)되는지 확인합니다.

* **PV 생성:**
```bash
kubectl apply -f pv.yaml

```


* **PVC 생성:**
```bash
kubectl apply -f pvc.yaml

```


* **상태 점검:**
```bash
kubectl get pv,pvc

```


> **핵심 체크:** `STATUS` 컬럼이 **`Bound`**로 표시되어야 합니다. 만약 `Pending` 상태라면 `storageClassName`이나 `accessModes`, `capacity` 요구사항이 서로 일치하는지 확인해야 합니다.



---

### 2. Pod에서 PVC 사용하기 (실행 예제)

생성된 PVC를 파드에 볼륨으로 마운트하여 실제 데이터를 저장할 준비를 합니다. 아래는 파드 설정 파일(`pod-storage.yaml`)의 예시입니다.

```yaml
apiVersion: v1 # 쿠버네티스 API 버전
kind: Pod # 리소스 종류
metadata: # 메타데이터
  name: storage-test-pod # 파드 이름
spec: # 사양 설정
  containers: # 컨테이너 설정
    - name: nginx-container # 컨테이너 이름
      image: nginx # 사용할 이미지
      volumeMounts: # 컨테이너 내부 마운트 경로 설정
        - mountPath: "/usr/share/nginx/html" # 컨테이너 내 저장 경로
          name: my-storage # 아래 volumes에서 정의한 이름과 매칭
  volumes: # 파드에서 사용할 볼륨 정의
    - name: my-storage # 볼륨 이름 설정
      persistentVolumeClaim: # PVC를 통해 볼륨을 할당받음
        claimName: ebs-pvc-claim # 위에서 생성한 PVC 이름과 반드시 일치해야 함

```

* **파드 실행:**
```bash
kubectl apply -f pod-storage.yaml

```



---

### 3. 데이터 영속성 검증 (실습)

파드가 삭제되어도 데이터가 유지되는지 직접 확인하는 과정입니다.

1. **파드 내부에 파일 생성:**
```bash
kubectl exec -it storage-test-pod -- /bin/bash
# (컨테이너 내부 접속 후)
echo "Hello, Storage!" > /usr/share/nginx/html/index.html
exit

```


2. **파드 강제 삭제 및 재시작:**
```bash
kubectl delete pod storage-test-pod
kubectl apply -f pod-storage.yaml

```


3. **데이터 복구 확인:**
```bash
kubectl exec -it storage-test-pod -- cat /usr/share/nginx/html/index.html
# 결과로 "Hello, Storage!"가 출력되면 성공

```



---

### 4. 자원 정리 및 과금 방지

테스트가 끝난 후 자원을 삭제하는 올바른 순서입니다.

1. **파드 및 PVC 삭제:**
```bash
kubectl delete pod storage-test-pod
kubectl delete pvc ebs-pvc-claim

```


2. **PV 삭제:**
```bash
kubectl delete pv ebs-pv-storage

```


3. **AWS 리소스 확인 (중요):**
* **AWS 관리 콘솔**의 EC2 > Volumes 메뉴로 이동합니다.
* 사용했던 EBS 볼륨의 상태가 `available`인지 확인하고, 필요 없다면 **볼륨 삭제(Delete Volume)**를 수동으로 수행해야 추가 과금을 완전히 막을 수 있습니다.



---

**Next Step: StorageClass 기반 동적 프로비저닝 구현**

* 수동으로 PV를 만들지 않고 PVC만 생성하면 AWS EBS가 자동 생성되도록 설정하는 방법을 알아볼까요?

---

