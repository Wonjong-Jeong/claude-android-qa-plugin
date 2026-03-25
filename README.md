# claude-android-qa-plugin

Android 앱 개발 워크플로우를 위한 Claude Code 플러그인입니다.

Figma 스토리보드 기반 분기 탐색, Gherkin 스펙 작성, Maestro UI 테스트 자동화, 디자인 QA를 파이프라인으로 연결합니다.

---

## 설치

### 방법 1 — Claude Code CLI

```bash
claude plugin install https://github.com/Wonjong-Jeong/claude-android-qa-plugin
```

### 방법 2 — 마켓플레이스 등록 후 설치

**① 마켓플레이스 등록** (최초 1회):

Claude Code 세션 내에서 실행합니다:

```
/plugin marketplace add https://github.com/Wonjong-Jeong/claude-android-qa-plugin
```

**② 플러그인 설치**:

```
/plugin install claude-android-qa-plugin@claude-android-qa-plugin
```

### 방법 3 — settings.json으로 자동 활성화

`~/.claude/settings.json`에 추가하면 모든 세션에서 자동으로 로드됩니다:

```json
{
  "extraKnownMarketplaces": {
    "claude-android-qa-plugin": {
      "source": {
        "source": "github",
        "repo": "Wonjong-Jeong/claude-android-qa-plugin"
      }
    }
  },
  "enabledPlugins": {
    "claude-android-qa-plugin@claude-android-qa-plugin": true
  }
}
```

### 방법 4 — 로컬 클론

```bash
git clone https://github.com/Wonjong-Jeong/claude-android-qa-plugin.git
claude --plugin-dir ./claude-android-qa-plugin
```

---

## 포함된 도구

### Agents

| 에이전트 | 설명 |
|---------|------|
| `flow-explorer-agent` | Figma 스토리보드를 읽어 화면 순서·분기 트리를 자동 추출 |
| `spec-writer` *(skill)* | 화면 설명 또는 Figma에서 Gherkin `.feature` 파일과 ViewModel 단위 테스트 작성 |
| `ui-test-agent` | `.feature` 파일 기반 Maestro Flow YAML 생성 및 UI 테스트 실행 |
| `qa-orchestrator-agent` | 프로젝트 전체 `.feature` 파일을 일괄 실행하고 결과를 집계 |
| `coverage-report-agent` | 분기 트리 대비 `.feature` 작성 커버리지 분석 및 누락 경로 리포트 |
| `design-qa-agent` | Figma MCP + ADB 스크린샷으로 디자인 명세와 실제 UI를 픽셀 단위로 비교 |

---

## 전체 파이프라인

```
Figma 스토리보드
      │
      ▼
[flow-explorer-agent]
  • 섹션별 Feature 경계 파악
  • 좌→우 화면 순서 추출
  • 분리된 그룹을 분기로 식별
  • 화면 내 UI 요소로 트리거 추론
  • 출력: 분기 트리 + Scenario 목록 (docs/flow-explorer/)
      │
      ▼
[spec-writer]
  • 분기 트리를 입력으로 Scenario 작성
  • 단일 화면: 인터랙션별 Gherkin Scenario
  • 멀티 화면: @flow-ref 태그로 화면 간 의존성 선언
  • 출력: src/uiTest/specs/<ScreenName>.feature
         src/test/java/.../<ScreenName>ViewModelTest.kt
      │
      ▼
[coverage-report-agent]
  • 분기 트리 vs 실제 .feature 파일 비교
  • Feature / Scenario / YAML 생성 커버리지 계산
  • 누락 Scenario 목록 + 다음 액션 안내
  • 출력: docs/qa-report/coverage.md
      │
      │  (누락 Scenario → spec-writer로 보완 후 재실행)
      ▼
[qa-orchestrator-agent]
  • 전체 .feature 파일 순회
  • @flow-ref 위상 정렬로 실행 순서 결정
  • Maestro YAML 일괄 생성 + 실행
  • PASS/FAIL 집계 보고서 생성
  • 출력: docs/qa-report/aggregate.md
      │
      ├─ FAIL 화면 → [ui-test-agent] 개별 실행·디버깅
      │
      └─ 디자인 검증 필요 시
            ▼
      [design-qa-agent]
        • Figma 명세 vs 실제 앱 화면 픽셀 비교
        • 출력: docs/design-qa/<screen_name>.md
```

---

## 도구별 사용 방법

### flow-explorer-agent

Figma 스토리보드에서 분기 트리를 자동 추출합니다.

**필요 환경**: Figma Desktop + Figma MCP 서버 설정

**입력 예시**:
```
figma_storyboard:
  node_id: "1234:5678"
  label: "그룹 생성 ~ 아기 등록 스토리보드"
project_root: /path/to/project
```

**출력**: `docs/flow-explorer/<storyboard_label>.md` — 분기 트리 + Scenario 목록

---

### spec-writer

화면 설명 또는 Figma에서 비즈니스 로직 명세를 작성합니다.

```
/spec-writer
스펙 작성해줘
feature 파일 써줘
```

**출력**:
- `<module_path>/src/uiTest/specs/<ScreenName>.feature` — Gherkin 명세
- `<module_path>/src/test/java/.../<ScreenName>ViewModelTest.kt` — ViewModel 단위 테스트

**멀티 화면 flow 지원**:

진입 경로에 다른 화면이 필요한 경우 `@flow-ref` 태그로 선언합니다.

```gherkin
@flow-ref: Login, Home
Feature: 포스트 상세 화면 — 좋아요 기능

Background:
  Given Login flow를 완료한다
  And Home flow를 완료한다
  And 첫 번째 포스트 카드를 탭하여 상세 화면에 진입한다
```

---

### ui-test-agent

단일 화면의 `.feature` 파일에서 Maestro Flow YAML을 생성하고 테스트를 실행합니다.

**필요 환경**: Maestro CLI, ADB 연결된 기기

**입력 예시**:
```
screen_name: PostDetail
app_package: com.example.android
project_root: /path/to/project
module_path: feature/post
```

**출력**: `<module_path>/maestro/flows/<screen_name>/` — Maestro Flow YAML 파일들

---

### qa-orchestrator-agent

프로젝트 전체 `.feature` 파일을 일괄 실행합니다.

**입력 예시**:
```
project_root: /path/to/project
app_package: com.example.android
module_path: feature/post
run_mode: all          # all | failed_only | yaml_only
```

**출력**: `docs/qa-report/aggregate.md` — 전체 PASS/FAIL 집계 보고서

| run_mode | 동작 |
|---|---|
| `all` | 전체 `.feature` YAML 생성 + 실행 |
| `failed_only` | 이전 보고서의 FAIL 항목만 재실행 |
| `yaml_only` | YAML 생성만 (기기 없을 때 유용) |

---

### coverage-report-agent

분기 트리 대비 `.feature` 작성 현황을 분석합니다.

**입력 예시**:
```
project_root: /path/to/project
flow_tree_path: docs/flow-explorer/storyboard.md
```

**출력**: `docs/qa-report/coverage.md` — Feature / Scenario / YAML 커버리지 리포트

---

### design-qa-agent

Figma 디자인 명세와 실제 앱 화면을 비교합니다.

**필요 환경**: Figma Desktop + Figma MCP 서버, ADB 연결된 기기

**입력 예시**:
```
screen_name: 회원가입 화면
app_package: com.example.android
figma_nodes:
  - node_id: "700:11696"
    label: "기본 상태"
entry_method: "deep_link:example://signup"
project_root: /path/to/project
module_path: feature/auth
```

**출력**: `docs/design-qa/<screen_name>.md` — Critical/Minor/Pass 이슈 목록

---

## 요구 사항

| 도구 | 필요 환경 |
|---|---|
| 전체 공통 | Claude Code (최신 버전), ADB |
| `flow-explorer-agent`, `design-qa-agent`, `spec-writer` | Figma Desktop + MCP 서버 |
| `ui-test-agent`, `qa-orchestrator-agent` | Maestro CLI |
| `design-qa-agent` | Python 3 + Pillow (`pip install pillow`), ImageMagick (선택) |

---

## Figma MCP 설정

`flow-explorer-agent`, `design-qa-agent`, `spec-writer`의 Figma 연동을 위해 Figma Desktop에서 MCP 서버를 활성화해야 합니다.

Figma Desktop → 설정 → Enable Developer Tools → MCP Server 활성화
