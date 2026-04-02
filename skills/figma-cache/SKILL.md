---
description: "Figma 명세서 데이터를 로컬에 캐싱하여 API 호출 없이 반복 활용할 수 있게 합니다. Rate limit 회피."
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - mcp__figma-desktop__get_design_context
  - mcp__figma-desktop__get_screenshot
  - mcp__figma-desktop__get_variable_defs
---

# Figma 명세서 로컬 캐시 내보내기

`$ARGUMENTS`를 파싱하여 Figma 명세서의 모든 화면 데이터를 로컬 파일로 수집합니다.
수집된 캐시는 `/design-qa-all --cache`에서 API 호출 없이 사용됩니다.

**데이터 소스 우선순위**:
```
1순위: Figma MCP (Dev Mode) — get_design_context, get_screenshot, get_variable_defs
2순위: Figma REST API — MCP rate limit 도달 시 자동 전환 또는 사용자 명시 지정
```

## Usage

```
/figma-cache
Figma: <Figma 페이지 또는 섹션 URL>
딜레이: 2              # 선택, API 호출 간 대기 초 (기본: 2)
소스: mcp              # 선택, mcp(기본) | rest-api
토큰: <TOKEN>          # rest-api 모드 시 필요 (Figma Personal Access Token)
```

이어서 수집 (rate limit으로 중단된 경우):
```
/figma-cache --resume
```

> MCP rate limit 도달 시 REST API로 자동 전환을 시도합니다 (토큰이 환경변수 `FIGMA_TOKEN`에 있는 경우).
> 또는 `소스: rest-api`로 명시하여 처음부터 REST API를 사용할 수 있습니다.

---

## Steps

### Step 0. 인자 파싱

`$ARGUMENTS`에서 다음을 추출합니다:

1. `--resume` 플래그 확인 → resume 모드
2. `Figma:` → URL에서 `node-id` 및 `file_key` 추출 (`-`를 `:`로 변환)
3. `딜레이:` → `DELAY_SECONDS` (기본: 2)
4. `소스:` → `DATA_SOURCE` (기본: `mcp`)
5. `토큰:` → `FIGMA_TOKEN` (미제공 시 환경변수 `FIGMA_TOKEN` 확인)

resume 모드가 아닌데 Figma URL이 없으면 즉시 중단하고 사용법을 안내합니다.

### Step 1. 환경 감지

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
CACHE_DIR="$PROJECT_ROOT/.figma-cache"

# REST API용 변수 (DATA_SOURCE=rest-api 또는 MCP fallback 시 사용)
FILE_KEY="<URL에서 추출한 file key>"
# 예: https://www.figma.com/design/ABC123/... → FILE_KEY="ABC123"
FIGMA_TOKEN="${FIGMA_TOKEN}"   # 토큰: 인자 또는 환경변수
```

### Step 2. Resume 모드 확인

`--resume` 플래그가 있는 경우:

```
resume-list.json 확인:
  Glob: <CACHE_DIR>/resume-list.json

  존재하면:
    → 남은 노드 목록을 읽어서 Step 5로 바로 진행
  존재하지 않으면:
    → "이어서 수집할 항목이 없습니다" 안내 후 종료
```

### Step 3. Figma 구조 탐색 — 화면 프레임 목록 추출

design-qa-all의 Step 2~3과 동일한 로직으로 화면 프레임을 추출하고 그룹화합니다.

**MCP 모드** (`DATA_SOURCE=mcp`, 기본):
```
mcp__figma-desktop__get_design_context(
  nodeId: "<root_node_id>",
  clientLanguages: "kotlin",
  clientFrameworks: "jetpack-compose"
)
```

**REST API 모드** (`DATA_SOURCE=rest-api` 또는 MCP fallback):
```bash
curl -s -H "X-Figma-Token: ${FIGMA_TOKEN}" \
  "https://api.figma.com/v1/files/${FILE_KEY}/nodes?ids=${ROOT_NODE_ID}" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
# 'nodes' → root node → 'document' → children 추출
# Frame 타입 자식을 화면 프레임으로 파싱
print(json.dumps(data, indent=2))
" > /tmp/figma_structure.json
```

> REST API는 per-minute 제한만 있고 일일 제한이 없어, 대량 수집에 적합합니다.
> MCP에서 "Rate limit exceeded" 에러 발생 시 자동으로 REST API로 전환합니다.

반환된 구조에서 화면 프레임을 추출합니다:
- 최상위 자식 Frame → 섹션(Feature)
- 섹션 내 자식 Frame → 개별 화면
- Frame 이름에서 화면명과 상태를 파싱
- 동일 화면명의 프레임들을 그룹화

**결과**: `SCREEN_GROUPS` + 전체 `NODE_LIST` (수집 대상 node_id 목록)

### Step 4. 수집 계획 표시

사용자에게 수집 계획을 보여주고 확인을 요청합니다:

```
## Figma 캐시 수집 계획

캐시 경로: <PROJECT_ROOT>/.figma-cache/

수집 대상: N개 화면 (상태 포함 M개 프레임)
예상 API 호출: M × 4 = K회 (SVG 포함, 토큰 없으면 M × 3)
  - get_design_context: M회
  - get_screenshot (PNG): M회
  - get_variable_defs: M회
  - get_image (SVG): M회 (FIGMA_TOKEN 있을 때만)
딜레이: <DELAY_SECONDS>초/호출
예상 소요: ~X분

| # | 화면명 | 상태 수 | 프레임 수 |
|---|--------|--------|---------|
| 1 | 메인화면 | 2 | 2 |
| 2 | 회원가입 | 3 | 3 |
| ... | ... | ... | ... |

진행할까요?
```

### Step 5. 순차 수집 (딜레이 삽입)

```bash
mkdir -p "$CACHE_DIR"
```

각 NODE_LIST 항목에 대해 순차적으로 수집합니다:

```
진행 표시:
[1/M] 메인화면 - 앨범 있는 상태 (700:7506) ...

각 node에 대해:
  NODE_DIR = <CACHE_DIR>/<node_id에서 :을 -로 변환>
  mkdir -p "$NODE_DIR"
```

**MCP 모드** (`DATA_SOURCE=mcp`, 기본):
```
  1) get_design_context 호출:
     mcp__figma-desktop__get_design_context(
       nodeId: "<node_id>",
       clientLanguages: "kotlin",
       clientFrameworks: "jetpack-compose"
     )
     → 응답을 <NODE_DIR>/context.json에 저장
     sleep(DELAY_SECONDS)

  2) get_screenshot 호출:
     mcp__figma-desktop__get_screenshot(
       nodeId: "<node_id>",
       clientLanguages: "kotlin",
       clientFrameworks: "jetpack-compose"
     )
     → 이미지를 <NODE_DIR>/screenshot.png에 저장
     sleep(DELAY_SECONDS)

  3) get_variable_defs 호출:
     mcp__figma-desktop__get_variable_defs(nodeId: "<node_id>")
     → 응답을 <NODE_DIR>/variables.json에 저장
     sleep(DELAY_SECONDS)

  4) SVG 내보내기 (REST API 전용):
     MCP 모드에서는 SVG 내보내기를 지원하지 않으므로,
     FIGMA_TOKEN이 있으면 REST API로 SVG를 추가 수집합니다.
     FIGMA_TOKEN이 없으면 이 단계를 건너뜁니다.

     SVG_URL=$(curl -s -H "X-Figma-Token: ${FIGMA_TOKEN}" \
       "https://api.figma.com/v1/images/${FILE_KEY}?ids=${NODE_ID}&format=svg" \
       | python3 -c "import json,sys; d=json.load(sys.stdin); print(list(d['images'].values())[0])")
     curl -s -o "${NODE_DIR}/design.svg" "${SVG_URL}"
     sleep(DELAY_SECONDS)
```

**REST API 모드** (`DATA_SOURCE=rest-api` 또는 MCP에서 자동 전환):
```bash
  # 1) 노드 데이터 (구조 + 스타일) — get_design_context 대체
  curl -s -H "X-Figma-Token: ${FIGMA_TOKEN}" \
    "https://api.figma.com/v1/files/${FILE_KEY}/nodes?ids=${NODE_ID}&geometry=paths" \
    -o "${NODE_DIR}/context.json"
  sleep ${DELAY_SECONDS}

  # 2) 스크린샷 — get_screenshot 대체
  #    먼저 이미지 렌더링 URL을 요청하고, 해당 URL에서 이미지를 다운로드
  IMG_URL=$(curl -s -H "X-Figma-Token: ${FIGMA_TOKEN}" \
    "https://api.figma.com/v1/images/${FILE_KEY}?ids=${NODE_ID}&format=png&scale=2" \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(list(d['images'].values())[0])")
  curl -s -o "${NODE_DIR}/screenshot.png" "${IMG_URL}"
  sleep ${DELAY_SECONDS}

  # 3) 변수/토큰 — get_variable_defs 대체
  curl -s -H "X-Figma-Token: ${FIGMA_TOKEN}" \
    "https://api.figma.com/v1/files/${FILE_KEY}/variables/local" \
    -o "${NODE_DIR}/variables.json"
  sleep ${DELAY_SECONDS}

  # 4) SVG 내보내기
  SVG_URL=$(curl -s -H "X-Figma-Token: ${FIGMA_TOKEN}" \
    "https://api.figma.com/v1/images/${FILE_KEY}?ids=${NODE_ID}&format=svg" \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(list(d['images'].values())[0])")
  curl -s -o "${NODE_DIR}/design.svg" "${SVG_URL}"
  sleep ${DELAY_SECONDS}
```

> **REST API 제한**: per-minute 제한만 존재 (Pro: 15/min, Enterprise: 120/min).
> 일일 제한이 없으므로 `DELAY_SECONDS=5`로 설정하면 분당 12회로 안전하게 수집 가능합니다.

```
진행 표시:
[1/M] 메인화면 - 앨범 있는 상태 ✅
[2/M] 메인화면 - 앨범 없는 상태 ...
```

**Rate limit 에러 처리 및 자동 전환**:
```
에러 감지 시 ("Rate limit exceeded" 또는 429 에러):

  [MCP → REST API 자동 전환] (DATA_SOURCE=mcp이고 FIGMA_TOKEN이 있는 경우):
    1. "MCP rate limit 도달. REST API로 자동 전환합니다." 표시
    2. DATA_SOURCE를 "rest-api"로 변경
    3. 현재 노드부터 REST API 모드로 계속 수집
    4. cache-meta.json에 "data_source": "mixed (mcp+rest-api)" 기록

  [전환 불가] (FIGMA_TOKEN이 없는 경우):
    1. 현재까지 수집 완료된 데이터는 보존
    2. 남은 노드 목록을 resume-list.json에 저장:
       {
         "remaining_nodes": [
           { "node_id": "700:12935", "screen_name": "메인화면", "label": "앨범 없는 상태" },
           ...
         ],
         "completed_nodes": N,
         "total_nodes": M,
         "interrupted_at": "2026-03-28T15:30:00+09:00"
       }
    3. cache-meta.json의 status를 "partial"로 저장
    4. 사용자에게 안내:
       "Rate limit에 도달했습니다. N/M개 수집 완료.
        REST API로 전환하려면 FIGMA_TOKEN을 설정하세요:
          export FIGMA_TOKEN=<your-token>
        이어서 수집:
          /figma-cache --resume"
```

**기타 에러 처리**:
- get_design_context 실패: 1회 재시도, 실패 시 해당 노드 Skip + 기록
- get_screenshot 실패: 1회 재시도, 실패 시 context/variables만 저장 + 기록
- get_variable_defs 실패: 1회 재시도, 실패 시 Skip + 기록

### Step 6. 캐시 메타데이터 생성

```
<CACHE_DIR>/cache-meta.json:
{
  "figma_file_key": "<URL에서 추출>",
  "figma_url": "<원본 URL>",
  "root_node_id": "<최상위 node_id>",
  "exported_at": "<현재 ISO 8601 타임스탬프>",
  "data_source": "mcp" | "rest-api" | "mixed (mcp+rest-api)",
  "delay_seconds": <DELAY_SECONDS>,
  "total_nodes": M,
  "completed_nodes": M,
  "skipped_nodes": [],
  "status": "complete",
  "screen_groups": [
    {
      "screen_name": "메인화면",
      "figma_nodes": [
        { "node_id": "700:7506", "label": "앨범 있는 상태", "cached": true },
        { "node_id": "700:12935", "label": "앨범 없는 상태", "cached": true }
      ]
    },
    ...
  ]
}
```

```
<CACHE_DIR>/screen-groups.json:
→ cache-meta.json의 screen_groups를 별도 파일로도 저장
  (design-qa-all --cache에서 빠르게 읽기 위함)
```

### Step 7. .gitignore 확인

```
프로젝트의 .gitignore에 .figma-cache가 포함되어 있는지 확인:
  Grep: ".figma-cache" in <PROJECT_ROOT>/.gitignore

  없으면 사용자에게 안내:
    ".figma-cache/ 를 .gitignore에 추가하시겠습니까?
     (캐시에 이미지 파일이 포함되어 있어 Git에 올리지 않는 것을 권장합니다)"
```

### Step 8. 완료 보고

```
## Figma 캐시 수집 완료

캐시 경로: <PROJECT_ROOT>/.figma-cache/
수집 화면: N개 (상태 포함 M개 프레임)
API 호출: K회
스킵: S개

캐시를 사용하여 디자인 QA를 실행하세요:
  /design-qa-all --cache
  모듈: feature/signup, feature/home

캐시 유효기간: Figma 명세가 변경되면 다시 /figma-cache를 실행하세요.
```

---

## 캐시 디렉토리 구조

```
<PROJECT_ROOT>/.figma-cache/
  ├─ cache-meta.json               # 캐시 메타데이터
  ├─ screen-groups.json            # 화면 그룹 목록
  ├─ resume-list.json              # 미완료 시에만 생성
  │
  ├─ 700-7506/                     # node-id 기준 폴더 (: → -)
  │   ├─ context.json              # get_design_context 응답
  │   ├─ screenshot.png            # get_screenshot 이미지 (PNG)
  │   ├─ design.svg                # SVG 내보내기 (FIGMA_TOKEN 있을 때만)
  │   └─ variables.json            # get_variable_defs 응답
  │
  ├─ 700-12935/
  │   ├─ context.json
  │   ├─ screenshot.png
  │   ├─ design.svg
  │   └─ variables.json
  │
  └─ ...
```

---

## 예외 처리

| 상황 | 처리 방법 |
|------|----------|
| Figma URL 없음 (resume 아닌 경우) | 사용법 안내 후 종료 |
| MCP rate limit 도달 (FIGMA_TOKEN 있음) | REST API로 자동 전환, 수집 계속 |
| MCP rate limit 도달 (FIGMA_TOKEN 없음) | 현재까지 보존, resume-list.json 생성, 토큰 설정 안내 |
| REST API rate limit (429) | `DELAY_SECONDS`를 2배로 증가 후 60초 대기, 재시도 |
| 개별 노드 수집 실패 | 1회 재시도, 실패 시 Skip + skipped_nodes에 기록 |
| 캐시 디렉토리 이미 존재 | 덮어쓰기 여부 확인 ("기존 캐시를 덮어쓸까요?") |
| resume-list.json 없이 --resume | "이어서 수집할 항목이 없습니다" 안내 |
| 최상위 노드가 너무 큼 (sparse metadata) | 하위 Frame ID 추출 후 개별 호출 (최대 깊이 2) |
| rest-api 모드인데 FIGMA_TOKEN 없음 | 즉시 중단, 토큰 설정 방법 안내 |
