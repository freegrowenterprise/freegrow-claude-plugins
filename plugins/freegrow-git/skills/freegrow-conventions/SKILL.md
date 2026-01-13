---
name: freegrow-conventions
description: Freegrow 팀 Git 워크플로우 컨벤션. 커밋, 브랜치, 이슈, PR 생성 시 자동 참조됨.
---

# Freegrow Git 컨벤션

이 스킬은 Freegrow 팀의 Git 워크플로우 규칙을 정의합니다. 커밋, 브랜치, 이슈, PR 생성 시 반드시 이 규칙을 따릅니다.

## 커밋 메시지 형식

```
<type>: <description> (#이슈번호)
```

### 타입 종류
| 타입 | 설명 | 예시 |
|------|------|------|
| `feat` | 새로운 기능 | `feat: 사용자 인증 기능 추가 (#3)` |
| `fix` | 버그 수정 | `fix: 로그인 실패 버그 수정 (#5)` |
| `docs` | 문서 변경 | `docs: README 업데이트 (#7)` |
| `style` | 코드 포맷팅 | `style: 코드 정렬 (#8)` |
| `refactor` | 리팩토링 | `refactor: 인증 모듈 구조 개선 (#9)` |
| `test` | 테스트 추가/수정 | `test: 로그인 테스트 추가 (#10)` |
| `chore` | 빌드, 설정 변경 | `chore: CI 설정 수정 (#11)` |

### 규칙
- 이슈 번호는 **필수** - 모든 커밋 끝에 `(#이슈번호)` 포함
- 한글 사용 권장
- 50자 이내 권장

## 브랜치 명명 규칙

```
feature/이슈번호-간단한-설명
```

### 예시
- `feature/3-user-authentication`
- `feature/5-qr-scan-fix`
- `feature/7-readme-update`

### 규칙
- 소문자만 사용
- 단어 구분은 하이픈(`-`) 사용
- 이슈 번호 필수 포함

## 이슈 제목 형식

```
[타입] 제목
```

### 타입 종류
| 타입 | 설명 |
|------|------|
| `[FEAT]` | 새로운 기능 |
| `[FIX]` | 버그 수정 |
| `[DOCS]` | 문서 변경 |
| `[REFACTOR]` | 리팩토링 |
| `[CHORE]` | 빌드, 설정 변경 |
| `[TEST]` | 테스트 추가/수정 |

### 이슈 본문 템플릿
```markdown
## 💡 Issue
<!-- 이슈에 대한 내용을 설명해주세요. -->

## 📝 todo
<!-- 해야 할 일들을 적어주세요. -->
- [ ] todo !
```

## PR 제목 형식

```
[타입] 제목
```

커밋 타입과 동일한 규칙 적용 (대문자)

### PR 본문 템플릿
```markdown
## 📋 Summary
<!-- PR에 대한 간단한 설명을 작성해주세요. -->


## 📣 Related Issue
- close #이슈번호

---

## ✨ 구현된 기능

### 1. 기능명
- 기능 상세 1
- 기능 상세 2

---

## 📝 Development Log
<!-- 커밋 히스토리와 변경 파일을 분석하여 자동 생성 -->

### 주요 변경 사항
- feat: 기능 설명 (커밋해시)
- fix: 수정 내용 (커밋해시)

### 변경된 파일
- `경로/파일명` - 변경 내용 요약

### 작업 요약
작업 내용을 간결하게 정리한 문단.
```

**Development Log 자동 생성 규칙:**
1. `git log develop..HEAD` 분석하여 커밋 목록 추출
2. `git diff develop..HEAD --stat` 분석하여 변경 파일 목록 추출
3. 커밋 메시지와 변경 내용을 바탕으로 작업 요약 작성

## GitHub 규칙

- **Assignee**: 항상 `MMMIIIN`으로 설정
- **기본 브랜치**: `develop` (main 아님)
- **커밋 시**: `.claude/pr-logger/session-log-*.md` 파일 포함

## 워크플로우 순서

```
1. /start → 이슈 생성 + 브랜치 생성 (#3, feature/3-desc)
      ↓
2. /com → 작업 & 커밋 (feat: 기능 구현 (#3))
      ↓
3. /pr → PR 생성 (close #3, Development Log 자동 생성)
      ↓
4. 리뷰 & 머지 → develop
```

## 커맨드 요약

| 커맨드 | 기능 |
|--------|------|
| `/start` | 이슈 생성 + 브랜치 생성 (통합) |
| `/com` | Freegrow 형식 커밋 |
| `/pr` | Freegrow 템플릿 PR 생성 |
