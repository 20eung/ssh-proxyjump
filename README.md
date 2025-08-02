# SSH ProxyJump for Windows

- Windows 11 PC를 Jump Host(Proxy Jump)로 구성하는 방법
- 점프 호스트는 내부 네트워크에 있는 서버에 안전하게 접속할 수 있도록 중간에서 SSH 연결을 중계하는 역할
- 리모트 PC1에서 Windwos PC를 통해 리모트 PC2로 접속하는 방법임

## 1. Windows 11 설정

### 1) OpenSSH 서버 설치
- 설정 > 앱 > 선택적 기능
- **"OpenSSH Server"** 설치
- PowerShell(관리자)에서 아래 명령어로 SSH 서비스 시작 및 자동 실행 설정:
```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
```

### 2) sshd_config 설정
- 파일위치: C:\ProgramData\ssh\sshd_config
- 필수 설정들

```conf
PubkeyAuthentication yes
PasswordAuthentication no
AuthorizedKeysFile .ssh/authorized_keys
```

### 3) authorized_keys 파일 설정
- 파일위치: C:\Users\\<윈도우 사용자>\\.ssh\authorized_keys
- 이 점프 호스트에 접속할 리모트 PC1의 user의 rsa public key

### 4) 파일 권한 조정 (관리자 권한)
```powershell
icacls $env:USERPROFILE\.ssh /inheritance:r
icacls $env:USERPROFILE\.ssh\authorized_keys /inheritance:r
icacls $env:USERPROFILE\.ssh /grant "$($env:USERNAME):F"
icacls $env:USERPROFILE\.ssh\authorized_keys /grant "$($env:USERNAME):R"
```
### 5) administrators_authorized_keys 파일
- 파일위치: C:\ProgramData\ssh\administrators_authorized_keys
- 만약 점프 호스트의 사용자 계정이 Administrators 그룹에 속한다면 필수 설정
- 이 점프 호스트에 접속할 리모트 PC1의 user의 rsa public key

### 6) 파일 권한 조정 (관리자 권한)
```powershell
icacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r
icacls "C:\ProgramData\ssh\administrators_authorized_keys" /grant "BUILTIN\Administrators:R"
```

### 7) OpenSSH SSH Server(sshd) 서비스 재시작
```powershell
Restart-Service sshd
```

### 8) Tailscale 설정 및 로그인

### 9) 트러블 슈팅
- C:\ProgramData\ssh\sshd_config에 추가
```conf
LogLevel DEBUG3
```
- 서비스 재시작
```powershell
Restart-Service sshd
```
- 리모트 PC1 에서는 아래 명령어로 접속 시도
```bash
ssh -vvv jumphostuser@<windows_vm_ip> -p 22
```
- 점프 호스트 이벤트 뷰어에서 확인
- 이벤트 뷰어 → Applications and Services Logs → OpenSSH

### 10) 공개키 지문 확인 방법
- Linux/Mac
```bash
ssh-keygen -lf ~/.ssh/id_rsa.pub
```
- Windows
```powershell
ssh-keygen -lf $env:USERPROFILE\.ssh\authorized_keys
```

## 2. Jump Host에 접속하는 리모트 PC1 설정

### 1) .ssh/config
```bash
Host jumphost
  HostName 100.100.x.x  # 점프 호스트 서버의 Tailscale IP
  User your_windows_username
  IdentityFile ~/.ssh/remote1-user_rsa
  IdentitiesOnly yes

Host target-server1
  HostName 192.168.x.x  # 타겟 서버1의 IP
  User traget-server_username
  ProxyJump jumphost

Host target-server2
  HostName 192.168.x.x  # 타겟 서버2의 IP
  User traget-server_username
  IdentityFile ~/.ssh/jumphost-user_rsa
  IdentitiesOnly yes
  ProxyJump jumphost
```

### 2) 접속 방법
```bash
ssh target-server1
ssh target-server2
```
