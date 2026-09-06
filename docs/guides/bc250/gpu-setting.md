---
tags:
  - platform:bc250
  - topic:linux-kernel
  - topic:gaming
  - topic:llm
---

# GPU 세팅 (vulkan)

bc250은 ROCm이 지원되지 않아 로컬 LLM 구동시 vulkan 환경이 필요합니다.

실제로 진행하면서 남긴 기록입니다. 확인이 안 된 단계는 따로 표시했습니다.

- GPU : Cyan Skillfish (gfx1013)
- 메모리 : 16GB GDDR6 (UMA)
- OS : Debian Forky, 커널 7.1.12+deb14-amd64
- gfx1013은 RDNA1 계열 ISA라 ROCm 공식 지원에서 제외되어 있습니다. Mesa RADV는 정상 지원하므로 Vulkan으로 우회합니다.

#### 1. Mesa 최신 버전 설치 (experimental + pinning)

안정판 Mesa는 gfx1013을 제대로 지원하지 않습니다. Mesa만 experimental에서 받아옵니다.

/etc/apt/sources.list 에 추가.

```
deb http://deb.debian.org/debian experimental main contrib non-free non-free-firmware
```

/etc/apt/preferences.d/experimental 생성. experimental 전체는 막고 Mesa만 허용합니다.

```
Package: *
Pin: release a=experimental
Pin-Priority: 1

Package: mesa-vulkan-drivers libgl1-mesa-dri
Pin: release a=experimental
Pin-Priority: 500
```

```bash
sudo apt update
apt-cache policy mesa-vulkan-drivers
```

apt update 후 업그레이드 후보가 13개만 잡히면 pin이 정상 동작하는 것입니다. policy에서 Candidate가 26.2.1-4 (experimental, 500)로 나왔습니다.

```bash
sudo apt install -t experimental mesa-vulkan-drivers libgl1-mesa-dri
```

26.1.6-1 → 26.2.1-4 로 정상 업그레이드됨.

glxinfo는 별도 패키지입니다.

```bash
sudo apt install mesa-utils vulkan-tools
glxinfo -B
```

#### 2. XanMod 커널 (불필요)

원문 가이드에는 XanMod 커널을 올리라고 되어 있지만 필요 없었습니다.

```bash
uname -r
# 7.1.12+deb14-amd64
```

이미 커널이 충분히 최신이라 건너뛰었습니다. 참고로 deb.xanmod.org, dl.xanmod.org 둘 다 404가 나서 어차피 설치도 안 됐습니다.

```bash
sudo rm -f /etc/apt/sources.list.d/xanmod-kernel.list
```

#### 3. GRUB 파라미터 (블랙스크린 방지)

/etc/default/grub 수정.

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amdgpu.sg_display=0"
```

```bash
sudo update-grub
sudo reboot
```

amdgpu.sg_display=0 은 BC-250 화면 깜빡임/블랙스크린 방지용 필수 옵션입니다.

#### 4. 절전 모드 차단

세팅 중 15분마다 자동으로 suspend에 들어가는 현상이 있었습니다.

```bash
journalctl -u systemd-suspend.service -b --no-pager | tail -n 20
```

15분 간격으로 suspend 로그가 반복됨. logind.conf는 전부 주석 상태였고 원인은 GNOME 기본 절전 정책이었습니다.

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
systemctl status sleep.target suspend.target
```

Loaded: masked 확인. 이후 절전으로 안 빠집니다.

되돌릴 때.

```bash
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

#### 5. GPU 클럭 언락 (cyan-skillfish-governor-smu)

BC-250은 SMU 통신 문제로 GPU 클럭이 400~800MHz에 고정됩니다. 아래 데몬으로 해결합니다.

[https://github.com/filippor/cyan-skillfish-governor](https://github.com/filippor/cyan-skillfish-governor)

소스 빌드(cargo) 대신 Releases의 .deb를 받아 설치했습니다.

```bash
wget https://github.com/filippor/cyan-skillfish-governor/releases/download/v0.4.12/cyan-skillfish-governor-smu_0.4.12-1_amd64.deb
sudo apt install ./cyan-skillfish-governor-smu_0.4.12-1_amd64.deb
```

missing 'Maintainer' field 경고는 무시해도 됩니다.

```bash
sudo systemctl enable --now cyan-skillfish-governor-smu.service
systemctl status cyan-skillfish-governor-smu
```

클럭 확인.

```bash
cat /sys/class/drm/card0/device/pp_dpm_sclk
```

```
0: 1000Mhz *
1: 1500Mhz
2: 2000Mhz
```

최대 2.0GHz까지 올라가는 것 확인. 바이오스 오버클럭이 아니라 AMD DPM 프로파일 사이를 전환하는 방식이라 서비스 끄면 원상복구됩니다.

#### 6. 센서 인식 (nct6683)

```bash
sudo apt install lm-sensors
echo 'nct6683' | sudo tee /etc/modules-load.d/nct6683.conf
echo 'options nct6683 force=true' | sudo tee /etc/modprobe.d/sensors.conf
sudo modprobe nct6683 force=true
sensors
```

미확인 : sensors 출력이 로그에 없습니다.

#### 7. Vulkan 툴체인

```bash
sudo apt install -y mesa-vulkan-drivers libdrm-amdgpu1 firmware-amd-graphics
sudo apt install -y vulkan-tools libvulkan-dev glslang-tools
```

여러 줄로 나뉜 명령을 복사하면 `\` 뒤에 NBSP가 섞여서 Unable to locate package 에러가 납니다. 한 줄로 붙여서 실행하세요.

#### 8. Vulkan GPU 인식 안 되는 문제

```bash
vulkaninfo --summary 2>/dev/null | grep "deviceName"
# deviceName = llvmpipe (LLVM 21.1.8, 256 bits)
```

llvmpipe면 GPU가 아니라 CPU 에뮬레이션입니다.

확인한 순서.

```bash
lspci -nnk -d 1002:*
# 01:00.0 VGA compatible controller: AMD/ATI Cyan Skillfish [BC-250] [1002:13fe]
# Kernel driver in use: amdgpu

ls -l /dev/dri
# renderD128 존재, 그룹 render

VK_LOADER_DEBUG=all vulkaninfo --summary
# Could not open device /dev/dri/renderD128: Permission denied

groups
# video는 있는데 render가 없음
```

sudo로 vulkaninfo를 돌리면 X11 인증 에러가 나니 일반 계정으로 해야 합니다.

해결.

```bash
sudo usermod -aG render $USER
newgrp render
vulkaninfo --summary 2>/dev/null | grep "deviceName"
# deviceName = AMD BC-250 (RADV GFX1013)
```

#### 9. TTM 메모리 16GB 언락

기본값은 RAM 절반까지만 GPU가 씁니다.

```bash
echo 4194304 | sudo tee /sys/module/ttm/parameters/pages_limit

echo 'options ttm pages_limit=4194304' | sudo tee /etc/modprobe.d/ttm.conf
sudo update-initramfs -u
```

```bash
cat /sys/module/ttm/parameters/pages_limit
# 4194304
```

#### 10. llama.cpp Vulkan 빌드

```bash
sudo apt install -y git cmake build-essential python3-pip
```

한 줄씩 따로 실행해야 합니다. 붙여서 복사하면 `error: unknown switch 'B'` 가 납니다.

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_VULKAN=ON
```

SPIRV-Headers 를 못 찾는다는 에러가 나면.

```bash
sudo apt install -y spirv-headers spirv-tools
rm -rf build
cmake -B build -DGGML_VULKAN=ON
```

Including Vulkan backend 가 뜨면 성공. OpenSSL not found 경고는 무시.

```bash
cmake --build build --config Release -j$(nproc)
```

미확인 : 빌드 완료 로그가 없습니다. `ls build/bin/` 에 llama-server가 있는지 확인 필요.

#### 11. 모델 다운로드

```bash
pip install huggingface_hub --break-system-packages
```

huggingface-cli: command not found 가 나면 PATH 문제.

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

파일명 대소문자 주의. 저장소 이름은 12B (대문자)인데 실제 파일명은 12b (소문자)입니다. 대문자로 넣으면 File not found in repository 가 납니다.

파일 목록 확인. hf api, hf repos files 는 최신 CLI에서 안 됩니다.

```bash
python3 -c "from huggingface_hub import list_repo_files; print('\n'.join(list_repo_files('unsloth/gemma-4-12B-it-GGUF')))"
```

```bash
hf download unsloth/gemma-4-12B-it-GGUF gemma-4-12b-it-UD-Q6_K_XL.gguf --local-dir ~/llama.cpp/models
```

미확인 : 다운로드 완료 로그가 없습니다. `ls -lh ~/llama.cpp/models` 로 확인.

#### 12. 실행

```bash
cd ~/llama.cpp
./build/bin/llama-server -m ./models/gemma-4-12b-it-UD-Q6_K_XL.gguf --n-gpu-layers 999 -c 16384 -ub 512 -b 512 --host 0.0.0.0 --port 8080
```

미확인 : 아직 실행 로그 없음. 실행 후 로그에 Vulkan0 로 Cyan Skillfish가 잡히는지 확인.
