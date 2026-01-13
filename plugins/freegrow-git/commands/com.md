---
description: Freegrow 형식으로 커밋 생성. 형식: <type>: <description> (#이슈번호)
---

# Freegrow 커밋

Freegrow 팀 컨벤션에 맞는 커밋을 생성합니다.

## 인자

`$ARGUMENTS` - 커밋 메시지 또는 빈 값 (대화형)

## 실행 절차

### 1. 현재 상태 확인

```bash
git status
git diff --staged --stat
```

스테이징된 파일이 없으면 `git add .` 실행 여부를 사용자에게 확인합니다.

### 2. 이슈 번호 확인

현재 브랜치에서 이슈 번호를 추출합니다:
```bash
git branch --show-current
```

브랜치 형식이 `feature/123-description`이면 이슈 번호는 `123`입니다.
이슈 번호를 찾을 수 없으면 사용자에게 직접 입력받습니다.

### 3. 커밋 타입 결정

변경 내용을 분석하여 적절한 타입을 제안합니다:

| 타입 | 사용 시점 |
|------|----------|
| `feat` | 새 기능 추가 |
| `fix` | 버그 수정 |
| `docs` | 문서 변경 (.md 파일) |
| `style` | 포맷팅, 세미콜론 등 |
| `refactor` | 기능 변경 없는 코드 개선 |
| `test` | 테스트 추가/수정 |
| `chore` | 빌드, 설정 파일 변경 |

### 4. 커밋 메시지 생성

형식: `<type>: <description> (#이슈번호)`

예시:
- `feat: 사용자 인증 기능 추가 (#3)`
- `fix: 로그인 버그 수정 (#5)`
- `docs: README 업데이트 (#7)`

### 5. PR 로그 파일 포함

`.claude/pr-logger/session-log-*.md` 파일이 수정되었으면 함께 스테이징합니다:
```bash
git add .claude/pr-logger/
```

### 6. 커밋 실행

```bash
git commit -m "<type>: <description> (#이슈번호)"
```

### 7. 결과 출력

커밋 해시와 메시지를 출력합니다:
```bash
git log -1 --oneline
```

## 사용 예시

```
/freegrow-git:com                    # 대화형 - 타입, 메시지, 이슈번호 순차 입력
/freegrow-git:com feat: 로그인 구현   # 이슈번호는 브랜치에서 자동 추출
/freegrow-git:com fix: 버그 수정 #5  # 전체 메시지 직접 입력
```

## 주의사항

- 이슈 번호 없이는 커밋 불가
- PR 로그 파일 자동 포함
- 스테이징된 파일 없으면 확인 후 진행
