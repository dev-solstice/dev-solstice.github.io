---
tags:
  - platform:bc250
  - topic:linux-kernel
  - topic:gaming
  - topic:llm
---

# 리눅스 배포판 선택

게이밍의 경우 Bazzite 를 사용하며 터미널 사용에 불편함이 있습니다.
LLM 목적의 경우 Fedora / Debian 등을 사용합니다.

바자이트 : [https://elektricm.github.io/amd-bc250-docs/linux/bazzite/#voltage-configuration](https://elektricm.github.io/amd-bc250-docs/linux/bazzite/#voltage-configuration)

데비안 : [https://elektricm.github.io/amd-bc250-docs/linux/debian](https://elektricm.github.io/amd-bc250-docs/linux/debian/)

!!! note "이 가이드에서 사용한 데비안 ISO"
    아래 세팅 기록은 **2026년 6월 기준 데비안 ISO**로 설치한 환경을 바탕으로 합니다.
    같은 ISO는 여기서 받을 수 있습니다.

    📀 [Debian ISO 다운로드 (Google Drive)](https://drive.google.com/file/d/1FhA3c8r_7F5RMKaiPxBOQEb7_sve23fL/view?usp=sharing)

    다른 시점의 ISO를 쓰면 커널 버전이나 Mesa 패키지 버전이 달라 일부 단계(3장 Mesa, 4장 커널)의 결과가 다를 수 있습니다.

---

# 데비안 세팅 기록 (Mesa 26 / GPU 클럭 언락 / Vulkan LLM)

!!! abstract "이 문서의 성격"
    실제 터미널 로그(성공/실패 출력 포함)를 바탕으로 정리한 **작업 기록**입니다.
    각 단계 제목의 아이콘으로 실제 결과를 구분했습니다.

    | 표시 | 의미 |
    |---|---|
    | ✅ 성공 확인 | 터미널 출력으로 정상 동작이 확인된 단계 |
    | 🛠️ 트러블슈팅 후 성공 | 처음엔 에러가 났지만 원인을 찾아 해결한 단계 |
    | ⛔ 폐기됨 | 원문 가이드에 있었지만 이 환경에는 불필요해서 건너뛴 단계 |
    | ⏳ 미확인 | 명령어는 실행했지만 결과(성공 여부)가 로그에 없는 단계 |

## 진행 상황 한눈에 보기

| 단계 | 내용 | 상태 |
|---|---|---|
| 1 | root/sudo 권한 부여 + SSH 서버 설치 | ⏳ 접속 테스트 미확인 |
| 2 | 터미널 로그 기록 (`script`) | ✅ |
| 3 | Mesa 최신 버전 (Experimental + APT Pinning) | ✅ 26.2.1-4 설치 확인 |
| 4 | XanMod 커널 | ⛔ 이미 커널 7.1.12라 불필요 |
| 5 | GRUB `amdgpu.sg_display=0` | 적용함 (재확인 권장) |
| 6 | 절전 모드 차단 | ✅ masked 확인 |
| 7 | GPU 클럭 언락 (cyan-skillfish-governor-smu) | ✅ 2.0GHz 부스트 실측 |
| 8 | 센서 인식 (lm-sensors / nct6683) | ⏳ |
| 9 | Steam 게이밍 환경 | ⏳ |
| 10-1 | Vulkan 툴체인 설치 | 적용함 |
| 10-2 | Vulkan GPU 인식 (render 그룹 권한) | 🛠️ `AMD BC-250 (RADV GFX1013)` 확인 |
| 10-3 | TTM 16GB 메모리 언락 | ✅ |
| 10-4 | llama.cpp Vulkan 빌드 | 🛠️ configure 성공 / 최종 빌드 미확인 |
| 10-5 | 모델 다운로드 | ⏳ 파일명 문제 원인 규명, 다운로드 미확인 |
| 10-6 | llama-server 실행 | ⏳ |

---

## 0. 환경 및 하드웨어 정보

| 항목 | 값 |
|---|---|
| 보드 | AMD BC-250 (PS5용 Oberon APU의 QC 탈락 칩을 재활용한 채굴 보드) |
| GPU | 커널 코드네임 `Cyan Skillfish`, Vulkan 타깃 ID `gfx1013` |
| 메모리 | 16GB GDDR6 통합 메모리(UMA) |
| OS | Debian (Forky 계열) |
| 설치 ISO | 2026년 6월 기준 ISO — [Google Drive 링크](https://drive.google.com/file/d/1FhA3c8r_7F5RMKaiPxBOQEb7_sve23fL/view?usp=sharing) |
| 커널 | `7.1.12+deb14-amd64` (이미 최신, 별도 커널 업그레이드 불필요) |

!!! info "핵심 전략: ROCm 대신 Vulkan"
    `gfx1013`은 RDNA1 계열 ISA라 공식 ROCm 지원 대상이 아닙니다.
    그래서 **Vulkan(RADV) 백엔드로 우회**하는 것이 이 세팅 전체의 핵심입니다. (자세한 이유는 10장 참고)

---

## 1. ⏳ 초기 계정 권한 및 SSH 원격 접속 설정

데비안을 새로 설치한 직후에는 일반 계정(`test`)에 `sudo` 권한이 없고, SSH도 꺼져 있습니다.
본격적인 세팅 전에 계정 권한과 원격 접속부터 처리했습니다.

### 1-1. root로 전환 후 sudo 권한 부여

```bash
su -
```

> 데비안 설치 시 설정한 **root 비밀번호**를 입력합니다.

```bash
apt update && apt install -y sudo
usermod -aG sudo test
exit
```

### 1-2. 재부팅하여 그룹 변경 반영

그룹 변경은 새 세션에서만 반영되므로 재부팅으로 바로 적용했습니다.

```bash
systemctl reboot -i
# 또는
reboot -f
```

재부팅 후 `test` 계정으로 로그인하면 `sudo`가 정상 동작합니다. **이후 모든 단계는 `sudo`를 전제로 합니다.**

### 1-3. SSH 서버 설치

```bash
sudo apt update
sudo apt install -y openssh-server
```

!!! tip "최신 데비안의 sshd_config가 짧게 보이는 이유"
    Debian 12 Bookworm 이후에는 설정이 `/etc/ssh/sshd_config.d/` 디렉터리로 분리되어 있습니다.
    기존 파일에 원하는 옵션 문구가 없어도, 파일 맨 아래에 직접 한 줄 추가하면 정상 반영됩니다.

!!! warning "⏳ 미확인"
    sudo 그룹 추가, 재부팅, SSH 설치까지는 확인되지만, 실제 SSH 접속 테스트 결과나
    `sshd_config`에 어떤 옵션(`PasswordAuthentication`, `PermitRootLogin` 등)을 넣었는지는 캡처되지 않았습니다.
    `ssh test@<BC-250의 IP>`로 직접 접속 테스트를 권장합니다.

---

## 2. ✅ 사전 준비: 터미널 세션 전체 기록

작업 중 모든 입출력을 파일로 남기기 위해 `script`를 사용했습니다.

```bash
script bc250_debian_setup.log   # 기록 시작
# ... 이 아래에서 모든 설치 작업 진행 ...
exit                            # 기록 종료
```

ANSI 색상 코드를 제거한 순수 텍스트가 필요하면:

```bash
col -b < bc250_debian_setup.log > bc250_debian_setup_clean.txt
```

---

## 3. ✅ Mesa 최신 드라이버 설치 (Experimental 저장소 + APT Pinning)

데비안 안정판 기본 Mesa는 `gfx1013` 파이프라인을 제대로 지원하지 않습니다.
**시스템 전체는 그대로 두고 Mesa 드라이버만** Experimental 저장소에서 끌어옵니다.

### 3-1. Experimental 저장소 추가

```bash
sudo nano /etc/apt/sources.list
```

```title="/etc/apt/sources.list (추가)"
deb http://deb.debian.org/debian experimental main contrib non-free non-free-firmware
```

### 3-2. APT Pin 설정

시스템 전체가 experimental 패키지로 오염되는 것을 막습니다.

```bash
sudo nano /etc/apt/preferences.d/experimental
```

```title="/etc/apt/preferences.d/experimental"
Package: *
Pin: release a=experimental
Pin-Priority: 1

Package: mesa-vulkan-drivers libgl1-mesa-dri
Pin: release a=experimental
Pin-Priority: 500
```

- `Pin-Priority: 1` → experimental의 **모든 패키지를 기본 차단**
- `Pin-Priority: 500` → **Mesa 드라이버만 예외적으로 허용**

### 3-3. 갱신 및 확인

```bash
sudo apt update
apt-cache policy mesa-vulkan-drivers
```

!!! success "실제 결과"
    - `apt update`: experimental 저장소가 반영되었고 `13 packages can be upgraded`만 표시. 수백 개가 아니라 필요한 것만 후보로 잡힘 → **핀 설정 정상 작동**
    - `apt-cache policy`: `Candidate: 26.2.1-4` (experimental, priority 500)

### 3-4. 설치

```bash
sudo apt install -t experimental mesa-vulkan-drivers libgl1-mesa-dri
```

!!! success "실제 결과"
    `libegl-mesa0`, `libgbm1`, `libgl1-mesa-dri`, `libglx-mesa0`, `mesa-libgallium`, `mesa-vulkan-drivers`가
    **26.1.6-1 → 26.2.1-4**로 에러 없이 업그레이드됨.

### 3-5. 확인 도구 설치

`glxinfo`는 드라이버가 아니라 별도 유틸리티 패키지에 들어있어, 처음엔 `glxinfo: command not found`가 발생했습니다.

```bash
sudo apt install mesa-utils vulkan-tools
glxinfo -B
```

---

## 4. ⛔ XanMod 커널 설치 — 시도했지만 폐기

원문 가이드에는 최신 그래픽 지원을 위해 XanMod 커널을 설치하라고 되어 있었지만, 실제로는 **불필요했습니다.**

??? failure "시도한 것 (모두 실패) — 펼쳐보기"
    ```bash
    sudo apt install linux-image-6.12
    # → Error: Unable to locate package linux-image-6.12 (패키지명 자체가 잘못됨)

    wget -qO - https://dl.xanmod.org/archive.key | sudo gpg --dearmor -o /usr/share/keyrings/xanmod-archive-keyring.gpg
    echo 'deb [signed-by=/usr/share/keyrings/xanmod-archive-keyring.gpg] http://deb.xanmod.org releases main' | sudo tee /etc/apt/sources.list.d/xanmod-kernel.list
    sudo apt update
    # → Err: 404 Not Found (구주소, deb.xanmod.org)

    # 최신 주소로 재시도해도 마찬가지
    echo 'deb [signed-by=/usr/share/keyrings/xanmod-archive-keyring.gpg] https://dl.xanmod.org/debian/ releases main' | sudo tee /etc/apt/sources.list.d/xanmod-kernel.list
    sudo apt update
    # → 여전히 404 Not Found, Unable to locate package linux-xanmod-lts-x64v3
    ```

### 원인 파악 후 폐기 결정

```bash
uname -r
# → 7.1.12+deb14-amd64
```

현재 데비안(Forky)에 이미 XanMod보다 훨씬 최신인 커널이 기본 탑재되어 있어 이 단계를 완전히 건너뛰었습니다.

```bash
# 정리: 에러 나던 저장소 파일 삭제
sudo rm -f /etc/apt/sources.list.d/xanmod-kernel.list
```

!!! tip "교훈"
    오래된 블로그/가이드의 커널 버전 요구사항은 참고만 하고,
    먼저 `uname -r`로 현재 커널이 이미 충분히 최신인지 확인하세요.

---

## 5. GRUB 커널 파라미터 — 블랙스크린 방지

BC-250은 Scatter-Gather Display 기능이 켜져 있으면 화면이 깜빡이거나 블랙스크린이 뜨는 알려진 결함이 있습니다.

```bash
sudo nano /etc/default/grub
```

```title="/etc/default/grub"
GRUB_CMDLINE_LINUX_DEFAULT="quiet amdgpu.sg_display=0"
```

```bash
sudo update-grub
sudo reboot
```

- `quiet` → 부팅 로그 숨김 (GPU와 무관)
- `amdgpu.sg_display=0` → **핵심**. 메모리 매핑 방식을 우회해 화면 출력 안정화

!!! warning "⏳ 재부팅 후 결과는 별도 캡처 없음"
    BC-250 커뮤니티 가이드의 필수 항목이라 계속 유지했습니다.

---

## 6. ✅ 절전(Suspend) 완전 차단

세팅 도중 **15분마다 시스템이 자동으로 절전 모드에 들어가는 현상**이 실제로 발생했습니다.

### 6-1. 원인 확인

```bash
journalctl -u systemd-suspend.service -b --no-pager | tail -n 20
```

!!! success "실제 결과"
    `Starting systemd-suspend.service` / `System returned from sleep operation` 로그가 **정확히 15분 간격**으로 반복.
    `/etc/systemd/logind.conf`는 전부 주석 처리된 비활성 상태였고, 실제 원인은 **GNOME 데스크톱의 기본 15분 절전 정책**이었습니다.

### 6-2. 조치

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
systemctl status sleep.target suspend.target
```

!!! success "실제 결과"
    ```
    Created symlink '/etc/systemd/system/sleep.target' → '/dev/null'.
    Created symlink '/etc/systemd/system/suspend.target' → '/dev/null'.
    Created symlink '/etc/systemd/system/hibernate.target' → '/dev/null'.
    Created symlink '/etc/systemd/system/hybrid-sleep.target' → '/dev/null'.
    ```
    `status`에서 `Loaded: masked` 확인 → **이후 절전 모드로 빠지지 않음.**

되돌리려면:

```bash
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

---

## 7. ✅ GPU 클럭 잠금 해제 — cyan-skillfish-governor-smu

BC-250은 SMU(전력 관리 장치) 통신 결함으로 GPU 코어 클럭이 **400~800MHz 최저 구간에 영구 고정**되는 문제가 있습니다.
이를 해결하는 오픈소스 데몬([filippor/cyan-skillfish-governor](https://github.com/filippor/cyan-skillfish-governor))을 설치했습니다.

!!! note "소스 빌드 대신 .deb 패키지 사용"
    처음 검토한 방법은 `git clone` + `cargo build --release`였지만,
    **실제로는 GitHub Releases의 미리 빌드된 `.deb`를 받아 설치**하는 훨씬 간단한 방법을 썼습니다.

### 7-1. .deb 다운로드 및 설치

```bash
wget https://github.com/filippor/cyan-skillfish-governor/releases/download/v0.4.12/cyan-skillfish-governor-smu_0.4.12-1_amd64.deb
sudo apt install ./cyan-skillfish-governor-smu_0.4.12-1_amd64.deb
```

!!! success "실제 결과"
    정상 설치 완료. 설치와 동시에 systemd 자동 실행(enable)까지 심볼릭 링크로 등록됨.
    `missing 'Maintainer' field` 경고는 개인 빌드 패키지 특성상 나오는 무해한 경고입니다.

### 7-2. 서비스 상태 확인

```bash
sudo systemctl enable --now cyan-skillfish-governor-smu.service
systemctl status cyan-skillfish-governor-smu
# → Active: active (running)
```

### 7-3. 클럭 상승 확인 (가장 중요한 검증)

```bash
cat /sys/class/drm/card0/device/pp_dpm_sclk
```

!!! success "실제 결과"
    ```
    0: 1000Mhz *
    1: 1500Mhz
    2: 2000Mhz
    ```
    400~800MHz에 갇혀 있던 클럭이 **최대 2.0GHz까지 부스트 가능한 상태**로 전환됨을 실측 확인.

!!! info "벽돌(영구 고장) 위험 없음"
    바이오스 오버클럭이 아니라, AMD가 이미 안전 범위로 검증해 둔 DPM 프로파일(1000/1500/2000MHz) 사이를
    소프트웨어가 전환해 주는 방식입니다. 서비스를 끄면 즉시 원상 복구됩니다.

---

## 8. ⏳ 메인보드 센서 인식 — lm-sensors / NCT6683

BC-250에는 Nuvoton NCT6683 계열 센서 칩이 있지만, 산업용 보드 특성상 정규 인증 정보가 없어 커널이 자동 인식하지 않습니다.

```bash
sudo apt install lm-sensors
echo 'nct6683' | sudo tee /etc/modules-load.d/nct6683.conf
echo 'options nct6683 force=true' | sudo tee /etc/modprobe.d/sensors.conf
sudo modprobe nct6683 force=true
sensors
```

!!! warning "⏳ 미확인"
    명령 실행까지는 기록되어 있지만 `sensors` 출력(온도/팬)이 캡처되지 않아 최종 성공 여부는 미확인입니다.
    재부팅 후 `sensors`로 직접 재확인을 권장합니다.

---

## 9. ⏳ Steam 게이밍 환경 구축

게이밍 전용 OS(Bazzite 등)를 새로 설치하지 않고, 지금까지 세팅한 데비안 위에 Steam을 얹었습니다.

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install -y steam-installer mesa-vulkan-drivers mesa-vulkan-drivers:i386 libglx-mesa0:i386 mangohud gamemode
```

Steam 게임 속성 → 시작 옵션에 아래를 넣으면 오버레이(FPS/온도/클럭)와 성능 최적화가 동시에 적용됩니다.

```
gamemoderun mangohud %command%
```

!!! warning "⏳ 미확인"
    설치 성공 로그가 캡처되지 않았습니다. 32비트 패키지명이 데비안 버전에 따라 다를 수 있으니
    `steam` 대신 `steam-installer`로 뜨는지 확인하세요.

---

## 10. Vulkan 기반 로컬 LLM 환경 구축 (llama.cpp)

!!! question "왜 ROCm이 아니라 Vulkan인가?"
    `gfx1013`(BC-250)은 겉보기엔 RDNA2 계열이지만 실제 연산 ISA는 RDNA1 세대에 가깝고, 행렬 가속 유닛(WMMA)도 없습니다.
    AMD 공식 ROCm 지원은 6000번대(gfx1030)까지가 마지노선이라 gfx1013은 **공식적으로 영구 제외**되어 있고,
    커널 프리징 문제까지 있어 사실상 사용 불가능합니다.
    반면 Mesa의 RADV Vulkan 드라이버는 gfx1013을 정상 지원하므로, `llama.cpp`의 Vulkan 백엔드로 GPU 가속 추론을 구현했습니다.

### 10-1. Vulkan 툴체인 설치

```bash
sudo apt install -y mesa-vulkan-drivers libdrm-amdgpu1 firmware-amd-graphics
sudo apt install -y vulkan-tools libvulkan-dev glslang-tools
```

!!! tip "복사-붙여넣기 함정: NBSP"
    줄 끝 `\` 뒤에 보이지 않는 특수 공백(NBSP)이 섞이면 `E: Unable to locate package` 같은 이상한 에러가 납니다.
    **여러 줄로 나뉜 명령은 한 줄로 붙여서 실행**하는 것이 안전합니다.

### 10-2. 🛠️ GPU 인식 실패 → 원인 추적 → 해결 (핵심 트러블슈팅)

**증상**: 툴체인 설치 후 확인해 보니 하드웨어가 아니라 CPU 소프트웨어 렌더러가 잡힘.

```bash
vulkaninfo --summary 2>/dev/null | grep "deviceName"
# → deviceName = llvmpipe (LLVM 21.1.8, 256 bits)   ← GPU 미인식, CPU 에뮬레이션 상태
```

**점검 순서 (실제로 확인한 것들):**

1. `sudo`로 실행 → `X11 connection rejected because of wrong authentication`
   (루트 권한으로 디스플레이 인증 실패, 일반 계정으로 재시도)
2. `lspci -nnk -d 1002:*` → 하드웨어 자체는 정상 인식
   ```
   01:00.0 VGA compatible controller: AMD/ATI Cyan Skillfish [BC-250] [1002:13fe]
   Kernel driver in use: amdgpu
   ```
3. `ls -l /dev/dri` → `renderD128` 노드도 정상 존재 (소유 그룹: `render`)
4. `VK_LOADER_DEBUG=all vulkaninfo --summary` (에러 숨김 해제) → **결정적 단서**
   ```
   WARNING: [...radv_physical_device.c] Could not open device /dev/dri/renderD128: Permission denied (VK_ERROR_INCOMPATIBLE_DRIVER)
   ERROR: setup_loader_term_phys_devs: Failed to detect any valid GPUs in the current config
   ```
5. `groups` → 계정이 `video` 그룹에는 있었지만 **`render` 그룹에는 빠져 있었음**

**해결:**

```bash
sudo usermod -aG render $USER
newgrp render   # 재부팅 없이 즉시 세션에 반영
vulkaninfo --summary 2>/dev/null | grep "deviceName"
```

!!! success "실제 결과"
    ```
    deviceName         = AMD BC-250 (RADV GFX1013)
    ```
    **하드웨어 GPU 정상 인식.** 영구 적용을 위해 이후 `sudo reboot`를 권장합니다.

### 10-3. ✅ TTM 메모리 16GB 전체 언락

커널의 TTM(Translation Table Manager)은 기본적으로 GPU가 시스템 RAM의 절반까지만 쓰도록 제한합니다.
12B급 모델을 온전히 올리려면 16GB 전체를 열어줘야 합니다.

```bash
# 즉시 적용
echo 4194304 | sudo tee /sys/module/ttm/parameters/pages_limit

# 재부팅 후에도 유지
echo 'options ttm pages_limit=4194304' | sudo tee /etc/modprobe.d/ttm.conf
sudo update-initramfs -u
```

!!! success "실제 결과"
    `4194304` 정상 반영, `update-initramfs: Generating /boot/initrd.img-7.1.12+deb14-amd64`로 initramfs 갱신 완료.
    확인: `cat /sys/module/ttm/parameters/pages_limit` → `4194304`

### 10-4. 🛠️ llama.cpp Vulkan 빌드 (SPIRV-Headers 에러 → 해결)

```bash
sudo apt install -y git cmake build-essential python3-pip
```

!!! warning "반드시 한 줄씩 따로 실행"
    여러 줄 명령을 복사할 때 줄바꿈 없이 붙어버려(`git clone ...llama.cppcd llama.cppcmake -B build...`)
    `error: unknown switch 'B'` 에러가 났었습니다.

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_VULKAN=ON
```

!!! failure "첫 시도 실패"
    ```
    CMake Error at ggml/src/ggml-vulkan/CMakeLists.txt:14 (find_package):
      Could not find a package configuration file provided by "SPIRV-Headers"
    ```

**해결:**

```bash
sudo apt install -y spirv-headers spirv-tools
rm -rf build
cmake -B build -DGGML_VULKAN=ON
```

!!! success "실제 결과"
    ```
    -- Found Vulkan: ... found components: glslc glslangValidator
    -- Including Vulkan backend
    -- Configuring done (2.2s)
    -- Generating done (0.3s)
    -- Build files have been written to: /home/test/llama.cpp/build
    ```
    `OpenSSL not found` 경고는 서버 HTTPS 기능용이라 로컬 사용 시 무시 가능.

```bash
cmake --build build --config Release -j$(nproc)
```

!!! warning "⏳ 빌드 완료 자체는 미확인"
    configure 단계는 성공이 확인되었지만, 컴파일 완료 로그(`[100%] Built target llama-server`)는 캡처되지 않았습니다.
    `ls build/bin/`으로 `llama-server` 바이너리 존재를 직접 확인하세요.

### 10-5. ⏳ 모델 다운로드 — 여러 차례 실패 후 원인 규명

```bash
pip install huggingface_hub --break-system-packages
```

!!! failure "실패 1: `huggingface-cli: command not found`"
    **원인**: pip가 `~/.local/bin`에 설치했는데 `$PATH`에 등록되지 않음.
    ```bash
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    ```

!!! failure "실패 2: `--include` 옵션이 무시되어 0바이트 다운로드"
    `hf download ... --include "gemma-4-12B-it-UD-Q6_K_XL.gguf" ...` → `Fetching 0 files: 0it`
    **원인**: NBSP 공백 오염으로 `--include`가 무시되고 잘못된 파일명이 인자로 들어감.

!!! failure "실패 3: 파일명을 직접 지정해도 `File not found in repository`"
    ```bash
    hf download unsloth/gemma-4-12B-it-GGUF gemma-4-12B-it-UD-Q6_K_XL.gguf --local-dir ~/llama.cpp/models
    ```
    **원인**: 저장소 이름(`gemma-4-12B-it-GGUF`, 대문자 B)은 실제로 존재하지만,
    **저장소 안의 실제 파일명은 소문자(`12b`)**였습니다.

**원인 확인 과정** (최신 `hf` CLI는 서브커맨드 구조가 달라 두 번 헛발질):

```bash
hf api list-repo-files unsloth/gemma-4-12B-it-GGUF   # → Error: No such command 'api'
hf repos files unsloth/gemma-4-12B-it-GGUF           # → Error: No such command 'files'

# 최종적으로 성공한 방법: Python 직접 호출
python3 -c "from huggingface_hub import list_repo_files; print('\n'.join(list_repo_files('unsloth/gemma-4-12B-it-GGUF')))"
```

!!! success "실제 결과"
    파일 목록이 정상 출력되었고, 원하던 양자화 파일은 **`gemma-4-12b-it-UD-Q6_K_XL.gguf`** (소문자 `12b`)임을 확인.

**최종 명령 (⏳ 실행 결과는 로그에 없음):**

```bash
hf download unsloth/gemma-4-12B-it-GGUF gemma-4-12b-it-UD-Q6_K_XL.gguf --local-dir ~/llama.cpp/models
```

### 10-6. ⏳ 추론 서버 실행

```bash
cd ~/llama.cpp
./build/bin/llama-server \
  -m ./models/gemma-4-12b-it-UD-Q6_K_XL.gguf \
  --n-gpu-layers 999 \
  -c 16384 \
  -ub 512 \
  -b 512 \
  --host 0.0.0.0 \
  --port 8080
```

!!! warning "파일명 대소문자 주의"
    `-m` 경로는 실제 다운로드된 파일명과 반드시 맞춰야 합니다. `ls ~/llama.cpp/models`로 확인하세요.

---

## 11. 다음에 이어서 할 일

- [ ] `sensors` 명령으로 온도/팬 인식 결과 직접 확인
- [ ] Steam 설치 완료 여부 및 `gamemoderun mangohud %command%` 적용 테스트
- [ ] `ls ~/llama.cpp/build/bin/`으로 `llama-server` 바이너리 생성 여부 확인
- [ ] 모델 다운로드 완료 확인 (`ls -lh ~/llama.cpp/models`)
- [ ] `llama-server` 실행 후 로그에서 `Vulkan0: ... Cyan Skillfish` 등 GPU 오프로딩 확인
