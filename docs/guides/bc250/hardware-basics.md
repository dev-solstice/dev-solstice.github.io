---
tags:
  - platform:bc250
---

# 하드웨어 설정 가이드

BIOS 언락, IOMMU 비활성화 등 이후 모든 문서에서 공통으로 전제하는 하드웨어/펌웨어 설정을 정리합니다.

## 1. 준비물

작업 전에 아래 도구와 부품을 미리 챙겨둡니다.

USB(BIOS 언락 및 리눅스 설치용)

## 2. BIOS 언락

![준비물 1](images/1.jpg){ width="400" }
![준비물 2](images/2.jpg){ width="400" }



전원을 완전히 차단한 상태에서 진행합니다.

![BIOS 언락 과정](images/3.jpg){ width="500" }

## 3. IOMMU 비활성화

커널 부팅 옵션에서 IOMMU 관련 설정을 끕니다.

![IOMMU 설정 화면](images/4.jpg){ width="500" }

## 4. 최종 점검

모든 설정이 끝나면 아래처럼 정상 부팅되는지 확인합니다.

![최종 점검](images/5.jpg){ width="500" }
