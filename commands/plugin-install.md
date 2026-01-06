---
description: Freegrow 플러그인 설치 - 개별 또는 전체 플러그인 설치
---

# Freegrow Plugin Install

Freegrow Enterprise 플러그인을 설치합니다.

## 사용 가능한 플러그인

| 플러그인 | 설명 | 추천 대상 |
|---------|------|----------|
| `arm-cortex-microcontrollers` | ARM Cortex-M MCU 개발 | 펌웨어 엔지니어 |
| `backend-development` | 백엔드 아키텍처, Event Sourcing, GraphQL | 백엔드 개발자 |
| `cloud-infrastructure` | AWS/Azure/GCP, Kubernetes, Terraform | DevOps/SRE |
| `code-documentation` | 기술 문서화, 튜토리얼 작성 | 테크니컬 라이터 |
| `code-refactoring` | 리팩토링, 레거시 모더나이제이션 | 시니어 개발자 |
| `code-review-ai` | AI 기반 코드 리뷰 | 코드 리뷰어, QA |
| `embedded-development` | RTOS, ESP32, nRF BLE, 임베디드 Linux | 임베디드/IoT 개발자 |
| `frontend-mobile-development` | React, Next.js, React Native | 프론트엔드 개발자 |
| `git-pr-workflows` | Git 워크플로우, PR 관리 | 모든 개발자 |
| `multi-platform-apps` | Flutter, iOS, 크로스플랫폼 | 모바일 앱 개발자 |

## 설치 방법

### 마켓플레이스 등록 (최초 1회)
```
/plugin marketplace add freegrowenterprise/freegrow-agents
```

### 개별 플러그인 설치
```
/plugin install <plugin-name>
```

### 역할별 추천 설치

**백엔드 개발자**
```
/plugin install backend-development
/plugin install code-review-ai
/plugin install git-pr-workflows
```

**프론트엔드/모바일 개발자**
```
/plugin install frontend-mobile-development
/plugin install multi-platform-apps
/plugin install git-pr-workflows
```

**DevOps/SRE**
```
/plugin install cloud-infrastructure
/plugin install git-pr-workflows
```

**임베디드/IoT 개발자**
```
/plugin install embedded-development
/plugin install arm-cortex-microcontrollers
```

**전체 설치**
```
/plugin install arm-cortex-microcontrollers
/plugin install backend-development
/plugin install cloud-infrastructure
/plugin install code-documentation
/plugin install code-refactoring
/plugin install code-review-ai
/plugin install embedded-development
/plugin install frontend-mobile-development
/plugin install git-pr-workflows
/plugin install multi-platform-apps
```

## Arguments

$ARGUMENTS

사용자가 특정 플러그인 이름이나 역할을 지정하면 해당 플러그인 설치 명령어를 안내합니다.
