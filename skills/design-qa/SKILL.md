---
description: "Figma 디자인 명세와 앱 구현을 Paparazzi(JVM 렌더링)로 비교하여 디자인 QA 보고서를 생성합니다. 에뮬레이터 불필요."
allowed-tools:
  - Read
  - Bash
  - Glob
  - Agent
---

# 디자인 QA (Paparazzi 기반)

`$ARGUMENTS`를 파싱하여 `design-qa-agent`에 위임합니다.
Paparazzi가 JVM에서 Composable을 직접 렌더링하므로 에뮬레이터/실기기가 필요하지 않습니다.

## Usage

```
/design-qa
화면명: <화면 이름>
Figma:
  - <URL1>  # 상태 설명 (선택)
  - <URL2>
모듈: <모듈 경로>                    # 선택, 기본: app
테스트: @<경로/파일명.kt>             # 선택 — Phase 4.5 교차 검증 활성화
범위: all | layout | typography | color | visual  # 선택, 기본: all
디바이스: PIXEL_5                     # 선택, Paparazzi DeviceConfig
```

> `진입:` 옵션은 더 이상 필요하지 않습니다. Paparazzi가 Composable을 직접 렌더링합니다.

예시 (기본):
```
/design-qa
화면명: 회원가입
Figma:
  - https://www.figma.com/design/ABC123?node-id=700-11696  # 기본 상태
  - https://www.figma.com/design/ABC123?node-id=700-11547  # 에러 상태
```

예시 (멀티모듈 + 테스트 교차검증):
```
/design-qa
화면명: 회원가입
Figma:
  - https://www.figma.com/design/ABC123?node-id=700-11696  # 기본 상태
  - https://www.figma.com/design/ABC123?node-id=700-11547  # 에러 상태
모듈: feature/signup
테스트: @feature/signup/src/test/SignUpViewModelTest.kt
```

예시 (색상만 빠르게 검증):
```
/design-qa
화면명: 홈
Figma:
  - https://www.figma.com/design/ABC123?node-id=500-300
범위: color
```

---

## Steps

### Step 0. 인자 파싱

`$ARGUMENTS`에서 다음을 추출합니다:

1. `화면명` -> `screen_name`
2. `Figma:` 블록 -> 각 URL에서 `node-id` 파라미터 추출 후 `-`를 `:`로 변환 (`700-11696` -> `700:11696`),
   인라인 `#` 주석을 `label`로 사용 (없으면 순번 `상태 N`)
3. `모듈:` -> `module_path` (기본: `app`)
4. `테스트:` 또는 `@파일` 참조 목록 -> `test_files`
5. `범위:` -> `qa_scope` (기본: `all`)
6. `디바이스:` -> `device_config` (기본: `PIXEL_5`)

Figma URL이 없으면 즉시 중단하고 사용법을 안내합니다.

### Step 1. 환경 자동 감지

**project_root 탐색:**
```bash
git rev-parse --show-toplevel 2>/dev/null || pwd
```

### Step 2. design-qa-agent에 위임

Agent 도구로 `design-qa-agent`를 호출하고 아래 구조화된 입력을 전달합니다:

```
screen_name: <화면명>
figma_nodes:
  - node_id: "<node_id>"
    label: "<label>"
  - ...
project_root: <탐지된 루트 경로>
module_path: <모듈 경로>
qa_scope: <범위>
device_config: <디바이스 설정>
test_files:
  - <test 파일 경로>
```

`test_files`가 비어 있으면 해당 키를 생략합니다.
