# Freegrow Git 플러그인

Freegrow 팀 전용 Git 워크플로우 자동화 플러그인입니다. 커밋, 브랜치, 이슈, PR 형식을 통일하여 일관된 개발 프로세스를 유지합니다.

## 설치

```bash
/plugin install freegrow-git
```

## 커맨드 요약

| 커맨드 | 설명 |
|--------|------|
| `/freegrow-git:start` | 이슈 생성 + 브랜치 생성 (통합) |
| `/freegrow-git:com` | Freegrow 형식 커밋 |
| `/freegrow-git:pr` | PR 생성 + Development Log 자동 정리 |

## 워크플로우

```
1. /start "[FEAT] 기능명"
      ↓
   이슈 #3 생성 + feature/3-기능명 브랜치 생성
      ↓
2. 코드 작업
      ↓
3. /com
      ↓
   feat: 기능 구현 (#3) 커밋
      ↓
4. /pr
      ↓
   PR 생성 (Development Log 자동 정리)
```

---

## 상세 사용법

### 1. `/start` - 작업 시작

이슈를 생성하고 해당 이슈 번호로 브랜치를 자동 생성합니다.

**사용법:**
```bash
# 대화형 (타입, 제목 순차 입력)
/freegrow-git:start

# 직접 입력
/freegrow-git:start [FEAT] 사용자 인증 기능
/freegrow-git:start feat 로그인 구현
```

**타입 종류:**
| 타입 | 설명 |
|------|------|
| `[FEAT]` / `feat` | 새로운 기능 |
| `[FIX]` / `fix` | 버그 수정 |
| `[DOCS]` / `docs` | 문서 변경 |
| `[REFACTOR]` / `refactor` | 리팩토링 |
| `[CHORE]` / `chore` | 빌드, 설정 변경 |
| `[TEST]` / `test` | 테스트 추가/수정 |

**결과:**
```
✅ 작업 시작 완료!

이슈: #3 [FEAT] 사용자 인증 기능
URL: https://github.com/owner/repo/issues/3

브랜치: feature/3-user-auth
현재 위치: feature/3-user-auth
```

---

### 2. `/com` - 커밋

Freegrow 형식에 맞는 커밋을 생성합니다.

**커밋 형식:**
```
<type>: <description> (#이슈번호)
```

**사용법:**
```bash
# 대화형 (타입, 메시지 순차 입력)
/freegrow-git:com

# 직접 입력 (이슈번호는 브랜치에서 자동 추출)
/freegrow-git:com feat: 로그인 기능 구현

# 이슈번호 직접 지정
/freegrow-git:com fix: 버그 수정 #5
```

**커밋 타입:**
| 타입 | 설명 | 예시 |
|------|------|------|
| `feat` | 새로운 기능 | `feat: 사용자 인증 추가 (#3)` |
| `fix` | 버그 수정 | `fix: 로그인 오류 수정 (#3)` |
| `docs` | 문서 변경 | `docs: README 업데이트 (#3)` |
| `style` | 코드 포맷팅 | `style: 코드 정렬 (#3)` |
| `refactor` | 리팩토링 | `refactor: 인증 모듈 개선 (#3)` |
| `test` | 테스트 | `test: 로그인 테스트 추가 (#3)` |
| `chore` | 빌드/설정 | `chore: CI 설정 수정 (#3)` |

**자동 기능:**
- 브랜치 이름에서 이슈 번호 자동 추출
- 스테이징 안된 파일 확인 후 `git add` 제안
- PR 로그 파일 자동 포함

---

### 3. `/pr` - PR 생성

Freegrow 템플릿에 맞는 PR을 생성합니다. Development Log는 커밋 히스토리를 분석하여 자동 생성됩니다.

**사용법:**
```bash
# 자동 생성 (브랜치/커밋 분석)
/freegrow-git:pr

# 제목 직접 지정
/freegrow-git:pr [FEAT] 사용자 인증 기능
```

**PR 템플릿:**
```markdown
## 📋 Summary
{커밋 히스토리 기반 자동 요약}

## 📣 Related Issue
- close #3

---

## ✨ 구현된 기능

### 1. 사용자 인증
- Firebase Auth 연동
- 로그인/로그아웃 기능

---

## 📝 Development Log

### 주요 변경 사항
- feat: 사용자 인증 기능 추가 (a1b2c3d)
- fix: 로그인 버그 수정 (d4e5f6g)

### 변경된 파일
- `src/auth/login.dart` - 로그인 로직 구현
- `src/models/user.dart` - 사용자 모델 추가

### 작업 요약
Firebase Auth를 활용한 사용자 인증 기능을 구현했습니다...
```

**자동 기능:**
- 브랜치에서 이슈 번호 추출
- 커밋 히스토리 분석하여 타입 결정
- Development Log 자동 정리 (커밋 목록, 변경 파일, 작업 요약)
- 푸시 안된 커밋 자동 푸시
- Assignee 자동 설정 (`MMMIIIN`)

---

## 컨벤션 규칙

### 브랜치 명명
```
feature/이슈번호-설명
```
예: `feature/3-user-authentication`

### 커밋 메시지
```
<type>: <description> (#이슈번호)
```
예: `feat: 사용자 인증 기능 추가 (#3)`

### 이슈/PR 제목
```
[타입] 제목
```
예: `[FEAT] 사용자 인증 기능`

---

## 예시 시나리오

### 새 기능 개발

```bash
# 1. 작업 시작
/freegrow-git:start [FEAT] 사용자 프로필 페이지

# → 이슈 #5 생성
# → feature/5-user-profile 브랜치 생성 및 체크아웃

# 2. 코드 작업 후 커밋
/freegrow-git:com feat: 프로필 UI 구현
/freegrow-git:com feat: 프로필 수정 기능 추가

# 3. PR 생성
/freegrow-git:pr

# → PR 자동 생성 (Development Log 포함)
```

### 버그 수정

```bash
# 1. 작업 시작
/freegrow-git:start [FIX] 로그인 실패 버그

# 2. 수정 후 커밋
/freegrow-git:com fix: 토큰 만료 처리 추가

# 3. PR 생성
/freegrow-git:pr
```

---

## 주의사항

- **기본 브랜치**: `develop` (main 아님)
- **이슈 번호 필수**: 모든 커밋에 이슈 번호 포함
- **Assignee**: 자동으로 `MMMIIIN` 설정
- **PR 로그**: `.claude/pr-logger/` 파일 자동 포함

---

## 관련 링크

- [Freegrow Git 컨벤션](../../CLAUDE.md)
- [범용 Git 워크플로우 (git-pr-workflows)](../git-pr-workflows/)
