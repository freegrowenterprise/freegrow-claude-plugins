---
description: 워크트리 기반 격리 작업 시작. 이슈 생성 + git worktree로 독립 디렉토리 생성.
---

# Freegrow 워크트리 작업 시작

이슈를 생성하고, **격리된 워크트리 디렉토리**에서 작업을 시작합니다.
기존 브랜치 작업을 유지하면서 병렬로 새 작업을 할 때 유용합니다.

## 인자

`$ARGUMENTS` - 작업 내용 (타입은 자동 판단)

## 실행 절차

### 1. 현재 상태 확인

```bash
git status
git branch --show-current
```

> 현재 브랜치의 미커밋 변경사항이 있으면 경고를 출력합니다. (워크트리 생성은 계속 진행)

### 2. 타입 자동 판단

**사용자가 타입을 명시하지 않으면 내용에서 자동으로 판단합니다.**

| 키워드 | 판단 타입 |
|--------|----------|
| `fix`, `bug`, `수정`, `버그`, `오류`, `에러` | `[FIX]` |
| `docs`, `readme`, `문서`, `doc` | `[DOCS]` |
| `refactor`, `리팩`, `개선`, `정리` | `[REFACTOR]` |
| `chore`, `설정`, `빌드`, `config`, `ci`, `cd` | `[CHORE]` |
| `test`, `테스트` | `[TEST]` |
| 그 외 모든 경우 | `[FEAT]` (기본값) |

### 3. 이슈 생성

```bash
gh issue create \
  --title "[타입] 제목" \
  --body "## 💡 Issue
이슈 설명

## 📝 todo
- [ ] 작업 항목" \
  --assignee @me
```

### 4. 이슈 번호 추출

```bash
gh issue list --limit 1 --json number --jq '.[0].number'
```

### 5. 브랜치 및 워크트리 이름 생성

이슈 제목에서 브랜치 설명을 추출합니다:
- `[FEAT] 사용자 인증 기능` → 브랜치: `feature/3-user-auth`, 워크트리 디렉토리: `.claude/worktrees/3-user-auth`

규칙:
- 소문자 변환
- 공백 → 하이픈
- 특수문자 제거
- 영문 변환 권장

### 6. 워크트리 디렉토리 생성

`.claude/worktrees/` 디렉토리가 없으면 먼저 생성합니다:

```bash
mkdir -p .claude/worktrees
```

새 브랜치와 함께 워크트리를 생성합니다:

```bash
git worktree add .claude/worktrees/이슈번호-설명 -b feature/이슈번호-설명
```

### 7. .gitignore 확인

`.claude/worktrees/`가 `.gitignore`에 등록되어 있는지 확인합니다.
없으면 자동으로 추가합니다:

```bash
grep -q ".claude/worktrees" .gitignore || echo ".claude/worktrees/" >> .gitignore
```

### 8. 결과 출력

```
✅ 워크트리 작업 시작 완료!

이슈: #3 [FEAT] 사용자 인증 기능
URL: https://github.com/owner/repo/issues/3

브랜치: feature/3-user-auth
워크트리: .claude/worktrees/3-user-auth

📁 워크트리 이동 방법:
  cd .claude/worktrees/3-user-auth

🧹 작업 완료 후 정리 방법:
  git worktree remove .claude/worktrees/3-user-auth
  git branch -d feature/3-user-auth

다음 단계:
1. 위 경로로 이동하여 코드 작업
2. /com 으로 커밋 (워크트리 내에서)
3. /pr 으로 PR 생성
```

## 사용 예시

```
/freegrow-git:wt 새 기능 병렬 개발       # → [FEAT] 새 기능 병렬 개발
/freegrow-git:wt 긴급 버그 수정          # → [FIX] 긴급 버그 수정
/freegrow-git:wt [FEAT] 결제 모듈 추가   # → [FEAT] 결제 모듈 추가
```

## start vs wt 비교

| 항목 | `/start` | `/wt` |
|------|----------|-------|
| 작업 디렉토리 | 현재 repo (브랜치 전환) | 격리된 워크트리 디렉토리 |
| 병렬 작업 | 불가 (브랜치 전환 필요) | 가능 (각 워크트리 독립) |
| 사용 시점 | 일반적인 단일 작업 | 여러 이슈 동시 작업 |
| 정리 필요 | 없음 | `git worktree remove` 필요 |

## 주의사항

- develop 브랜치 기준으로 워크트리가 생성됩니다
- `.claude/worktrees/` 경로는 `.gitignore`에 자동 추가됩니다
- 워크트리 작업 완료 후 반드시 `git worktree remove`로 정리하세요
- Assignee 자동 설정: 현재 로그인된 사용자 (`@me`)
