# PX4_FMU_V6C-V6C21_Drone-Benign-Flight-Test

# PX4 드론 정상 비행 로그 데이터 및 GPS 공격 탐지 연구

---

## 1. 실험 개요

### 1.1 실험 환경
- **드론 모델**: PX4_FMU_V6C (V6C21)  
- **비행 컨트롤러**: PX4 Autopilot (EKF2 기반 자세 추정)  
- **지상통제소(GCS)**: QGroundControl  
- **비행 장소**: 경희대학교 럭비장 (야외 오픈 필드-GPS 수신 원활)  
- **비행 시간**: 약 2분 (배터리 70% 이상 유지)  
- **비행 조건**: **정상 비행(No Attack)** 상태에서 수집  

### 1.2 연구 목적
1. **정상 비행 로그**를 기반으로 PX4 드론이 기록하는 센서 및 상태 정보를 기술적으로 분석  
2. **GPS 스푸핑(위치 조작)**, **GPS 재밍(신호 방해)** 공격에 대응할 수 있는 **주요 피처**를 식별  
3. 향후 **TinyML 기반 이상 탐지 모델** 적용 시, 어떤 데이터가 공격 탐지에 유용한지 방향성 제시  

---

## 2. 데이터 수집 및 변환

### 2.1 ULog 파일 수집
- PX4 펌웨어는 비행 중 **센서(IMU, 자이로, GPS 등)**, 비행 상태(배터리, 모터 출력, 비행 모드) 정보를 `*.ulg` 형식으로 기록  
- 비행 종료 후 **QGroundControl** 또는 **Flight Review** 기능을 통해 `ULog` 파일을 다운로드  

### 2.2 ULog → CSV 변환
- `ULog`는 **바이너리 형식**으로 사람이 직접 읽기 어려움  
- **pyulog** 라이브러리를 활용해 CSV 파일로 변환:

---
## 3. 정상 비행 데이터 분석 

### 3.1 주요 센서별 특징 (정상 상태)

### 3.1.1 GPS
1. **satellites_used**: 보통 12~17개의 위성을 사용
2. **fix_type**: 3D Fix (=4) 상태를 일정하게 유지
3. **eph, epv**: 수평·수직 정확도가 일정 범위(약 1~5 사이) 내에서 유지
4. **noise_per_ms, jamming_indicator**: 정상 구간에서는 과도한 잡음이나 방해 지표가 거의 없음

### 3.1.2 IMU (Roll, Pitch, Yaw)
- 드론의 좌우 기울기(Roll), 전후 기울기(Pitch), 회전(Yaw)을 측정
- 정상 비행에서는 **Setpoint(목표 각도)**와 **Estimated(실제 각도)**가 유사하게 유지
- 급격한 변화 없이 일정한 범위 내에서 움직임

### 3.1.3 고도(Altitude)
- GPS Altitude, Barometer Altitude, Fused Altitude가 큰 차이 없이 거의 일치
- 착륙 혹은 급상승 구간 외엔 완만한 변동을 보임

---
## 4. GPS 공격(스푸핑·재밍) 탐지를 위한 주요 피처

### 4.1 GPS Spoofing (위치 조작)
- lat, lon, alt: 정상 경로에서 벗어나 급격히 튀는 값 발생 시 의심
- satellites_used: 비정상적으로 많은 위성을 잡거나, 갑자기 값 변동 발생 가능
- eph, epv: GPS 조작으로 인해 정확도 지표가 갑자기 악화
- fix_type: 3D → 2D 혹은 No Fix 등으로 변할 수 있음

### 4.2 GPS Jamming (신호 방해)
- jamming_indicator: 값이 0 → 10 이상으로 급등하면 재밍 가능성
- noise_per_ms: 잡음 세기가 급격히 증가 → 재밍 징후
- satellites_used: 재밍으로 위성 수가 급격히 줄어들 수 있음
- fix_type: GPS 고정 해제(3D → 0)로 급격히 바뀔 가능성

---

## 6. 결론 및 향후 과제
- 정상 비행 로그 분석을 통해 각 센서별 정상 패턴을 파악
- GPS 스푸핑·재밍 시 주요 피처(위성 개수, jamming_indicator, noise_per_ms, eph, epv 등) 에서 급격한 변동 발생
- 단순 임계값 접근 외에도 머신러닝 기반 이상 탐지 (예: KNN, Isolation Forest, LSTM) 등으로 성능 개선 가능
- 실제 공격 시뮬레이션(GPS 스푸핑 장치, 재밍 장치) 환경에서 로그 수집 후, 해당 모델의 검증 수행 예정
- TinyML 적용해 드론 내부 MCU에서 실시간 분석 가능성 연구 (메모리, 연산량 제한 고려)
---
## 7. 참고 자료 
- PX4 공식 문서
- PX4/pyulog 깃허브
- QGroundControl
- Flight Review
---
## 8. 라이선스
- 본 프로젝트는 MIT License를 따릅니다.
  연구·학술 목적으로 자유롭게 활용 가능하며, 인용 시 본 리포지토리 링크를 표기해 주세요

### 9. 문의
- 연구자: JM
- 이메일: jkl3496@khu.ac.kr
---

## 5. 코드 예시: GPS 재밍 탐지 스크립트

```bash
import pandas as pd

# 재밍 지표 임계값 설정
JAMMING_THRESHOLD = 5
NOISE_THRESHOLD = 110.0

df_gps = pd.read_csv('vehicle_gps_position_0.csv')

# 조건: jamming_indicator > 5 또는 noise_per_ms > 110
suspect = df_gps[
    (df_gps['jamming_indicator'] > JAMMING_THRESHOLD) |
    (df_gps['noise_per_ms'] > NOISE_THRESHOLD)
]

if not suspect.empty:
    print("Potential GPS jamming detected. Rows:")
    print(suspect[['timestamp','jamming_indicator','noise_per_ms','satellites_used']])
else:
    print("No GPS jamming indication found.")



```bash
pip install pyulog
ulog2csv example.ulg


