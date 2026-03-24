# claude-android-qa-plugin

Android 앱 개발 워크플로우를 위한 Claude Code 플러그인입니다.

Figma 디자인 명세서 검증, Android Studio Journeys 기반 UI 테스트, Gherkin 스펙 작성을 자동화합니다.

---

## 포함된 도구

### Agents

| 에이전트 | 설명 |
|---------|------|
| `design-qa-agent` | Figma MCP + ADB 스크린샷으로 디자인 명세와 실제 UI를 픽셀 단위로 비교 |
| `ui-test-agent` | `.feature` 파일 기반 Android Studio Journeys XML 생성 및 UI 테스트 실행 |

### Skills

| 스킬 | 트리거 | 설명 |
|------|--------|------|
| `spec-writer` | `/spec-writer`, `스펙 작성해줘` | Figma 또는 화면 설명에서 Gherkin `.feature` 파일과 ViewModel 단위 테스트 작성 |

---

## 설치

### 1. 마켓플레이스 등록

```
/plugin marketplace add https://github.com/Wonjong-Jeong/claude-android-qa-plugin
```

### 2. 플러그인 설치

```
/plugin install claude-android-qa-plugin@claude-android-qa-plugin
```

### 또는 직접 설치 (CLI)

```bash
claude plugin install https://github.com/Wonjong-Jeong/claude-android-qa-plugin
```

---

## 사용 방법

### design-qa-agent

Figma 디자인 명세와 실제 앱 화면을 비교합니다.

**필요 환경**:
- Figma Desktop + Figma MCP 서버 설정
- ADB 연결된 Android 기기 또는 에뮬레이터

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

**출력**: `docs/design-qa/<screen_name>.md` 보고서 (Critical/Minor/Pass 이슈 목록)

---

### ui-test-agent

`.feature` 파일(Gherkin)에서 Android Studio Journeys XML을 생성하고 테스트를 실행합니다.

**필요 환경**:
- AGP 9.0.0+ (Android Studio Journeys 지원)
- Gemini 인증 완료
- ADB 연결된 기기

**입력 예시**:
```
screen_name: PostDetail
app_package: com.example.android
project_root: /path/to/project
module_path: feature/post
```

---

### spec-writer

Figma 또는 화면 설명에서 비즈니스 로직 명세를 작성합니다.

```
/spec-writer
스펙 작성해줘
feature 파일 써줘
```

**출력**:
- `<module_path>/src/journeysTest/specs/<ScreenName>.feature` — Gherkin 명세 (ui-test-agent 입력용)
- `<module_path>/src/test/java/.../<ScreenName>ViewModelTest.kt` — ViewModel 단위 테스트

---

## Figma MCP 설정

`design-qa-agent`와 `spec-writer`의 Figma 연동을 위해 Figma Desktop에서 MCP 서버를 활성화해야 합니다.

Figma Desktop → 설정 → Enable Developer Tools → MCP Server 활성화

---

## 워크플로우

```
spec-writer          →   .feature 파일 생성
    ↓
ui-test-agent        →   Journey XML 생성 + 테스트 실행
    ↓
design-qa-agent      →   디자인 명세 vs 실제 UI 비교 보고서
```

---

## 요구 사항

- Claude Code (최신 버전)
- Android Studio (Journeys 사용 시 AGP 9.0.0+)
- ADB (디바이스 연결 시)
- Figma Desktop + MCP 서버 (Figma 연동 시)
- Python 3 + Pillow (`pip install pillow`) — 이미지 비교 기능
- ImageMagick (선택, 더 정밀한 이미지 비교)
