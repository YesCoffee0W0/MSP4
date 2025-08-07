
---
# 05. VM과 호스트 간 통신

## Pnet - Vagrant (Ubuntu) 설정
```bash
"""create another folder for ubuntu_vm2"""
> mkdir ubuntu_vm

"""create Vagrantfile and add the following:"""
Vagrant.configure("2") do |config|
  
  config.vm.box = "bento/ubuntu-22.04"
  
  config.vm.network 
  "private_network", ip: "192.168.12.10" # ip shall be sharing same domain with ubuntu_vm2: 12.0
  
  config.vm.provider "vmware_desktop" do |vmware|
    vmware.memory = 2048
    vmware.cpus = 2
  end

end

"""check ping to host PC"""
# configure firewall if ping is not working (see no.4: Ping이 안 될 때 방화벽 설정)
> ping 192.168.12.1 -c 4 # ping 4번 제한

```
## Pnet Network Adapter 설정
- pnet terminal 에서 ifconfig | more 로 Vmware Pnet network adapter 의 MAC 주소 매칭 확인

## Pnet Topology
Network (Cloud1) - Router - VPC
Cloud1: 192.168.12.10/24 주소를 사용하고 있는 Ubuntu 게스트 OS 와 연결
VPC: 172.16.1.56/24 (IP)
Router: 172.16.1.175/24 (Gateway eth 0/1)

XShell Router terminal configure
XShell VPC terminal configure
후에 Router 에서 ping 192.16.12.10 


## 1. VMware 가상 네트워크(vmnet2)란?

- VMware에서 만든 소프트웨어 네트워크로, 실제 네트워크와는 별도로 동작합니다.
- 예시: vmnet2를 192.168.12.0/24로 설정하면, 이 대역에서만 통신이 가능합니다.

## 2. 호스트(로컬 PC)와 VM의 IP 차이

- **호스트(로컬 PC) IP:**  
  - VMware가 vmnet2를 만들 때, 로컬 PC에 가상 네트워크 어댑터를 추가하고 IP(예: 192.168.12.1)를 할당합니다.
  - 이 IP는 vmnet2 네트워크에서만 사용됩니다.
- **VM IP:**  
  - Vagrantfile에서 지정한 IP(예: 192.168.12.10).
  - VM이 vmnet2 네트워크에서 사용하는 주소입니다.

## 3. 게이트웨이와 물리 네트워크의 차이

- **물리적 게이트웨이:**  
  - 집/회사 네트워크의 공유기 등(예: 192.168.0.1).
- **가상 네트워크 게이트웨이:**  
  - vmnet2에서 호스트 PC가 가지는 IP(예: 192.168.12.1).
  - VM에서 이 주소로 ping을 보내면, 가상 네트워크 연결 상태를 확인할 수 있습니다.

## 4. Ping이 안 될 때 방화벽 설정

- Windows 방화벽이 ICMP(핑) 요청을 차단할 수 있습니다.
- **해결 방법:**  
  1. Windows Defender 방화벽 고급 설정 → 인바운드 규칙 → 새 규칙
  2. "사용자 지정" → 프로토콜 ICMPv4 → 연결 허용
  3. 도메인/개인/공용 프로필 선택(모두 선택해도 무방)
  4. 규칙 이름 지정 후 완료

## 5. 도메인/개인/공용 프로필 설명

- **도메인:** 회사 등 도메인(Active Directory) 네트워크
- **개인:** 집 등 신뢰할 수 있는 네트워크
- **공용:** 카페, 공항 등 신뢰할 수 없는 네트워크
- VM 네트워크가 어떤 프로필로 인식되는지에 따라 선택

---

## NAT 비활성화 및 라우팅 경로 변경 정리

1. **Vagrant VM에서 NAT 비활성화 및 SSH 재접속**
   - VM에서 로그아웃  
     ```
     logout
     ```
   - SSH 접속 정보 확인  
     ```
     vagrant ssh-config
     ```
   - 출력된 IdentityFile 경로 복사 후, 새로운 터미널에서 SSH 접속  
     ```
     ssh -i D:/MSP4/workspace/vagrant/ubuntu_vm/.vagrant/machines/default/vmware_desktop/private_key vagrant@192.168.12.10
     ```

2. **known_hosts 초기화**
   - Windows 터미널에서 known_hosts 파일 열기  
     ```
     cursor $env:USERPROFILE/.ssh/known_hosts
     ```
   - 파일이 열리면 기존 호스트 정보(키) 삭제

3. **라우팅 테이블 확인 및 기본 경로 변경**
   - 현재 라우팅 테이블 확인  
     ```
     ip route
     ```
     예시 출력:
     ```
     default via 192.168.65.2 dev eth0 proto dhcp src 192.168.65.134 metric 100 
     192.168.12.0/24 dev eth1 proto kernel scope link src 192.168.12.10
     192.168.65.0/24 dev eth0 proto kernel scope link src 192.168.65.134 metric 100     
     192.168.65.2 dev eth0 proto dhcp scope link src 192.168.65.134 metric 100
     ```
   - 기존 기본 경로(NAT) 삭제  
     ```
     sudo ip route del default via 192.168.65.2 dev eth0
     ```
   - 새로운 기본 경로(Cloud1 네트워크 게이트웨이) 추가  
     ```
     sudo ip route add default via 192.168.12.253 dev eth1
     ```
   - 네트워크 연결 확인  
     ```
     ping 172.16.1.56
     ```
   - 참고: `dev`는 로컬 인터페이스 지정

4. **Pnet에서 Cloud0 추가 및 라우터 DHCP 설정**
   - Pnet에서 Cloud0 네트워크 추가
   - 라우터에서 DHCP로 IP 할당받도록 설정  
     ```
     ip add dhcp
     ```
   - 외부 네트워크 연결 확인  
     ```
     ip route get 8.8.8.8
     ping 8.8.8.8
     ```

5. **Cloud0 제거 및 Cloud_nat 추가**
   - 실습 목적에 따라 Cloud0(Management) 네트워크를 제거하고, Cloud_nat 네트워크 추가  
   - 관리 목적이라면 Cloud0 유지가 더 나은 방법일 수 있음

---

# Cisco ACL(Access Control List) 정리

---

## 1. ACL 종류

### Standard Access List (표준 ACL)
- **번호 범위:** 1~99, 1300~1999
- **기능:**  
  - 소스 IP 주소만 기준으로 트래픽을 허용/차단
  - 목적지 IP, 포트, 프로토콜 등은 제어 불가
- **용도:**  
  - 간단한 네트워크 접근 제어, 특정 호스트/네트워크 차단 등
- **예시:**
  ```bash
  access-list 10 permit 192.168.1.0 0.0.0.255
  ```
- **적용 위치:**  
  - 주로 목적지에 가까운 인터페이스(INBOUND)에 적용

---

### Extended Access List (확장 ACL)
- **번호 범위:** 100~199, 2000~2699
- **기능:**  
  - 소스/목적지 IP, 프로토콜(TCP/UDP/ICMP 등), 포트 번호 등 세부 조건까지 지정 가능
  - 트래픽의 종류(예: HTTP, SSH 등)까지 세밀하게 제어
- **용도:**  
  - 서비스별, 방향별, 세부 정책이 필요한 경우
- **예시:**
  ```bash
  access-list 100 permit tcp 192.168.1.0 0.0.0.255 any eq 80
  ```
  - 위 예시는 192.168.1.0/24에서 나가는 TCP 80번(HTTP) 트래픽만 허용

- **적용 위치:**  
  - 주로 트래픽의 소스에 가까운 인터페이스(OUTBOUND)에 적용

---

## 2. 번호 선택 기준 및 관리 팁

- **번호는 ACL의 종류와 용도에 따라 선택**
  - 1~99, 1300~1999: Standard
  - 100~199, 2000~2699: Extended
- **여러 정책을 그룹화할 때는 10, 20, 30... 또는 100, 110, 120... 등 간격을 두고 관리**
- **정책별로 번호를 나누면 유지보수에 유리**

---

## 3. 주요 키워드 설명

- **eq 80:**  
  - "equal 80"의 약자, 포트 80(HTTP)만 허용/차단
- **any:**  
  - 모든 IP 또는 모든 포트
- **host:**  
  - 단일 IP 지정 (예: `host 192.168.1.10`)

---

## 4. 적용 예시

```bash
# 표준 ACL 예시
access-list 10 permit 192.168.1.0 0.0.0.255

# 확장 ACL 예시 (HTTP만 허용)
access-list 100 permit tcp any any eq 80

# 인터페이스에 적용
interface ethernet 0/0
ip access-group 100 in
```

---
## Task 1: ACL 연습 정리

### 목적
- R4 라우터에서 PC2(172.16.1.56)로 나가는 트래픽을 아웃바운드(OUTBOUND)에서 필터링(차단)하는 연습

### 절차 및 명령어
1. **글로벌 설정 모드 진입**
   ```
   conf t
   ```
2. **현재 설정된 access-list 확인**
   ```
   show running-config | include access-lists
   ```
3. **ACL(Access List) 생성: PC2 IP 차단**
   ```bash
   # access-list access-list-number {permit | deny} source source-wildcard
   access-list 10 deny 192.168.12.0 0.0.0.255
   access-list 10 permit any
   ```

4. **인터페이스에 ACL 적용 (아웃바운드)**
   ```
   ip access-group 10 out
   ```
   - 해당 인터페이스(예: ethernet 0/1)에서 나가는 트래픽에 ACL 10번 적용

5. **설정 저장**
   ```
   do wr
   ```





