# Dobot E6 + SmolVLA 판단·수행 시스템: 문제점과 해결 방법

> 작성일: 2026-10-01

## 용어 정의

| 용어 | 역할 |
|---|---|
| **AI** | 판단 주체. Perception 결과와 DB 이력을 보고 어떤 물체를 집을지 결정 |
| **SmolVLA** | AI가 정한 Task를 Action Chunk로 변환하는 실행 Policy |
| **Agent** | 실제 동작 수행 주체 = **Dobot E6**. Action Chunk를 물리적으로 실행 |

## 전체 파이프라인

```
Camera → Perception → AI(판단) → SmolVLA(동작 생성) → Agent/Dobot E6(수행)
       → 결과 판정 → DB → AI 재판단
```

---

## 1. AI가 고른 물체를 SmolVLA가 구분하지 못합니다

### 문제
AI가 "Object A를 집어라"라고 지시해도, SmolVLA가 그 문장으로 물체를 구분하도록 학습되어 있지 않습니다.
지금 계획은 **물체 1개, 같은 지시문으로 50 Episode**(5 Zone × 10 Episode)를 모으는 것입니다.
이렇게 학습하면 SmolVLA는 문장을 무시하고 보이는 물체를 집게 됩니다.
그러면 Agent는 AI가 고르지 않은 물체로 갈 수 있습니다.

> 참고: SmolVLA의 언어 입력은 녹화할 때 정한 `task` 문자열입니다.
> - `src/lerobot/scripts/lerobot_record.py` → `--dataset.single_task`
> - `src/lerobot/policies/smolvla/processor_smolvla.py` → `task`를 모델 입력으로 변환

### 해결 방법
- 한 장면에 물체를 **2~4개** 놓습니다. 에피소드마다 다른 물체를 지시문으로 지정해서 녹화합니다.
  - 예: `"pick the blue box and place it in the tray"`
- 지시문에 쓰는 물체 이름은 **AI가 실제로 출력할 표현과 똑같이** 맞춥니다.
  AI가 지시문을 정해진 형식(템플릿)에 맞춰 출력하도록 하면 됩니다.
- 기본 동작용 50 Episode와 언어 조건 데이터는 **단계를 나눠서** 수집합니다.

---

## 2. 실패 원인이 섞입니다: 물체 탓인지, SmolVLA·Agent 동작 탓인지

### 문제
DB에 "실패"만 기록하면 두 경우를 구분할 수 없습니다.
- SmolVLA가 잘못된 동작을 만들어 Agent가 엉뚱한 곳으로 간 경우
- 물체가 원래 흡착되지 않는 경우

구분하지 않으면 AI가 **멀쩡한 물체를 흡착 불가로 잘못 판단**하게 됩니다.

### 해결 방법
DB에 `failure_type`을 기록합니다.

| failure_type | 의미 | 원인 | 누가 활용하나 |
|---|---|---|---|
| `reach_fail` | Agent가 물체까지 가지 못함 | SmolVLA 동작 문제 | SmolVLA 개선용 |
| `no_vacuum` | 접촉했지만 흡착 안 됨 | 물체 표면·크기 | **AI 판단용** |
| `drop_in_transit` | 이동 중 떨어뜨림 | 물체 무게·흡착 면적 | **AI 판단용** |
| `place_fail` | 목표 위치에 놓지 못함 | SmolVLA 동작 문제 | SmolVLA 개선용 |

AI의 흡착 가능성 판단에는 `no_vacuum`과 `drop_in_transit`만 반영합니다.

---

## 3. Agent의 작업 결과를 자동으로 판정할 수단이 없습니다

### 문제
"결과 → DB → AI 재판단" 루프가 사람 없이 돌려면 Agent의 성공·실패를 자동으로 알아야 합니다.
또 SmolVLA는 작업이 끝났다는 신호를 내지 않습니다.
기본 설정(`chunk_size=50`, `n_action_steps=50`)에서는 Action Chunk를 계속 생성할 뿐입니다.

### 해결 방법
- **흡착 판정**: Agent(Dobot E6)의 진공 압력 센서나 흡착 확인 I/O 신호를 읽습니다.
  흡착 성공(`no_vacuum` 여부)과 이동 중 낙하(`drop_in_transit`)를 바로 판정할 수 있습니다.
  → E6 컨트롤러로 이 신호를 읽을 수 있는지 **먼저 확인**해야 합니다.
- **놓기 판정**: 작업이 끝난 뒤 카메라로 Target Box를 촬영해 물체가 들어갔는지 확인합니다.
- **작업 종료 판정**: timeout과 위 판정 조건을 묶은 종료 로직을 직접 구현합니다.

---

## 4. Agent(Dobot E6)가 LeRobot에 연결되어 있지 않습니다

### 문제
LeRobot에는 Dobot E6 드라이버가 없습니다.
`src/lerobot/robots/`에는 so_follower, koch_follower 같은 로봇만 있습니다.

### 해결 방법
- `src/lerobot/robots/robot.py`의 `Robot` 인터페이스를 구현한 **E6 클래스를 직접 만듭니다.**
  SmolVLA와 Agent는 이 클래스를 통해 연결됩니다.
- **흡착 on/off를 action의 한 차원으로 넣을지** 결정합니다.
  SmolVLA가 흡착 타이밍까지 생성하게 할지를 정하는 문제입니다.

---

## 5. AI의 판단 방식이 정의되어 있지 않습니다

### 문제
AI가 무엇으로, 어떤 기준으로 판단하는지가 문서에 없습니다.

### 해결 방법
AI를 두 부분으로 구성합니다.

- **VLM/LLM 사전 판단**: 처음 보는 물체를 판단하고(예: "망사 표면이라 흡착이 어려울 것") 선택 이유를 설명합니다.
- **DB 통계**: 물체 종류와 특징별 흡착 성공률을 코드로 계산합니다.
  - 예: `성공률 = (성공 + 1) / (시도 + 2)`
- **최종 점수**: 시도 횟수가 적으면 VLM 판단을, 시도가 쌓이면 DB 통계를 더 믿도록 섞습니다.

추가 원칙:
- 성공률 같은 수치 계산은 LLM에 맡기지 않고 **코드로 처리한 뒤 결과만 AI에 컨텍스트로 전달**합니다.
- DB는 Object ID가 아니라 **크기·형태·표면이 비슷한 물체**를 찾는 방식으로 조회합니다.

---

## 추천 진행 순서

1. **Agent 연동**: E6 `Robot` 클래스를 만들고, 물체 1개로 SmolVLA Pick-and-Place를 안정화합니다(50 Episode).
2. **결과 판정 + DB**: 진공 센서로 자동 판정하고, `failure_type`을 포함해 기록합니다.
3. **SmolVLA 언어 조건 학습**: 여러 물체를 놓고 에피소드마다 다른 지시문으로 데이터를 수집한 뒤 재학습합니다.
4. **AI 결합**: VLM 판단과 DB 통계를 합쳐 "판단 → 실행 → 기록 → 재판단" 루프를 완성합니다.
