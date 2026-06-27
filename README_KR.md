[한국어](README_KR.md) | [English](README.md)

# OMX-AI 모방 학습 — ACT 기반 Pick and Place

ROBOTIS OMX-AI 로봇 팔에서 ACT(Action Chunking with Transformers)를 이용해 Pick and Place 태스크를 수행하는 모방 학습 실험입니다.

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [환경 구성](#2-환경-구성)
3. [데이터 수집](#3-데이터-수집)
4. [학습](#4-학습)
5. [실험 결과](#5-실험-결과)
6. [주요 관찰 사항](#6-주요-관찰-사항)
7. [문제 해결](#7-문제-해결)
8. [고찰](#8-고찰)
9. [참고 자료](#9-참고-자료)

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 로봇 | OMX-AI (ROBOTIS) |
| 태스크 | Pick and Place |
| 정책 | ACT (Action Chunking with Transformers) |
| 도구 | Physical AI Tools, LeRobot, ROS2 |
| 데이터셋 | 100 에피소드, 약 45,000 프레임, 30fps, 카메라 1대 |
| HuggingFace 데이터셋 | [cheolhyunkim/omx_f_pick_and_place](https://huggingface.co/datasets/cheolhyunkim/omx_f_pick_and_place) |
| **최종 결과** | **100K 체크포인트에서 6/6 위치 성공 (100% 성공률)** |

인간의 시연을 학습 데이터로 삼아 엔드-투-엔드 모방 학습을 구현하는 것이 목표입니다.

---

## 2. 환경 구성

### 하드웨어

- **로봇**: ROBOTIS OMX-AI (6-DOF)
- **카메라**: APKO APC890W, 30fps
- **학습 GPU**: NVIDIA RTX 5060 8GB (Laptop)
- **미들웨어**: ROS2 Jazzy

### 소프트웨어 환경

| 항목 | 내용 |
|---|---|
| OS | Ubuntu 24.04 LTS |
| ROS2 | Jazzy |
| 컨테이너 | Docker + NVIDIA Container Toolkit |
| Python | 3.12 |
| 학습 프레임워크 | LeRobot (HuggingFace) |
| 로봇 제어 | Physical AI Tools (ROBOTIS) |

### 실행 환경

![터미널 환경](assets/setup/terminal.png)

이 프로젝트에서 사용된 터미널 구성:

- **좌측 상단**: open_manipulator Docker 컨테이너 내부의 OMX 노드
- **우측 상단**: physical_ai_tools Docker 컨테이너 내부의 카메라 노드 (v4l2_camera)
- **하단**: physical_ai_tools Docker 컨테이너 내부의 Physical AI Tools 서버 노드

---

## 3. 데이터 수집

| 항목 | 값 |
|---|---|
| 에피소드 수 | 100 |
| 총 프레임 수 | 약 45,000 |
| 프레임 레이트 | 30 fps |
| 해상도 | 640×480 |
| 카메라 수 | 1 |
| 수집 방식 | 텔레오퍼레이션 (리더-팔로워) |
| 수집 도구 | Physical AI Tools + LeRobot |

각 에피소드는 목표 물체를 집어 지정된 트레이에 올려놓는 단일 Pick-and-Place 사이클로 구성됩니다.

### 데이터 수집 영상


https://github.com/user-attachments/assets/cf648b12-937d-4c4f-a8f6-d8004809b01a


[YouTube에서 보기](https://youtu.be/JBE216r2U6A)

---

## 4. 학습

### 학습 Loss 그래프

![학습 Loss](assets/results/training_loss.png)

**분석:**
- **Total Loss**: 처음 5K step 이내에 급격히 감소하며 약 10K step 이후 안정화
- **L1 Loss** (행동 예측): 약 0.03 수준으로 수렴하며 소폭 변동
- **KLD Loss** (VAE 정규화): 5K step 이내에 거의 0으로 수렴
- KLD의 빠른 수렴은 VAE 인코더가 빠르게 압축된 잠재 표현을 학습했음을 시사

### 하이퍼파라미터

| 항목 | 값 |
|---|---|
| 정책 | ACT |
| 백본 | ResNet-18 |
| 배치 크기 | 32 |
| 학습 스텝 | 100,000 |
| 학습률 | 1e-5 |
| 학습 시간 | 약 18시간 |
| GPU | NVIDIA RTX 5060 8GB (Laptop) |
| 체크포인트 저장 주기 | 20,000 step |

---

## 5. 실험 결과

### 실험 환경

![실험 그리드](assets/results/experiment_grid.jpg)

6개의 고정 위치에 물체를 놓고 체크포인트별로 Pick-and-Place 성공 여부를 테스트했습니다. 위치 인덱스는 아래와 같습니다.

```
로봇
 ↓
[3] [6]
[2] [5]
[1] [4]
```

Position 3이 로봇 베이스에서 가장 가까운 위치이며, Position 4, 5, 6은 로봇에서 먼 쪽입니다.

### 평가 방법

각 체크포인트를 작업 공간의 6개 고정 위치에 물체를 놓고 평가했습니다. 로봇은 물체를 집어 트레이에 놓는 동작을 시도합니다.

**범례:**
- ✅ = 첫 시도 성공
- ⚠️ = 조건부 성공 (재시도 필요 또는 불안정한 그리핑)
- ❌ = 실패

### 위치별 상세 결과

| 체크포인트 | Pos 1 | Pos 2 | Pos 3 | Pos 4 | Pos 5 | Pos 6 | 첫 시도 성공률 |
|---|---|---|---|---|---|---|---|
| 020k | ❌ | ✅ | ✅ | ✅ | ⚠️ 재시도 | ❌ | 3/6 (50%) |
| 040k | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | 5/6 (83%) |
| 060k | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | 5/6 (83%) |
| 080k | ⚠️ 재시도 | ✅ | ✅ | ✅ (불안정) | ✅ | ✅ | 5/6 (83%) |
| 100k | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | **6/6 (100%)** |

### 체크포인트별 추론 영상

#### 020k Steps (50%)


https://github.com/user-attachments/assets/f75c6cb0-0c7c-4a2c-b353-f82cd4706368


[YouTube에서 보기](https://youtu.be/YuQHmx89cCM)

#### 040k Steps (83%)


https://github.com/user-attachments/assets/86a429f1-ef09-4f8b-81c1-7f09a9df7278


[YouTube에서 보기](https://youtu.be/s6fwgxGGtAk)

#### 060k Steps (83%)


https://github.com/user-attachments/assets/4c0c1815-07ea-41c4-a213-9088dcde4e17


[YouTube에서 보기](https://youtu.be/SJT8HtYSkhY)

#### 080k Steps (83%)


https://github.com/user-attachments/assets/6a597adb-4678-4a53-812d-b5b6f3d0a8f5


[YouTube에서 보기](https://youtu.be/jNEdFIn7p0s)

#### 100k Steps (100%)


https://github.com/user-attachments/assets/5f69d5f6-e097-44fb-901f-c06e884674b9


[YouTube에서 보기](https://youtu.be/jQcLh0VwcxY)

---

## 6. 주요 관찰 사항

- **100K 체크포인트에서 6/6 완벽한 성공률 달성** — 충분한 학습 스텝이 모든 위치에서의 강건한 일반화에 핵심적
- **Position 1이 가장 어려운 위치** — 020k, 040k, 060k에서 유일하게 실패, 080k에서야 처음 성공(재시도 필요). 해당 영역의 학습 데이터가 상대적으로 부족했을 가능성
- **040k, 060k, 080k 모두 동일한 83% 성공률** — 중간 학습 구간에서 정체 후 100k에서 최종 도약
- **080k에서 Position 1 첫 성공** — 재시도가 필요했지만 처음으로 완료. Position 4에서는 불안정한 그리핑이 관찰되었으나 첫 시도에 성공
- **성능 향상 패턴 (50% → 83% → 83% → 83% → 100%)** — 중반부 장기 정체 후 최종 체크포인트에서 도약하는 양상
- 초기 체크포인트(020k~060k)의 성공 사례는 중앙 및 먼 영역(Position 2, 3, 4, 5)에 집중. 로봇 근처 영역(Position 1)은 가장 많은 학습이 필요

---

## 7. 문제 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| DYNAMIXEL 모터 무응답 | 케이블 연결 불량으로 FastBulkRead 에러 발생 | Dynamixel Wizard 2.0으로 진단 후 모터 케이블 재연결 |
| v4l2_camera가 잘못된 카메라 캡처 | 리부팅 시 USB 장치 순서 변경, 내장 IR 카메라가 할당됨 | `/sys/class/video4linux/video*/name`으로 올바른 장치 확인 |
| LeRobot 버전 불일치 (Colab vs Docker) | 서로 다른 LeRobot 버전으로 인해 "mean is infinity" 추론 에러 | Colab 환경을 Physical AI Tools Docker 컨테이너와 동일한 LeRobot 버전으로 고정 |
| DroidCam "Connection reset" 에러 | v4l2loopback 커널 모듈 미로드 | `sudo modprobe v4l2loopback` 실행 |
| 학습 CSV 로그 미생성 | 소스 코드 수정 전에 이미 Python 모듈이 메모리에 로드됨 | 트레이너 소스 코드 수정 후 컨테이너 재시작, `os.makedirs()`로 출력 디렉토리 존재 보장 |

---

## 8. 고찰

#### 물체 형상과 그리핑 전략

이전 학습에서는 정사각형 박스를 pick 대상으로 사용했다. 그러나 사각형의 면에 따라 그리퍼가 잡아야 할 각도가 계속 달라지는 문제가 있었다. 그리퍼 위에 근접 카메라가 없어 학습 데이터를 제대로 수집하더라도 올바른 그립 각도를 추론해내지 못했다. 향후 그리퍼에 카메라를 추가하여 물체의 그립 포인트까지 추론하는 태스크를 진행해보고 싶다.

#### 데이터 품질의 중요성

이전 학습에서는 데이터셋 수집 과정에서 그리퍼가 실수로 물체를 떨어뜨리거나, 카메라 앵글 안에 조작자의 손이 자주 들어갔다 나오는 등의 노이즈를 크게 신경쓰지 않고 데이터를 수집했다. 이렇게 노이즈가 많이 포함된 데이터를 바탕으로 학습시켰을 때, 로봇은 시연과 전혀 다른 불규칙한 동작을 보이며 태스크를 제대로 수행하지 못했다. 이번 프로젝트를 통해 데이터의 품질이 추론 성능에 큰 영향을 미친다는 점을 깨닫게 되었다. 깨끗하고 일관된 시연 데이터가 모방학습의 성공에 필수적이다.

---

## 9. 참고 자료

- [ACT: Action Chunking with Transformers](https://tonyzhaozh.github.io/aloha/) — Zhao et al., 2023
- [LeRobot](https://github.com/huggingface/lerobot) — HuggingFace
- [Physical AI Tools](https://github.com/ROBOTIS-GIT/physical_ai_tools) — ROBOTIS
- [ROBOTIS OMX-AI](https://www.robotis.com) — 로봇 하드웨어
- [ROS2](https://docs.ros.org)
- 데이터셋: [cheolhyunkim/omx_f_pick_and_place](https://huggingface.co/datasets/cheolhyunkim/omx_f_pick_and_place)
- 유튜브 재생목록: [OMX-AI ACT Imitation Learning](https://www.youtube.com/playlist?list=PLPGPIB1GJkoY)
