좋습니다. VirtualBox 환경을 기준으로, 쿠버네티스 실습을 위한 \*\*리눅스 VM (골든 이미지)\*\*을 만드는 **1단계와 2단계 초반**의 상세 설치 방법과 커맨드를 안내해 드립니다.

## 🛠️ 1단계: 하이퍼바이저 및 리눅스 VM 준비

### 1\. Oracle VirtualBox 설치 및 설정

  * **VirtualBox 설치:** Oracle 공식 웹사이트에서 **VirtualBox** 최신 버전을 다운로드하여 윈도우 PC에 설치합니다.
  * **리눅스 OS 다운로드:** **Ubuntu Server LTS** 버전의 ISO 파일을 공식 사이트에서 다운로드합니다. (예: Ubuntu 24.04 LTS)

### 2\. VM 생성 (Master Node용 골든 이미지)

VirtualBox를 실행하여 새로운 VM을 생성합니다.

| 설정 항목 | 권장 값 | 설명 |
| :--- | :--- | :--- |
| **이름** | `k8s-golden-template` | 나중에 복제할 때 기준이 될 이름입니다. |
| **OS 종류** | Linux / Ubuntu (64-bit) | 다운로드한 이미지에 맞춥니다. |
| **메모리(RAM)** | 2048 MB (2GB) 이상 | 최소 2GB, 안정적인 운영을 위해 4GB 권장 |
| **프로세서(CPU)** | 2개 이상 | 쿠버네티스 노드 요구사항 |
| **하드 디스크** | 30 GB 이상 | 기본 운영체제 및 컨테이너 이미지 공간 확보 |
| **네트워크 어댑터** | **브리지 어댑터(Bridged Adapter)** | 클러스터 VM들이 호스트 PC와 **동일한 네트워크**를 사용하도록 하여, 외부에서 SSH 접속이 용이해집니다. |

### 3\. Ubuntu Server 설치 (CLI 환경)

VM을 시작하고 다운로드한 ISO 파일을 연결하여 설치를 진행합니다.

  * 설치 과정 중 **SSH 서버 설치** 옵션을 **반드시 선택**합니다.
  * GUI(데스크톱 환경) 없이 \*\*서버 버전(CLI 전용)\*\*으로 설치를 완료합니다.

-----

## 💾 2단계: 골든 이미지 필수 설정 커맨드

리눅스 설치를 완료하고 VM을 재부팅한 후, SSH를 통해 접속하거나 VM 콘솔에서 다음 명령어를 실행하여 필수 설정을 합니다.

### 1\. 패키지 업데이트 및 방화벽 설정

```bash
# 패키지 목록 갱신 및 모든 패키지 업데이트
sudo apt update
sudo apt upgrade -y

# 방화벽(UFW) 비활성화 또는 초기화 (쿠버네티스 설치 시 충돌 방지)
sudo ufw disable 
# 참고: 프로덕션 환경에서는 필요한 포트만 개방해야 하지만, 실습 편의를 위해 비활성화합니다.
```

### 2\. 리눅스 서버 필수 설정

#### A. 스왑(Swap) 영역 비활성화 (필수)

쿠버네티스(`kubelet`)는 성능 문제로 **스왑(Swap)** 메모리 사용을 허용하지 않습니다.

```bash
# 현재 스왑 상태 확인
sudo swapon --show

# 스왑 영역 즉시 비활성화
sudo swapoff -a

# 영구적으로 비활성화: /etc/fstab 파일 편집
# vi 또는 nano 에디터 사용
sudo vi /etc/fstab
```

> **파일 편집 내용:** `/etc/fstab` 파일에서 스왑 파티션에 해당하는 라인(대개 `swap` 키워드가 포함된 라인)을 찾아서 맨 앞에 **`#`** 기호를 추가하여 **주석 처리**합니다.

#### B. 브리지 모듈 로드 (필수)

컨테이너 네트워킹(특히 CNI)을 위해 필요한 **브리지(bridge)** 관련 커널 모듈을 로드합니다.

```bash
# 필요한 커널 모듈 로드
sudo modprobe br_netfilter

# 커널 파라미터 활성화 (네트워크 브리지 관련 설정)
# sysctl 설정을 영구적으로 적용하는 파일 편집
sudo vi /etc/sysctl.d/k8s.conf
```

> **파일 편집 내용:** `/etc/sysctl.d/k8s.conf` 파일을 새로 만들고 다음 두 줄을 추가합니다.

```bash
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

```bash
# 설정 즉시 적용
sudo sysctl --system
```

### 3\. 호스트 이름 설정 (Master Node로 임시 설정)

복제 후 이름을 변경해야 하지만, 템플릿의 호스트 이름을 임시로 설정하고, 복제 과정에서 IP 충돌을 피하기 위해 MAC 주소를 리셋합니다.

```bash
# 호스트 이름 설정 (예: k8s-master)
sudo hostnamectl set-hostname k8s-master
```

### 4\. VM 종료 및 스냅샷 저장

모든 설정이 완료되면 VM을 종료하고 VirtualBox 관리자에서 **스냅샷**을 저장합니다.

  * VM 이름 `k8s-golden-template`을 마우스 오른쪽 클릭 $\rightarrow$ **스냅샷** $\rightarrow$ **스냅샷 찍기**
  * 스냅샷 이름: `Base_Ready_for_Kube_Install`

이제 이 스냅샷을 기반으로 **Master Node**를 시작하고, 나머지 2대의 **Worker Node**를 \*\*복제(Clone)\*\*하여 사용하면 됩니다.
