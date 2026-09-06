---
tags:
  - platform:bc250
  - topic:gaming
---

# 게이밍 환경 구성

Bazzite 같은 게이밍 전용 OS를 새로 깔지 않고, 데비안 위에 Steam을 올리는 방법입니다.
GPU 드라이버(Mesa), 클럭 언락은 GPU 세팅 문서를 먼저 진행해야 합니다.

#### 1. Steam 설치

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install -y steam-installer mesa-vulkan-drivers mesa-vulkan-drivers:i386 libglx-mesa0:i386 mangohud gamemode
```

32비트 패키지명은 데비안 버전에 따라 다를 수 있습니다. steam 대신 steam-installer 로 잡히는지 확인하세요.

미확인 : 설치 결과 로그가 없습니다.

#### 2. 게임 시작 옵션

Steam 게임 속성 → 시작 옵션에 추가하면 오버레이(FPS/온도/클럭)와 gamemode가 같이 적용됩니다.

```
gamemoderun mangohud %command%
```
