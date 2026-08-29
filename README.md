# Claude × Gemini PR Co-Review

Claude Code가 PR을 만들면 Gemini가 자동 리뷰하고, approve되면 자동 merge되는 자율 루프.

```
"PR 올려줘" 한 마디로:

사용자 요구사항
    → Claude Code: 코드 작성 + PR 생성
    → Gemini: 자동 코드 리뷰 (한국어)
    → APPROVE  → 자동 merge + Google Sheets 로그 ✅
    → REQUEST_CHANGES → Claude Code: 이슈 수정 + push → 재리뷰
    → 반복 → merge ✅
```

---

## 구성

| 파일 | 역할 |
|---|---|
| `setup.sh` | 대상 repo에 한 번만 실행 — 모든 설정 자동화 |
| `workflow.yml` | GitHub Actions: Gemini 리뷰 + 자동 merge + Sheets 로그 |
| `cowork.sh` | Claude Code 모니터링 루프 (리뷰 대기 → 수정 → push 사이클) |
| `AGENTS.md.template` | 리뷰 기준 템플릿 (repo에 없으면 자동 생성) |
| `CLAUDE.md.snippet` | Claude Code가 읽는 PR 워크플로우 규칙 |

---

## 사전 준비

**1. GitHub CLI 설치 및 로그인**
```bash
brew install gh
gh auth login
```

**2. Gemini API Key 발급 (필수)**

[Google AI Studio](https://aistudio.google.com) → API Keys → Create API Key

무료 티어로 충분합니다.

**3. Google Sheets 리뷰 로그 설정 (선택)**

PR 리뷰 결과를 Google Sheets에 자동 기록하려면 아래 순서로 설정합니다.

```
① Google Cloud Console → APIs & Services → Sheets API 활성화

② IAM & Admin → Service Accounts → + CREATE SERVICE ACCOUNT
   → 이름: github-pr-logger → CREATE AND CONTINUE → DONE
   → KEYS 탭 → ADD KEY → JSON → 다운로드

③ Google Sheets에서 새 스프레드시트 생성
   → 공유 → 서비스 계정 이메일 추가 (편집자 권한)
     (이메일은 JSON 파일의 "client_email" 값)

④ Sheet URL에서 ID 복사
   https://docs.google.com/spreadsheets/d/[여기가 SHEET_ID]/edit
```

`setup.sh` 실행 시 JSON 파일 경로와 Sheet ID를 입력하면 자동으로 Secret 등록까지 처리합니다.

---

## 설치

```bash
git clone https://github.com/dong7812/claude-gemini-pr-review
cd claude-gemini-pr-review
./setup.sh <owner/repo>
```

예시:
```bash
./setup.sh dong7812/my-project
```

실행 시 순서대로 입력:
1. Gemini API Key
2. Google SA JSON 파일 경로 (건너뛰려면 Enter)
3. Google Sheet ID (건너뛰려면 Enter)

실행하면 자동으로:
- `GEMINI_API_KEY` GitHub Secret 등록
- `GOOGLE_SA_JSON` / `GOOGLE_SHEET_ID` GitHub Secret 등록 (입력한 경우)
- `.github/workflows/ai-pr-review.yml` 업로드
- `.github/scripts/cowork.sh` 업로드
- `AGENTS.md` 생성 (이미 있으면 스킵)
- `CLAUDE.md`에 PR 워크플로우 규칙 주입
- repo auto-merge 설정 활성화

---

## 사용법

셋업 완료 후 해당 프로젝트를 Claude Code로 열고:

```
"로그인 기능 추가해서 PR 올려줘"
```

Claude Code가 CLAUDE.md를 읽고 자동으로:

1. 코드 작성 및 커밋
2. PR 생성
3. Gemini 리뷰 대기 (한국어 리뷰)
4. REQUEST_CHANGES → 이슈 수정 → push → 재리뷰
5. APPROVE → 자동 merge → Google Sheets에 로그 기록

---

## Google Sheets 로그 항목

PR merge 시 자동으로 한 행이 추가됩니다:

| 날짜 | Repo | PR# | 제목 | 판정 | 요약 | 이슈 수 | 링크 |
|---|---|---|---|---|---|---|---|

---

## 리뷰 기준 커스터마이징

repo 루트의 `AGENTS.md`를 수정하면 Gemini가 해당 기준으로 리뷰합니다.

```markdown
# AGENTS.md

## Must Check (🔴 치명적)
- Unhandled async errors
- Hardcoded secrets
- SQL injection / XSS risks

## Suggestions (🟡 제안)
- Functions longer than 50 lines

## Out of Scope
- Formatting (handled by linter)
```

---

## 수동 트리거

```bash
gh workflow run ai-pr-review.yml --repo <owner/repo> -f pr_number=<N>
```

---

## 구조

```
PR 오픈 / push
    └── GitHub Actions (workflow.yml)
            ├── PR diff 가져오기 (설정 파일 제외)
            ├── Gemini API 호출 (모델 자동 폴백)
            ├── 한국어 리뷰 코멘트 게시
            ├── APPROVE → gh pr merge (자동)
            └── Google Sheets에 로그 기록

로컬 Claude Code (cowork.sh)
    ├── 20초 간격으로 PR 상태 폴링
    ├── Gemini 코멘트 감지
    ├── APPROVE  → 종료
    └── REQUEST_CHANGES → 리뷰 출력 → exit 2
            └── Claude Code: 이슈 수정 → git push → 재실행
```

---

## 비용

- **Claude Code**: 기존 구독 그대로 (추가 비용 없음)
- **Gemini**: Google AI Studio 무료 API Key (무료 티어)
- **Google Sheets API**: 무료
- **GitHub Actions**: 공개 repo 무료 / 비공개 repo 월 2,000분 무료

---

## 다른 프로젝트에 적용

```bash
./setup.sh dong7812/another-project
./setup.sh dong7812/yet-another-project
```

repo마다 한 번씩만 실행하면 됩니다.
