---
tags:
  - platform:bc250
  - topic:linux-kernel
  - topic:gaming
---

# GPU 세팅 (vulkan)

bc250은 ROcm이 지원되지 않아 로컬 LLM 구동시 vulkan 환경이 필요합니다.

#### 1. 시스템 업데이트

bash

```bash
sudo apt update && sudo apt upgrade -y
```

#### 2. AMD GPU 드라이버 (AMDGPU + Mesa)

bash

```bash
# Mesa 최신 버전 (RADV Vulkan 드라이버 포함)
sudo apt install -y \
  mesa-vulkan-drivers \
  libdrm-amdgpu1 \
  firmware-amd-graphics
```

#### 3. Vulkan 도구 설치

bash

```bash
sudo apt install -y \  vulkan-tools \  libvulkan-dev \  glslc \  glslang-tools
```

#### 4. BC-250 인식 확인

bash

```bash
# BC-250이 보이면 성공
vulkaninfo --summary 2>/dev/null | grep "deviceName"
```

#### 5. TTM 메모리 설정

bash

```bash
# 즉시 적용
echo 4194304 | sudo tee /sys/module/ttm/parameters/pages_limit

# 재부팅 후에도 유지
echo 'options ttm pages_limit=4194304' | \
  sudo tee /etc/modprobe.d/ttm.conf
sudo update-initramfs -u
```

#### 6. llama.cpp 빌드 의존성

bash

```bash
sudo apt install -y git cmake build-essential
```

#### 7. llama.cpp 빌드

bash

```bash
git clone https://github.com/ggerganov/llama.cppcd llama.cppcmake -B build -DGGML_VULKAN=ONcmake --build build --config Release -j$(nproc)
```

#### 8. 모델 다운로드 도구

bash

```bash
pip install huggingface_hub --break-system-packages
```

#### 9. 모델 다운로드

bash

```bash
huggingface-cli download unsloth/gemma-4-12B-it-GGUF \  --include "gemma-4-12B-it-UD-Q6_K_XL.gguf" \  --local-dir ~/llama.cpp/models
```

#### 10. 실행

bash

```bash
cd ~/llama.cpp./build/bin/llama-server \  -m ./models/gemma-4-12b-it-UD-Q6_K_XL.gguf \  --n-gpu-layers 999 \  -c 16384 \  -ub 512 \  -b 512 \  --no-warmup \  --host 0.0.0.0 \  --port 8080
```