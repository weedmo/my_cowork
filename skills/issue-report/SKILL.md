---
name: issue-report
description: "제조 현장 이슈 문서화 — 비정형 텍스트(대화, 메모, 에러 로그)를 Notion 데이터베이스에 구조화된 트러블슈팅 문서로 작성. /issue-report로 문서 생성, search로 검색, list로 목록 조회."
---

# Issue Report — 제조 현장 이슈 문서화 (Notion 전용)

로보틱스/AI 제조 환경에서 비정형 이슈 텍스트(대화, 메모, 에러 로그 등)를 구조화된 트러블슈팅 문서로 변환하여 **Notion 데이터베이스에 저장**한다.

- **입력**: 자유 형식의 이슈 설명 (한국어, 영어, 혼합)
- **출력**: YAML frontmatter + Fix-First 본문 구조의 Notion 페이지
- **저장**: Notion 데이터베이스 (필수 — 다른 저장 방식 없음)

핵심 원칙: **즉시 대응(Quick Response) 정보가 최상단** — 다운타임 비용 최소화.

## Prerequisites

이 스킬은 Notion MCP 서버가 설정되어 있어야 합니다.
설정되지 않은 경우 사용자에게 안내:

```
Notion MCP 서버가 설정되어 있지 않습니다.
~/.claude/settings.json의 mcpServers에 notion 설정을 추가해주세요.
```

## Subcommands

| Command | Action | User Confirmation |
|---------|--------|-------------------|
| `/issue-report` (no args) | 현재 대화에서 이슈 추출, Notion에 문서 생성 | Required |
| `/issue-report <텍스트>` | 입력 텍스트로 Notion 이슈 문서 생성 | Required |
| `/issue-report search <keyword>` | Notion 데이터베이스에서 이슈 검색 | Not needed |
| `/issue-report list` | Notion 데이터베이스 전체 이슈 목록 조회 | Not needed |

## Procedure

### Step 0: Verify Notion Connection

1. Use `mcp__notion__notion-search` to verify the Notion MCP server is available.
2. If unavailable, display the prerequisites error and stop.

### Step 1: Parse Subcommand

Parse the user's input to determine which subcommand to execute:
- No args → **Record workflow** (Step 2)
- `<텍스트>` (not a subcommand) → **Record from text** (Step 2, using provided text)
- `search <keyword>` → **Search workflow** (Step 5)
- `list` → **List workflow** (Step 6)

### Step 2: Extract Information from Input

Analyze the conversation or provided text to extract:

**Required fields** (누락 시 사용자에게 질문):
- **증상 (Symptoms)**: 관찰된 동작, 에러 코드, 에러 메시지
- **해결 방법 (Solution)**: 적용한 수정 또는 임시 대응 (미해결이면 `status: unresolved`)

**Auto-detect fields** (텍스트에서 자동 추출 시도):
- **cell**: 셀 번호 (예: cell003, Cell-5 → cell005)
- **severity**: critical/high/medium/low (에러 심각도에서 추론)
- **environment**: IP, SW 버전, FW 버전 등
- **tags**: 핵심 키워드 (장비명, 에러코드, SW 모듈명)

**Optional fields** (있으면 포함, 없으면 생략):
- **재발시 대처 (If Recurs)**: 반복 시 에스컬레이션 절차
- **재현 방법 (Reproduction Steps)**: 재현 절차와 재현율
- **로그 및 자료 (Logs & Evidence)**: 에러 로그, 영상/사진 링크
- **비고 (Notes)**: 기대 결과, 특이사항, 재발 방지책

### Step 3: Ask for Missing Required Information

If required fields are missing, use AskUserQuestion to ask. Example:

```
이슈 문서를 작성하려면 추가 정보가 필요합니다:

1. **증상**: 구체적인 에러 메시지나 관찰된 동작이 있나요?
2. **해결 방법**: 어떻게 해결했나요? (미해결이면 "미해결"이라고 알려주세요)
3. **셀 번호**: 어느 셀에서 발생했나요?
```

Only ask about genuinely missing information — do not ask about optional fields unless context suggests they exist.

### Step 4: Generate Document and Confirm

1. Generate next **Issue ID**:
   - Use `mcp__notion__notion-search` to find existing issue pages in the target database.
   - Parse existing IDs to determine the next sequential number.
   - Format: `ISSUE-YYYY-NNN` (e.g., ISSUE-2026-004)

2. Show the draft to the user:

   ```
   ## Issue Draft: {next_id}

   **Cell**: {cell} | **Severity**: {severity} | **Status**: {status}
   **Tags**: {tag1, tag2, ...}

   ### 증상 (Symptoms)
   - ...

   ### 해결 방법 (Solution)
   1. ...

   ### 재발시 대처 (If Recurs)
   - ...

   ---

   ### 이슈 정리 (Issue Summary)
   ...

   ### 재현 방법 (Reproduction Steps)
   1. ...

   ### 로그 및 자료 (Logs & Evidence)
   - ...

   ### 비고 (Notes)
   - ...
   ```

3. **Wait for user confirmation** using AskUserQuestion:

   ```
   이 내용으로 Notion에 이슈 문서를 생성할까요? (수정할 부분이 있으면 말씀해주세요)
   ```

4. On confirmation, execute Step 4A.

#### Step 4A: Create Notion Page

1. Use `mcp__notion__notion-search` to find the target database.
   - Search for "이슈", "Issue", or "Issue Report" database.
   - If multiple databases found, ask the user which one to use.
   - If no database found, ask the user for the database name or URL.

2. Map fields to Notion properties:

   | Field | Notion Property Type | Notes |
   |-------|---------------------|-------|
   | `id` (ISSUE-YYYY-NNN) | **Title** | Primary property |
   | `cell` | **Select** | e.g., cell003, cell005 |
   | `severity` | **Select** | critical/high/medium/low |
   | `status` | **Select** | resolved/unresolved |
   | `tags` | **Multi-select** | Keywords array |
   | `created` | **Date** | ISO 8601 format |
   | `environment` | **Rich text** | JSON or structured text |

3. Convert markdown body to Notion blocks:
   - `## heading` → heading_2 block
   - `### heading` → heading_3 block
   - `- item` → bulleted_list_item block
   - `1. item` → numbered_list_item block
   - `` `code` `` → code inline
   - `---` → divider block
   - Plain text → paragraph block

4. Use `mcp__notion__notion-create-pages` to create the page with:
   - Parent: target database ID
   - Properties: mapped fields
   - Children: converted blocks

5. Display the result:

   ```
   ✅ Notion 이슈 문서가 생성되었습니다.

   - **ID**: {next_id}
   - **제목**: {title}
   - **URL**: {notion_page_url}
   ```

### Step 5: Search (`/issue-report search <keyword>`)

1. Use `mcp__notion__notion-search` with the keyword to search the issue database.
   - Filter by database if possible.
   - Search across title, tags, and body content.

2. Display results:
   ```
   ## Issue Search: "<keyword>"

   | ID | Status | Cell | Severity | Title | URL |
   |----|--------|------|----------|-------|-----|
   | ISSUE-2026-001 | resolved | cell003 | high | 텔레옵 토크 미인가 | [link] |
   ```

3. If no results: `"<keyword>"에 대한 이슈 문서를 찾지 못했습니다.`

### Step 6: List (`/issue-report list`)

1. Use `mcp__notion__notion-search` to query the issue database.
   - Retrieve all pages with "ISSUE-" prefix in title.

2. For each page, extract properties: id, cell, severity, status, title, created date.

3. Display grouped by status:
   ```
   ## Issue Documents (Notion)

   ### Unresolved
   | ID | Cell | Severity | Title | Created | URL |
   |----|------|----------|-------|---------|-----|
   | ISSUE-2026-002 | cell005 | critical | Dataset Manager 프리징 | 2026-03-10 | [link] |

   ### Resolved
   | ID | Cell | Severity | Title | Created | URL |
   |----|------|----------|-------|---------|-----|
   | ISSUE-2026-001 | cell003 | high | 텔레옵 토크 미인가 | 2026-03-08 | [link] |
   ```

## Document Template (for Notion body)

The Notion page body follows this structure:

```
# [한줄 요약] 텔레옵 활성화 시 로봇 팔 토크 미인가

## 즉시 대응 (Quick Response)

### 증상 (Symptoms)
- 텔레옵 버튼 클릭 시 UI 활성화 표시되나 실제 토크 미인가
- Error Code: `E-402` (서보 드라이버 타임아웃)

### 해결 방법 (Solution)
1. 캘리브레이션 초기화
2. 서보 드라이버 콜드 부팅
3. 정상 동작 확인

### 재발시 대처 (If Recurs)
- 동일 증상 3회 반복 시 하네스 교체
- 네트워크 대역폭 모니터링 확인

---

## 상세 기록 (Investigation Detail)

### 이슈 정리 (Issue Summary)
문제 배경, 발견 경위, 근본 원인 분석

### 재현 방법 (Reproduction Steps)
1. Dataset Manager 실행 → 로봇 연결
2. Record 버튼 클릭
3. 약 2초 후 화면 멈춤 (재현율: 100%)

### 로그 및 자료 (Logs & Evidence)
- [에러 로그 텍스트/링크]
- [영상/사진 링크]

### 비고 (Notes)
- 기대 결과: 끊김 없이 데이터 저장
- 특이사항: Cell005에서는 정상 → 네트워크 문제 가능성
- 재발 방지: 매주 월요일 Network Health Check 실행
```

## Unresolved Document Template

For issues without a solution yet, replace "해결 방법" section with:

```
### 시도한 방법 (Attempted Solutions)
1. Dataset Manager 재시작 → 동일 증상 반복
2. 다른 Cell에서 테스트 → Cell003에서는 정상
3. 네트워크 케이블 교체 → 효과 없음

### 현재 상태 (Current Status)
- Cell005에서만 재현, 네트워크 스위치 문제 의심
- 네트워크 팀 확인 대기 중
```

## Information Extraction Patterns

When parsing unstructured input, look for these patterns:

| Pattern | Extraction Target |
|---------|-------------------|
| `cell\d+`, `Cell-?\d+`, `셀\d+` | cell field |
| `\d+\.\d+\.\d+\.\d+` | console_ip |
| `v\d+\.\d+\.\d+` | SW/FW version |
| `E-\d+`, `Error:`, `에러` | error code / symptoms |
| `해결`, `고침`, `fixed`, `solved` | solution markers |
| `재현`, `reproduce`, `반복` | reproduction info |
| `bg\d+_rev\d+` | firmware version |

## Key Principles

- **Notion-Only**: 모든 이슈 문서는 반드시 Notion 데이터베이스에 저장 — 로컬 파일 저장 없음
- **Fix-First**: 즉시 대응 정보가 항상 최상단 — 현장 엔지니어가 스크롤 없이 해결책 확인
- **Structured Properties**: Notion Select/Multi-select로 필터링/정렬/통계 용이
- **Quick/Detail 구분선**: 현장 엔지니어(Quick) vs R&D 팀(Detail) 각각 필요한 부분만 읽기
- **Resolved 문서는 반드시 사용자 확인** 후 저장
- **누락 정보는 질문으로 보완** — 추측으로 채우지 않음
- **ID 형식**: `ISSUE-YYYY-NNN` (연도별 시퀀스, 재사용 안 함)
- **TSG 스킬과 독립적** — TSG는 코드 이슈, issue-report는 제조 현장 이슈
