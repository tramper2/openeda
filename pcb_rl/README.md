# STM32 크리스탈 레이아웃 및 배선 강화학습 최적화 방안

## 1. 문제 개요

### 1.1 목표

강화학습(PPO 알고리즘)을 사용하여 STM32 최소 시스템 보드의 크리스탈(Y1) 및 관련 콘덴서(C1, C2)의 배치와 배선을 최적화하고 다음 요구사항을 충족합니다:

- 배선이 겹치지 않음
- 부품을 덮지 않음
- 배선 방향이 PCB 규격(45°/90° 꺾임)을 충족함
- 최소 간격 0.2mm
- 크리스탈 바로 하단 영역에는 다른 배선/동박을 배치하지 않음

### 1.2 대상 네트워크

```
OSC_IN:  U1(Pin5) ↔ Y1(Pin1) ↔ C1(Pin1)
OSC_OUT: U1(Pin6) ↔ Y1(Pin3) ↔ C2(Pin2)
```

### 1.3 성능 지표

| 지표 | 현재 방안 | RL 최적화 목표 |
|------|---------|------------|
| 배선 길이 | ~15mm | <12mm |
| 배선 대칭성 | 편차 있음 | 편차 0.5mm 미만 |
| 비아 개수 | 2개 | 0-1개 |
| DRC 통과율 | 베이스라인 | 100% |

---

## 2. 점유 그리드 맵 설계

### 2.1 맵 사양

```
보드 크기: 60mm × 50mm
해상도: 0.1mm / 그리드 셀
그리드 수: 600 × 500
```

### 2.2 그리드 채널 설계

점유 맵은 **3채널** 구조로 설계되었습니다:

| 채널 | 명칭 | 설명 | 값 범위 |
|------|------|------|--------|
| 채널 0 | `OCCUPIED_BY_COMP` | 부품 점유 영역 | 0=비어 있음, 1=부품 점유 |
| 채널 1 | `OCCUPIED_BY_TRACK` | 배선 점유 영역 | 0=비어 있음, 1=배선 점유 |
| 채널 2 | `DRC_FORBIDDEN` | DRC 제한 영역 | 0=배선 가능, 1=제한됨 |

### 2.3 점유 마스크 생성

```python
# 부품 점유 영역 (채널 0)
# 각 부품의 크기에 따라 점유 그리드를 계산하고 0.2mm 여유 공간을 확장합니다.
grid[0, x_min:x_max, y_min:y_max] = 1

# DRC 제한 영역 (채널 2)
# - 크리스탈 바로 아래 영역
# - 보드 외곽선 기준 1mm 범위 내
# - 고속 신호 격리 영역
grid[2, :, :] = 1  # 제한 영역 설정
```

### 2.4 간격 제약 구현

```python
# 배선과 부품 간 최소 간격: 0.2mm = 2 그리드 셀
# 부품 점유 영역 외곽으로 2 그리드 셀을 확장하여 제한 영역으로 지정합니다.
component_extended = expand_mask(grid[0], padding=2)
grid[2] |= component_extended

# 배선 간 최소 간격: 0.2mm = 2 그리드 셀
# 배선을 배치할 때마다 배선 마스크를 확장합니다.
track_extended = expand_mask(grid[1], padding=2)
```

---

## 3. 상태 공간 설계

### 3.1 상태 벡터

```
State = {
    "grid": (3, 600, 500),       # 3채널 점유 맵
    "net_mask": (600, 500),     # 현재 네트워크 경로 마스크
    "current_pos": (2,),        # 현재 배선 헤드 위치 (x, y)
    "target_pos": (2,),         # 목표 패드 위치 (x, y)
    "features": (N, 6)          # 부품 특징
}
```

### 3.2 부품 특징

```python
Feature = [x, y, rotation, type, pin_count, net_id]

# type 인코딩:
# 0 = MCU (U1)
# 1 = 크리스탈 (Y1)
# 2 = 콘덴서 (C1, C2)
# 3 = 기타
```

### 3.3 상태 정규화

```python
# 위치를 [0, 1] 범위로 정규화
x_norm = x / BOARD_WIDTH   # 60mm
y_norm = y / BOARD_HEIGHT  # 50mm

# 점유 맵 값을 [0, 1] 범위로 정규화
grid_norm = grid.astype(np.float32) / 1.0
```

---

## 4. 동작 공간 설계

### 4.1 계층적 동작 전략

**2단계** 동작 설계 적용:

```
1단계: 부품 배치
  └── 동작: (Y1, C1, C2)의 위치 및 회전 선택

2단계: 배선 계획
  └── 동작: 배선 방향 시퀀스
```

### 4.2 배선 방향 (PCB 규격 충족)

```
방향 집합: 8방향 + STOP
├── N: 북 (0, +1)
├── S: 남 (0, -1)  
├── E: 동 (+1, 0)
├── W: 서 (-1, 0)
├── NE: 북동 (+1, +1)
├── NW: 북서 (-1, +1)
├── SE: 남동 (+1, -1)
├── SW: 남서 (-1, -1)
└── STOP: 정지 (현재 넷 배선 완료)
```

### 4.3 동작 인코딩

```python
# 이산 동작 인코딩 (0-8)
ACTION_MAP = {
    0: (0, 1),    # N
    1: (0, -1),   # S
    2: (1, 0),    # E
    3: (-1, 0),   # W
    4: (1, 1),    # NE
    5: (-1, 1),   # NW
    6: (1, -1),   # SE
    7: (-1, -1),  # SW
    8: (0, 0),    # STOP
}

# 스텝당 이동 거리: 1 그리드 셀 = 0.1mm
STEP_SIZE = 1
```

---

## 5. 보상 함수 설계

### 5.1 보상 구성

```
R_total = R_completion + α*R_drc + β*R_length + γ*R_via + δ*R_overlap + ε*R_angle + ζ*R_symmetry
```

### 5.2 각 보상 항목

| 보상 항목 | 값 | 설명 |
|--------|-----|------|
| `R_completion` | +100 | 모든 넷 연결 완료 |
| `R_drc` | -50 | DRC 제한 영역 진입 |
| `R_overlap` | -100 | 배선 중첩 (하드 제약 조건) |
| `R_length` | -0.1 × 길이(mm) | 배선 길이 감점 |
| `R_via` | -10 | 비아 개수당 감점 |
| `R_angle` | -20 | 45°/90° 이외의 꺾임 |
| `R_symmetry` | -0.5 × 길이 차이 | OSC_IN 및 OSC_OUT 길이 차이 |
| `R_step` | -0.01 | 스텝당 탐색 마이너 패널티 |

### 5.3 보상 코드 구현

```python
def compute_reward(self, action, state, next_state, done):
    reward = 0.0
    
    # 1. 연결 완료 보상
    if done and self.all_nets_connected():
        reward += 100.0
    
    # 2. DRC 위반 패널티
    if self.check_drc_violation(next_state):
        reward -= 50.0
    
    # 3. 중첩 패널티 (하드 제약 조건)
    if self.check_overlap(next_state):
        reward -= 100.0
        return reward  # 하드 제약 조건이므로 즉시 반환
    
    # 4. 길이 패널티
    track_length = self.compute_track_length(next_state)
    reward -= 0.1 * track_length
    
    # 5. 비아 패널티
    via_count = self.count_vias(next_state)
    reward -= 10.0 * via_count
    
    # 6. 꺾임 각도 패널티
    if not self.is_valid_angle(action):
        reward -= 20.0
    
    # 7. 대칭성 패널티
    length_in = self.get_net_length("OSC_IN")
    length_out = self.get_net_length("OSC_OUT")
    reward -= 0.5 * abs(length_in - length_out)
    
    return reward
```

### 5.4 가중치 매개변수

```python
WEIGHTS = {
    'alpha': 1.0,   # DRC 가중치
    'beta': 0.1,    # 길이 가중치
    'gamma': 2.0,   # 비아 가중치
    'delta': 1.0,   # 중첩 가중치
    'epsilon': 1.0,  # 각도 가중치
    'zeta': 1.0,    # 대칭성 가중치
}
```

---

## 6. 충돌 검사 프로세스

### 6.1 검사 단계

```
매 동작 실행 후:

1단계: 신규 배선이 점유하는 그리드 셀 계산
        new_cells = compute_path_cells(current_pos, action)

2단계: 부품 점유 충돌 검사
        if grid[0, new_cells] == 1:
            collision = True

3단계: 배선 중첩 검사
        if grid[1, new_cells] == 1:
            overlap = True

4단계: DRC 제한 영역 검사
        if grid[2, new_cells] == 1:
            drc_violation = True

5단계: 꺾임 각도 검사
        if not is_45_or_90_degree(prev_direction, current_direction):
            angle_violation = True

6단계: 배선 간격 검사
        if min_distance_to_track(new_cells, grid[1]) < 2:
            spacing_violation = True
```

### 6.2 간격 계산

```python
def check_min_spacing(self, new_cells, min_distance=2):
    """기존 배선과의 최소 간격 검사"""
    # 기존 배선 마스크 확장
    track_mask = expand_mask(self.grid[1], padding=min_distance)
    
    # 신규 배선이 확장 영역에 진입하는지 확인
    for cell in new_cells:
        if track_mask[cell[0], cell[1]] == 1:
            return False  # 간격 부족
    
    return True
```

---

## 7. 환경 인터페이스 정의

### 7.1 Gymnasium 인터페이스

```python
import gymnasium as gym
from gymnasium import spaces
import numpy as np

class CrystalRLEnv(gym.Env):
    metadata = {'render_modes': ['human']}
    
    def __init__(self, config=None):
        super().__init__()
        
        # 설정 매핑
        self.board_width = 60.0   # mm
        self.board_height = 50.0  # mm
        self.resolution = 0.1     # mm/그리드 셀
        self.grid_width = int(self.board_width / self.resolution)
        self.grid_height = int(self.board_height / self.resolution)
        
        # 동작 공간: 9개의 이산 동작 (8방향 + STOP)
        self.action_space = spaces.Discrete(9)
        
        # 관찰 공간
        self.observation_space = spaces.Dict({
            'grid': spaces.Box(
                low=0, high=1,
                shape=(3, self.grid_height, self.grid_width),
                dtype=np.float32
            ),
            'net_mask': spaces.Box(
                low=0, high=1,
                shape=(self.grid_height, self.grid_width),
                dtype=np.float32
            ),
            'current_pos': spaces.Box(
                low=0, high=1,
                shape=(2,),
                dtype=np.float32
            ),
            'target_pos': spaces.Box(
                low=0, high=1,
                shape=(2,),
                dtype=np.float32
            ),
        })
        
        # 초기화
        self.grid = None
        self.current_net = None
        self.step_count = 0
        self.max_steps = 1000
        
    def reset(self, seed=None, options=None):
        """환경 리셋"""
        super().reset(seed=seed)
        
        # 점유 맵 초기화
        self.grid = np.zeros((3, self.grid_height, self.grid_width), dtype=np.float32)
        
        # MCU 배치 (고정 위치)
        self._place_mcu()
        
        # 목표 넷 초기화
        self.current_net = "OSC_IN"
        self.target_pos = self._get_target_position()
        
        # 스텝 수 리셋
        self.step_count = 0
        
        return self._get_obs(), {}
    
    def step(self, action):
        """동작 실행"""
        self.step_count += 1
        
        # 동작 해석
        if action == 8:  # STOP
            done = True
            reward = 0
        else:
            # 배선 이동
            dx, dy = ACTION_MAP[action]
            new_pos = (self.current_pos[0] + dx, self.current_pos[1] + dy)
            
            # 충돌 검사
            if self._check_collision(new_pos):
                reward = -100  # 중첩 패널티 (하드 제약 조건)
                done = True
            elif self._check_drc_violation(new_pos):
                reward = -50   # DRC 위반
                done = True
            elif self._reached_target(new_pos):
                reward = 50    # 목표 도달
                done = True
            else:
                # 정상 이동
                self._update_grid(new_pos)
                reward = -0.01  # 탐색 패널티
        
        # 타임아웃 여부 확인
        if self.step_count >= self.max_steps:
            done = True
        
        obs = self._get_obs()
        info = {'step': self.step_count}
        
        return obs, reward, done, False, info
    
    def _check_collision(self, pos):
        """충돌 검사"""
        x, y = int(pos[0]), int(pos[1])
        
        # 경계 검사
        if x < 0 or x >= self.grid_width or y < 0 or y >= self.grid_height:
            return True
        
        # 부품 점유 검사
        if self.grid[0, y, x] == 1:
            return True
        
        # 배선 중첩 검사
        if self.grid[1, y, x] == 1:
            return True
        
        return False
    
    def _check_drc_violation(self, pos):
        """DRC 위반 검사"""
        x, y = int(pos[0]), int(pos[1])
        return self.grid[2, y, x] == 1
    
    def _get_obs(self):
        """관찰 정보 획득"""
        return {
            'grid': self.grid,
            'net_mask': self.net_mask,
            'current_pos': np.array(self.current_pos) / [self.board_width, self.board_height],
            'target_pos': np.array(self.target_pos) / [self.board_width, self.board_height],
        }
```

---

## 8. PPO 에이전트 설계

### 8.1 네트워크 아키텍처

```
┌─────────────────────────────────────────┐
│           관찰 입력 (Observation)          │
│                                         │
├─────────────────┬───────────────────────┤
│   Grid (3×600×500)  │  Positions (4,)    │
└────────┬────────┴──────────┬────────────┘
         │                   │
         ▼                   ▼
┌─────────────────┐   ┌───────────────┐
│  CNN Encoder    │   │  MLP Encoder  │
│  (3 → 256)      │   │  (4 → 64)     │
└────────┬────────┘   └───────┬───────┘
         │                   │
         └────────┬──────────┘
                  ▼
         ┌─────────────────┐
         │  Concatenate    │
         │  (320,)         │
         └────────┬────────┘
                  ▼
         ┌─────────────────┐
         │   MLP Block     │
         │  (320 → 256)   │
         └────────┬────────┘
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
┌─────────────┐      ┌─────────────┐
│ Policy Head │      │ Value Head  │
│ (256 → 9)   │      │ (256 → 1)   │
└─────────────┘      └─────────────┘
```

### 8.2 PPO 구현

```python
import torch
import torch.nn as nn
from torch.distributions import Categorical
from stable_baselines3 import PPO

class PPOCrystalAgent:
    def __init__(self, env_config):
        # Stable-Baselines3의 PPO 사용
        self.model = PPO(
            "MultiInputPolicy",
            env=env_config,
            learning_rate=3e-4,
            n_steps=2048,
            batch_size=64,
            n_epochs=10,
            gamma=0.99,
            gae_lambda=0.95,
            clip_range=0.2,
            ent_coef=0.01,
            verbose=1,
            tensorboard_log="./logs/",
            device="cuda"  # 혹은 "cpu"
        )
    
    def train(self, total_timesteps=100000):
        """모델 학습"""
        self.model.learn(
            total_timesteps=total_timesteps,
            callback=TensorboardCallback(),
            progress_bar=True
        )
    
    def save(self, path):
        """모델 저장"""
        self.model.save(path)
    
    def load(self, path):
        """모델 로드"""
        self.model = PPO.load(path)
    
    def predict(self, obs, deterministic=True):
        """동작 예측"""
        action, _ = self.model.predict(obs, deterministic=deterministic)
        return action
```

---

## 9. 학습 설정

### 9.1 초매개변수

```python
TRAINING_CONFIG = {
    # PPO 초매개변수
    'learning_rate': 3e-4,
    'n_steps': 2048,
    'batch_size': 64,
    'n_epochs': 10,
    'gamma': 0.99,           # 할인 요인 (Gamma)
    'gae_lambda': 0.95,      # GAE 매개변수
    'clip_range': 0.2,       # PPO 클립 범위
    'ent_coef': 0.01,        # 엔트로피 계수 (탐색)
    
    # 학습 매개변수
    'total_timesteps': 100000,
    'eval_freq': 5000,
    'save_freq': 10000,
    
    # 환경 매개변수
    'max_steps': 1000,       # 에피소드당 최대 스텝 수
    'n_envs': 4,             # 병렬 환경 수
}
```

### 9.2 보상 정규화

```python
# RunningMeanStd를 사용하여 보상 정규화
class NormalizedReward(gym.Wrapper):
    def __init__(self, env, gamma=0.99):
        super().__init__(env)
        self.gamma = gamma
        self.returns = 0
        self.count = 0
    
    def step(self, action):
        obs, reward, done, truncated, info = self.env.step(action)
        
        self.returns = reward + self.gamma * self.returns
        self.count += 1
        
        # 정규화
        if self.count > 100:
            mean = self.returns / self.count
            reward = reward - mean
        
        return obs, reward, done, truncated, info
```

---

## 10. 구현 단계

### 10.1 파일 구조

```
pcb_rl/
├── crystal_env.py          # 크리스탈 RL 환경
├── ppo_agent.py           # PPO 에이전트 구현
├── train.py               # 학습 스크립트
├── evaluate.py           # 평가 스크립트
├── config.py              # 설정 파일
├── utils/
│   ├── grid_utils.py      # 그리드 유틸리티
│   ├── drc_checker.py     # DRC 검사기
│   └── __init__.py
├── models/                # 저장된 모델
│   └── crystal_ppo.zip
└── logs/                  # TensorBoard 로그
```

### 10.2 구현 순서

```
1단계: 환경 구현 (crystal_env.py)
  - 점유 그리드 맵 초기화
  - reset() 및 step() 구현
  - 충돌 검사 구현
  - 보상 계산 구현

2단계: 에이전트 구현 (ppo_agent.py)
  - 네트워크 아키텍처 정의
  - PPO 학습 루프 구현

3단계: 학습 (train.py)
  - 초매개변수 설정
  - 학습 시작
  - 수렴 모니터링

4단계: 평가 (evaluate.py)
  - 최적 모델 로드
  - 레이아웃 생성
  - 결과 시각화

5단계: 통합 (integration)
  - RL 결과를 기존 PCB 스크립트에 적용
  - 최종 PCB 파일 생성
```

---

## 11. 예상 결과

### 11.1 학습 수렴

| 단계 | 에피소드 | 평균 보상 | 성공률 |
|------|---------|---------|-------|
| 초기 | 0-1000 | -50 | 10% |
| 중기 | 1000-5000 | +20 | 50% |
| 후기 | 5000+ | +80 | 90% |

### 11.2 최적화 효과

| 지표 | 최적화 전 | 최적화 후 |
|------|-------|-------|
| 총 배선 길이 | ~15mm | <12mm |
| 길이 대칭 오차 | >1mm | <0.5mm |
| 비아 개수 | 2개 | 0-1개 |
| DRC 통과율 | ~80% | 100% |

---

## 12. 부록

### 12.1 의존성 라이브러리

```bash
pip install gymnasium
pip install stable-baselines3
pip install torch torchvision
pip install numpy
pip install matplotlib
```

### 12.2 하드웨어 요구사항

```
- GPU: NVIDIA GTX 1060+ (권장)
- RAM: 8GB+
- 학습 시간: ~2-4 시간
```

### 12.3 참고 자료

- [Stable-Baselines3 문서](https://stable-baselines3.readthedocs.io/)
- [PPO 논문](https://arxiv.org/abs/1707.06347)
- [Gymnasium 문서](https://gymnasium.farama.org/)

---

*문서 버전: 1.0*  
*마지막 업데이트: 2026-03-05*
