---
description: "Figma 페이지/섹션 내 모든 화면을 자동 탐색하여 프로젝트 Composable과 매칭 후 일괄 디자인 QA를 수행합니다."
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Agent
  - mcp__figma-desktop__get_design_context
  - mcp__figma-desktop__get_screenshot
  - mcp__figma-desktop__get_variable_defs
---

# 일괄 디자인 QA (Figma 기준)

`$ARGUMENTS`를 파싱하여 Figma 페이지/섹션 내 모든 화면 프레임을 탐색하고,
프로젝트의 Composable과 자동 매칭하여 `design-qa-agent`에 화면별로 위임합니다.

## Usage

```
/design-qa-all
Figma: <Figma 페이지 또는 섹션 URL>
모듈: <모듈 경로>          # 선택, 기본: app. 쉼표로 복수 지정 가능
범위: all                  # 선택, 기본: all
디바이스: PIXEL_5           # 선택
```

예시 (페이지 전체):
```
/design-qa-all
Figma: https://www.figma.com/design/ABC123/MyApp?node-id=0-1
모듈: feature/signup, feature/home, feature/gallery
```

예시 (특정 섹션만):
```
/design-qa-all
Figma: https://www.figma.com/design/ABC123/MyApp?node-id=100-200
모듈: feature/signup
```

---

## Steps

### Step 0. 인자 파싱

`$ARGUMENTS`에서 다음을 추출합니다:

1. `Figma:` -> URL에서 `node-id` 추출 (`-`를 `:`로 변환)
2. `모듈:` -> `module_paths` 배열 (쉼표 구분, 기본: `["app"]`)
3. `범위:` -> `qa_scope` (기본: `all`)
4. `디바이스:` -> `device_config` (기본: `PIXEL_5`)

Figma URL이 없으면 즉시 중단하고 사용법을 안내합니다.

### Step 1. 환경 자동 감지

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
```

### Step 2. Figma 구조 탐색 — 화면 프레임 목록 추출

Figma MCP로 지정된 노드의 하위 구조를 탐색합니다:

```
mcp__figma-desktop__get_design_context(
  nodeId: "<node_id>",
  clientLanguages: "kotlin",
  clientFrameworks: "jetpack-compose"
)
```

반환된 구조에서 **화면 프레임**을 추출합니다:

```
추출 기준:
  - 최상위 자식 Frame → 섹션(Feature)으로 간주
  - 섹션 내 자식 Frame → 개별 화면으로 간주
  - Frame 이름에서 화면명과 상태를 파싱:
    예: "회원가입 - 기본 상태" → screen_name: "회원가입", label: "기본 상태"
    예: "SignUp / Default"    → screen_name: "SignUp", label: "Default"
    예: "로그인"              → screen_name: "로그인", label: "기본"
```

**이름 파싱 규칙**:
- ` - `, ` / `, ` | ` 구분자로 화면명과 상태를 분리
- 구분자 없으면 Frame 이름 전체가 화면명, label은 "기본"
- 동일 화면명의 프레임들을 그룹화하여 상태별 목록으로 구성

**노드가 너무 크거나 Section인 경우**:
sparse metadata가 반환되면 하위 Frame ID들을 추출하여 개별 호출합니다.
재호출 최대 깊이: 2

### Step 3. 화면 그룹 구성

추출된 프레임들을 화면 단위로 그룹화합니다:

```
SCREEN_GROUPS = [
  {
    screen_name: "회원가입",
    figma_nodes: [
      { node_id: "700:11696", label: "기본 상태" },
      { node_id: "700:11547", label: "에러 상태" },
      { node_id: "700:11800", label: "로딩 상태" },
    ]
  },
  {
    screen_name: "로그인",
    figma_nodes: [
      { node_id: "800:1234", label: "기본" },
      { node_id: "800:1235", label: "에러" },
    ]
  },
  ...
]
```

### Step 4. 프로젝트 Composable 매칭

각 `module_path`에서 화면 Composable을 탐색하고 SCREEN_GROUPS와 매칭합니다:

```
각 module_path에 대해:
  Glob: <PROJECT_ROOT>/<module_path>/src/main/**/*Screen*.kt
  Glob: <PROJECT_ROOT>/<module_path>/src/main/**/*Composable*.kt

추출된 Composable 함수명과 SCREEN_GROUPS의 screen_name을 매칭:
  - 정확 일치: "SignUpScreen" ↔ "SignUp" (Screen 접미사 제거 후 비교)
  - 유사 일치: "회원가입" ↔ "SignUp" (한영 매핑은 수동 확인 필요)
  - 미매칭: Figma에는 있지만 Composable을 찾지 못한 화면
```

### Step 5. 매칭 테이블 확인

사용자에게 매칭 결과를 보여주고 확인/수정을 요청합니다:

```
## Figma ↔ Composable 매칭 결과

총 화면: N개 / 매칭: M개 / 미매칭: K개

| # | Figma 화면명 | 상태 수 | Composable | 모듈 | 매칭 |
|---|------------|--------|-----------|------|------|
| 1 | 회원가입     | 3      | SignUpScreen | feature/signup | O 자동 |
| 2 | 로그인       | 2      | LoginScreen  | feature/auth   | O 자동 |
| 3 | 갤러리       | 4      | GalleryScreen | feature/gallery | O 자동 |
| 4 | 설정         | 2      | ???          | ???            | X 미매칭 |
| 5 | 온보딩       | 5      | OnboardingScreen | app       | O 자동 |

미매칭 화면에 대해:
  - Composable 경로를 직접 입력하시겠습니까?
  - 또는 미매칭 화면은 건너뛸까요?

수정할 매칭이 있으면 번호를 알려주세요. (없으면 Enter)
```

사용자가 확인하면 최종 `QA_TARGETS` 목록을 확정합니다.

### Step 6. 일괄 QA 실행

확정된 QA_TARGETS를 순차적으로 `design-qa-agent`에 위임합니다:

```
각 target에 대해:

  Agent(design-qa-agent):
    screen_name: <화면명>
    figma_nodes:
      - node_id: "<node_id>"
        label: "<label>"
      - ...
    project_root: <PROJECT_ROOT>
    module_path: <매칭된 모듈>
    qa_scope: <범위>
    device_config: <디바이스>
```

**진행 상황 표시**:
```
[1/N] 회원가입 ... 실행 중
[1/N] 회원가입 ... 완료 (Critical: 2, Minor: 1, Pass: 15)
[2/N] 로그인 ... 실행 중
...
```

**에러 처리**:
- 특정 화면 QA 실패 시 해당 화면을 Skip하고 다음으로 진행
- 실패 사유를 최종 보고서에 기록

### Step 7. 통합 보고서 생성

모든 화면의 QA 결과를 통합한 요약 보고서를 생성합니다:

파일 경로: `<PROJECT_ROOT>/docs/design-qa/SUMMARY.md`

```markdown
# 디자인 QA 통합 보고서

- **작성일**: YYYY-MM-DD
- **Figma 소스**: <Figma URL>
- **검증 화면**: N개 / 스킵: K개
- **캡처 방식**: Paparazzi (DeviceConfig.<DEVICE_CONFIG>)

---

## 전체 요약

| 등급 | 건수 |
|------|------|
| Critical | N건 |
| Minor | N건 |
| Pass | N건 |

---

## 화면별 결과

| # | 화면명 | 모듈 | Critical | Minor | Pass | 상태 |
|---|--------|------|----------|-------|------|------|
| 1 | 회원가입 | feature/signup | 2 | 1 | 15 | 완료 |
| 2 | 로그인 | feature/auth | 0 | 0 | 12 | 완료 |
| 3 | 갤러리 | feature/gallery | 1 | 3 | 20 | 완료 |
| 4 | 설정 | — | — | — | — | 미매칭 (스킵) |

---

## Critical 이슈 전체 목록

| # | 화면 | 이슈 | method | 추정 위치 |
|---|------|------|--------|---------|
| 1 | 회원가입 | 버튼 색상 불일치 (dE=8.7) | quantitative | SignUpScreen.kt:45 |
| 2 | 회원가입 | 좌우 패딩 24dp (Figma: 16dp) | quantitative | SignUpScreen.kt:32 |
| 3 | 갤러리 | 그리드 간격 4dp (Figma: 8dp) | quantitative | GalleryScreen.kt:78 |

---

## 미매칭 화면 (Figma에 있지만 Composable 미발견)

| Figma 화면명 | 상태 수 | 비고 |
|-------------|--------|------|
| 설정 | 2 | Composable 탐색 실패 |

---

## 상세 보고서 링크

- [회원가입](./signup.md)
- [로그인](./login.md)
- [갤러리](./gallery.md)
```

### Step 8. 결과 반환

```
## 일괄 디자인 QA 완료

통합 보고서: docs/design-qa/SUMMARY.md

전체 결과:
  검증 화면: N개
  Critical: X건 (Y개 화면)
  Minor: X건
  Pass: X건
  스킵: K개 화면

Critical 화면 Top 3:
  1. 회원가입 — Critical 2건
  2. 갤러리 — Critical 1건
  3. ...

수정을 진행할까요?
```
