---
description: 이슈 생성 + 브랜치 생성을 한번에 수행. Freegrow 워크플로우의 시작점.
---

# Freegrow 작업 시작

이슈를 생성하고, 해당 이슈 번호로 브랜치를 자동 생성합니다.

## 인자

`$ARGUMENTS` - 작업 내용 (타입은 자동 판단)

## 실행 절차

### 1. develop 브랜치 최신화

```bash
git checkout develop
git pull origin develop
```

### 2. 타입 자동 판단

**사용자가 타입을 명시하지 않으면 내용에서 자동으로 판단합니다.**

**자동 판단 규칙:**
| 키워드 | 판단 타입 |
|--------|----------|
| `fix`, `bug`, `수정`, `버그`, `오류`, `에러` | `[FIX]` |
| `docs`, `readme`, `문서`, `doc` | `[DOCS]` |
| `refactor`, `리팩`, `개선`, `정리` | `[REFACTOR]` |
| `chore`, `설정`, `빌드`, `config`, `ci`, `cd` | `[CHORE]` |
| `test`, `테스트` | `[TEST]` |
| 그 외 모든 경우 | `[FEAT]` (기본값) |

**판단 불가 시 `[FEAT]`로 자동 설정됩니다. 사용자에게 타입을 묻지 않습니다.**

**이미 타입이 명시된 경우:**
- `[FEAT] 기능명` 또는 `feat 기능명` 형식은 그대로 사용

### 3. 이슈 생성

```bash
gh issue create \
  --title "[타입] 제목" \
  --body "## 💡 Issue
이슈 설명

## 📝 todo
- [ ] 작업 항목" \
  --assignee MMMIIIN
```

### 4. 이슈 번호 추출

생성된 이슈 번호를 추출합니다:
```bash
gh issue list --limit 1 --json number --jq '.[0].number'
```

### 5. 브랜치 이름 생성

이슈 제목에서 브랜치 설명을 추출합니다:
- `[FEAT] 사용자 인증 기능` → `feature/3-user-auth`

규칙:
- 소문자 변환
- 공백 → 하이픈
- 특수문자 제거
- 영문 변환 권장

### 6. 브랜치 생성 및 체크아웃

```bash
git checkout -b feature/이슈번호-설명
```

### 7. 결과 출력

```
✅ 작업 시작 완료!

이슈: #3 [FEAT] 사용자 인증 기능
URL: https://github.com/owner/repo/issues/3

브랜치: feature/3-user-auth
현재 위치: feature/3-user-auth

다음 단계:
1. 코드 작성
2. /com 으로 커밋
3. /pr 으로 PR 생성
```

## 사용 예시

```
/freegrow-git:start 새 테스트 작업 시작       # → [FEAT] 새 테스트 작업 시작 (기본값)
/freegrow-git:start 로그인 버그 수정          # → [FIX] 로그인 버그 수정 (버그 키워드)
/freegrow-git:start README 문서 업데이트      # → [DOCS] README 문서 업데이트 (문서 키워드)
/freegrow-git:start [FEAT] 사용자 인증        # → [FEAT] 사용자 인증 (명시적 타입)
```

## 자동 변환 규칙

| 입력 | 이슈 제목 | 브랜치 |
|------|----------|--------|
| `사용자 인증 기능` | `[FEAT] 사용자 인증 기능` | `feature/3-user-auth` |
| `로그인 버그 수정` | `[FIX] 로그인 버그 수정` | `feature/3-login-bug-fix` |
| `README 문서 업데이트` | `[DOCS] README 문서 업데이트` | `feature/3-readme-docs` |
| `[FEAT] 사용자 인증` | `[FEAT] 사용자 인증` | `feature/3-user-auth` |

## 주의사항

- develop 브랜치에서 시작
- Assignee 자동 설정: `MMMIIIN`
- 이슈 생성 실패 시 브랜치 생성 안함
- 이미 같은 브랜치가 있으면 경고
