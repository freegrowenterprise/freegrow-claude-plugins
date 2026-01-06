---
description: Freegrow 마켓플레이스 초기 설정 - 팀 설정 및 권장 플러그인 구성
---

# Freegrow Plugin Setup

Freegrow Enterprise 마켓플레이스 초기 설정을 안내합니다.

## 1단계: 마켓플레이스 등록

```
/plugin marketplace add freegrow-enterprise/freegrow-agents
```

## 2단계: 팀 설정 (.claude/settings.json)

프로젝트 루트에 `.claude/settings.json` 파일을 생성하여 팀 전체가 동일한 마켓플레이스를 사용하도록 설정합니다:

```json
{
  "extraKnownMarketplaces": [
    "freegrow-enterprise/freegrow-agents"
  ]
}
```

## 3단계: 역할별 플러그인 설치

### 백엔드 개발팀
```
/plugin install backend-development
/plugin install code-review-ai
/plugin install git-pr-workflows
/plugin install code-documentation
```

### 프론트엔드/모바일 개발팀
```
/plugin install frontend-mobile-development
/plugin install multi-platform-apps
/plugin install git-pr-workflows
/plugin install code-review-ai
```

### DevOps/인프라팀
```
/plugin install cloud-infrastructure
/plugin install git-pr-workflows
/plugin install code-documentation
```

### 임베디드/IoT 개발팀
```
/plugin install embedded-development
/plugin install arm-cortex-microcontrollers
/plugin install git-pr-workflows
```

### 풀스택 개발자
```
/plugin install backend-development
/plugin install frontend-mobile-development
/plugin install cloud-infrastructure
/plugin install git-pr-workflows
/plugin install code-review-ai
```

## 4단계: 설치 확인

```
/plugin list
```

## 로컬 개발/테스트

마켓플레이스를 로컬에서 테스트하려면:

```bash
# 단일 플러그인 테스트
claude --plugin-dir ./plugins/<plugin-name>

# 여러 플러그인 동시 테스트
claude --plugin-dir ./plugins/backend-development --plugin-dir ./plugins/cloud-infrastructure
```

## Arguments

$ARGUMENTS

역할이나 팀 이름을 지정하면 해당 팀에 맞는 설정 가이드를 제공합니다.
