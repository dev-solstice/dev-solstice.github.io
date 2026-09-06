---
tags:
  - platform:bc250
  - topic:linux-kernel
  - topic:gaming
---

# 리눅스 배포판 선택

게이밍의 경우 Bazzite 를 사용하며 터미널 사용에 불편함이 있습니다.
LLM 목적의 경우 Fedora / Debian 등을 사용합니다.

바자이트 : [https://elektricm.github.io/amd-bc250-docs/linux/bazzite/#voltage-configuration](https://elektricm.github.io/amd-bc250-docs/linux/bazzite/#voltage-configuration)

데비안 : [https://elektricm.github.io/amd-bc250-docs/linux/debian](https://elektricm.github.io/amd-bc250-docs/linux/debian/)

아래 세팅은 26.6 기준 ISO로 설치한 환경입니다. 사용한 ISO는 여기서 받을 수 있습니다.

[https://drive.google.com/file/d/1FhA3c8r_7F5RMKaiPxBOQEb7_sve23fL/view?usp=sharing](https://drive.google.com/file/d/1FhA3c8r_7F5RMKaiPxBOQEb7_sve23fL/view?usp=sharing)

# 데비안 초기 설정

- 보드 : AMD BC-250
- OS : Debian Forky, 커널 7.1.12+deb14-amd64

GPU 드라이버, 클럭 언락, LLM 서빙 관련은 GPU 세팅 문서에서 이어집니다.

#### 1. sudo 권한 부여

설치 직후에는 일반 계정에 sudo가 없습니다. root로 들어가서 추가합니다.

```bash
su -
apt update && apt install -y sudo
usermod -aG sudo test
exit
```

그룹 변경 반영을 위해 재부팅합니다.

```bash
systemctl reboot -i
```

#### 2. SSH 서버

```bash
sudo apt update
sudo apt install -y openssh-server
```

최신 데비안은 설정이 /etc/ssh/sshd_config.d/ 로 분리되어 있어서 sshd_config 파일이 짧게 보입니다. 필요한 옵션은 파일 맨 아래에 직접 추가하면 됩니다.

미확인 : 실제 SSH 접속 테스트 결과는 로그에 없습니다. `ssh test@<IP>` 로 확인 필요.

#### 3. 터미널 로그 기록

이후 작업 기록을 남기려면 script를 켜두고 진행합니다.

```bash
script bc250_debian_setup.log
# 작업 진행
exit
```

색상 코드 제거.

```bash
col -b < bc250_debian_setup.log > bc250_debian_setup_clean.txt
```
