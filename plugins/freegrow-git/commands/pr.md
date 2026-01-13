---
description: Freegrow 템플릿 기반 PR 생성. Development Log 자동 정리 및 이슈 연동.
---

# Freegrow PR 생성

Freegrow 팀 템플릿에 맞는 Pull Request를 생성합니다. Development Log는 커밋 히스토리를 분석하여 자동 생성됩니다.

## 인자

`$ARGUMENTS` - PR 제목 또는 빈 값 (자동 생성)

## 실행 절차

### 1. 현재 브랜치 정보 수집

```bash
git branch --show-current
git log develop..HEAD --oneline
git diff develop..HEAD --stat
```

### 2. 이슈 번호 추출

브랜치 이름에서 이슈 번호를 추출합니다:
- `feature/3-user-auth` → 이슈 번호: `3`
- 추출 실패 시 사용자에게 입력 요청

### 3. PR 타입 결정

커밋 히스토리를 분석하여 주요 타입을 결정합니다:

| 타입 | PR 제목 형식 |
|------|-------------|
| 새 기능 | `[FEAT] 기능명` |
| 버그 수정 | `[FIX] 버그 설명` |
| 문서 | `[DOCS] 문서명` |
| 리팩토링 | `[REFACTOR] 대상` |
| 설정 | `[CHORE] 변경 내용` |
| 테스트 | `[TEST] 테스트 대상` |

### 4. Development Log 자동 생성

커밋 히스토리와 변경 파일을 분석하여 개발 로그를 자동 생성합니다:

**분석 대상:**
```bash
git log develop..HEAD --pretty=format:"%h %s"
git diff develop..HEAD --stat
git diff develop..HEAD --name-only
```

**생성 형식:**
```markdown
## 📝 Development Log

### 주요 변경 사항
- feat: 사용자 인증 기능 추가 (a1b2c3d)
- fix: 로그인 버그 수정 (d4e5f6g)

### 변경된 파일
- `src/auth/login.dart` - 로그인 로직 구현
- `src/models/user.dart` - 사용자 모델 추가

### 작업 요약
{커밋 메시지와 변경 내용을 바탕으로 작성한 간결한 요약}
```

### 5. PR 본문 생성

Freegrow 템플릿 형식으로 본문을 작성합니다:

```markdown
## 📋 Summary
{커밋 히스토리 기반 요약 - 무엇을 왜 변경했는지}

## 📣 Related Issue
- close #이슈번호

---

## ✨ 구현된 기능

### 1. {주요 기능명}
- {기능 상세 1}
- {기능 상세 2}

---

## 📝 Development Log

### 주요 변경 사항
{커밋 목록}

### 변경된 파일
{파일별 변경 내용 요약}

### 작업 요약
{전체 작업 내용 정리}
```

### 6. 푸시 확인

푸시되지 않은 커밋이 있으면 먼저 푸시합니다:
```bash
git push -u origin $(git branch --show-current)
```

### 7. PR 생성

```bash
gh pr create \
  --base develop \
  --title "[타입] PR 제목" \
  --body "PR 본문" \
  --assignee @me
```

### 8. 결과 출력

```
✅ PR 생성 완료!

PR: [FEAT] 사용자 인증 기능
URL: https://github.com/owner/repo/pull/5

연결된 이슈: #3
브랜치: feature/3-user-auth → develop
```

## 사용 예시

```
/freegrow-git:pr                      # 자동 생성 (브랜치/커밋 분석)
/freegrow-git:pr [FEAT] 사용자 인증    # 제목 직접 지정
```

## 주의사항

- 기본 브랜치는 `develop`
- Assignee 자동 설정: 현재 로그인된 사용자 (`@me`)
- Development Log는 커밋 분석으로 자동 생성
- 푸시되지 않은 커밋이 있으면 먼저 푸시
