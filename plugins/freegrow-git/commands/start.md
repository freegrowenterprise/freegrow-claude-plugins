---
description: 이슈 생성 + 브랜치 생성을 한번에 수행. Freegrow 워크플로우의 시작점.
---

# Freegrow 작업 시작

이슈를 생성하고, 해당 이슈 번호로 브랜치를 자동 생성합니다.

## 인자

`$ARGUMENTS` - `[타입] 제목` 형식 또는 빈 값 (대화형)

## 실행 절차

### 1. develop 브랜치 최신화

```bash
git checkout develop
git pull origin develop
```

### 2. 이슈 정보 수집

사용자로부터 다음 정보를 수집합니다:

**타입 선택:**
| 타입 | 설명 |
|------|------|
| `[FEAT]` | 새로운 기능 |
| `[FIX]` | 버그 수정 |
| `[DOCS]` | 문서 변경 |
| `[REFACTOR]` | 리팩토링 |
| `[CHORE]` | 빌드, 설정 변경 |
| `[TEST]` | 테스트 추가/수정 |

**제목:** 이슈 제목 (예: 사용자 인증 기능 구현)

**설명 (선택):** 이슈 본문에 들어갈 설명

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
/freegrow-git:start                           # 대화형 - 타입, 제목 순차 입력
/freegrow-git:start [FEAT] 사용자 인증        # 타입+제목 직접 입력
/freegrow-git:start feat 로그인 기능          # 간단 형식
```

## 자동 변환 규칙

| 입력 | 이슈 제목 | 브랜치 |
|------|----------|--------|
| `[FEAT] 사용자 인증` | `[FEAT] 사용자 인증` | `feature/3-user-auth` |
| `feat 로그인 기능` | `[FEAT] 로그인 기능` | `feature/3-login` |
| `fix QR 스캔 버그` | `[FIX] QR 스캔 버그` | `feature/3-qr-scan-bug` |

## 주의사항

- develop 브랜치에서 시작
- Assignee 자동 설정: `MMMIIIN`
- 이슈 생성 실패 시 브랜치 생성 안함
- 이미 같은 브랜치가 있으면 경고
