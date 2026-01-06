# Freegrow Enterprise Claude Code Plugins

Freegrow 내부 개발자용 Claude Code 플러그인 마켓플레이스입니다.

## 설치

### 마켓플레이스 등록

```bash
claude marketplace add https://github.com/freegrow-enterprise/freegrow-agents
```

### 개별 플러그인 설치

```bash
claude plugin install <plugin-name>@freegrow-agents
```

## 플러그인 출처

| 출처 | 플러그인 |
|------|----------|
| Freegrow 자체 개발 | `embedded-development` |
| [wshobson/agents](https://github.com/wshobson/agents) 기반 + 확장 | `multi-platform-apps` |
| [wshobson/agents](https://github.com/wshobson/agents) 포크 | 나머지 8개 플러그인 |

## 플러그인 목록

### 1. arm-cortex-microcontrollers

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**ARM Cortex-M 마이크로컨트롤러 펌웨어/드라이버 개발 전문**

| 구분 | 내용 |
|------|------|
| 지원 플랫폼 | Teensy 4.x, STM32 (F4/F7/H7), nRF52, SAMD |
| 주요 기능 | HAL/베어메탈 드라이버, 인터럽트, DMA, 캐시 동기화, 메모리 배리어 |
| 포함 에이전트 | `arm-cortex-expert` |

**추천 대상**
- 펌웨어 엔지니어
- 임베디드 C/C++ 개발자
- MCU 드라이버 개발자
- 하드웨어 인터페이스 담당자

---

### 2. backend-development

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**백엔드 시스템 아키텍처 및 분산 시스템 개발 전문**

| 구분 | 내용 |
|------|------|
| API 설계 | REST, GraphQL, gRPC, WebSocket |
| 아키텍처 패턴 | Event Sourcing, CQRS, Saga, Clean Architecture, DDD |
| 워크플로우 | Temporal Python SDK |
| 개발 방법론 | TDD, BDD |
| 포함 에이전트 | `backend-architect`, `event-sourcing-architect`, `graphql-architect`, `tdd-orchestrator`, `temporal-python-pro` |
| Skills | API 설계 원칙, 마이크로서비스 패턴, CQRS 구현, Saga 오케스트레이션, Temporal 테스팅 |

**추천 대상**
- 백엔드 개발자
- 시스템 아키텍트
- API 설계 담당자
- 분산 시스템 개발자
- Temporal 워크플로우 개발자

---

### 3. cloud-infrastructure

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**클라우드 인프라 및 DevOps 전문**

| 구분 | 내용 |
|------|------|
| 클라우드 | AWS, Azure, GCP, 멀티클라우드, 하이브리드 |
| 컨테이너 | Kubernetes (EKS/AKS/GKE), Docker |
| Service Mesh | Istio, Linkerd |
| IaC | Terraform, OpenTofu, CDK, Pulumi |
| CI/CD | ArgoCD, Flux, GitHub Actions, GitLab CI |
| 포함 에이전트 | `cloud-architect`, `kubernetes-architect`, `terraform-specialist`, `deployment-engineer`, `network-engineer`, `hybrid-cloud-architect`, `service-mesh-expert` |
| Skills | Terraform 모듈, 비용 최적화, mTLS, Service Mesh 옵저버빌리티, 하이브리드 네트워킹 |

**추천 대상**
- DevOps 엔지니어
- SRE (Site Reliability Engineer)
- 클라우드 아키텍트
- 인프라 엔지니어
- 플랫폼 엔지니어

---

### 4. code-documentation

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**기술 문서화 및 코드 분석 전문**

| 구분 | 내용 |
|------|------|
| 문서 유형 | API 문서, 아키텍처 가이드, 튜토리얼, 온보딩 문서 |
| 분석 기능 | AI 기반 코드 분석, 복잡도 분석, 패턴 식별 |
| 포함 에이전트 | `code-reviewer`, `docs-architect`, `tutorial-engineer` |
| Commands | `code-explain`, `doc-generate` |

**추천 대상**
- 테크니컬 라이터
- 시니어 개발자 (문서화 담당)
- 팀 리드
- 온보딩 담당자
- 오픈소스 메인테이너

---

### 5. code-refactoring

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**코드 품질 개선 및 레거시 현대화 전문**

| 구분 | 내용 |
|------|------|
| 리팩토링 | 클린 코드, SOLID 원칙, 디자인 패턴 적용 |
| 레거시 | 기술 부채 분석, 모더나이제이션 전략 |
| 컨텍스트 | 장기 프로젝트 컨텍스트 복원 |
| 포함 에이전트 | `code-reviewer`, `legacy-modernizer` |
| Commands | `refactor-clean`, `tech-debt`, `context-restore` |

**추천 대상**
- 시니어 개발자
- 리팩토링 전담 개발자
- 레거시 시스템 담당자
- 기술 부채 해결 담당자

---

### 6. code-review-ai

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**AI 기반 코드 리뷰 및 품질 분석 전문**

| 구분 | 내용 |
|------|------|
| 분석 영역 | 보안 취약점, 성능 최적화, 아키텍처 패턴, 테스트 커버리지 |
| 도구 연동 | SonarQube, CodeQL, Semgrep, Snyk |
| 자동화 | PR 자동 분석, CI/CD 파이프라인 연동 |
| 포함 에이전트 | `architect-review` |
| Commands | `ai-review` |

**추천 대상**
- 코드 리뷰어
- 팀 리드
- QA 엔지니어
- 보안 담당자
- 품질 관리 담당자

---

### 7. embedded-development

> 출처: **Freegrow 자체 개발**

**임베디드 시스템 및 IoT 개발 전문**

| 구분 | 내용 |
|------|------|
| RTOS | FreeRTOS, Zephyr, ThreadX |
| IoT 플랫폼 | ESP32, nRF52 (BLE) |
| 임베디드 Linux | 드라이버, 빌드 시스템 |
| 프로토콜 | CAN, Modbus, I2C, SPI, UART, MQTT, CoAP |
| 저전력 설계 | 슬립 모드, 전력 도메인, 배터리 최적화 |
| 포함 에이전트 | `embedded-architect`, `rtos-expert`, `esp32-iot-expert`, `nrf-ble-expert`, `embedded-linux-expert` |
| Skills | RTOS 패턴, BLE 개발, 저전력 설계, 임베디드 빌드 시스템, 통신 프로토콜 |

**추천 대상**
- 임베디드 개발자
- IoT 개발자
- RTOS 개발자
- BLE 애플리케이션 개발자
- 하드웨어-소프트웨어 통합 담당자

---

### 8. frontend-mobile-development

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**프론트엔드 및 React Native 모바일 개발 전문**

| 구분 | 내용 |
|------|------|
| 프레임워크 | React, Next.js 14+ (App Router), React Native |
| 스타일링 | Tailwind CSS, 디자인 시스템 |
| 상태 관리 | Redux Toolkit, Zustand, Jotai, React Query |
| 포함 에이전트 | `frontend-developer`, `mobile-developer` |
| Skills | Next.js App Router 패턴, React 상태 관리, React Native 아키텍처, Tailwind 디자인 시스템 |
| Commands | `component-scaffold` |

**추천 대상**
- 프론트엔드 개발자
- React Native 개발자
- UI/UX 구현 담당자
- 웹 애플리케이션 개발자

---

### 9. git-pr-workflows

> 출처: [wshobson/agents](https://github.com/wshobson/agents)

**Git 워크플로우 및 협업 프로세스 전문**

| 구분 | 내용 |
|------|------|
| Git | Conventional Commits, 브랜치 전략 (trunk-based, feature-branch) |
| PR | PR 최적화, 자동 리뷰, 템플릿 |
| 온보딩 | 신규 팀원 온보딩 프로세스 |
| 포함 에이전트 | `code-reviewer` |
| Commands | `git-workflow`, `pr-enhance`, `onboard` |

**추천 대상**
- 모든 개발자 (필수 권장)
- 팀 리드
- 프로젝트 매니저
- 신규 입사자 온보딩 담당자

---

### 10. multi-platform-apps

> 출처: [wshobson/agents](https://github.com/wshobson/agents) 기반 + **Freegrow 확장**

**크로스플랫폼 앱 개발 전문 (Flutter 중심)**

| 구분 | 내용 |
|------|------|
| Flutter | Clean Architecture, 상태 관리 (Riverpod, BLoC, GetX) |
| 네이티브 연동 | Platform Channels, Method Channels |
| 테스팅 | Unit, Widget, Integration, Golden 테스트 |
| iOS | Swift/SwiftUI, UIKit |
| 포함 에이전트 | `flutter-expert`, `ios-developer`, `mobile-developer`, `frontend-developer`, `backend-architect`, `ui-ux-designer` |
| Skills | Flutter 아키텍처, 상태 관리, 플랫폼 채널, 테스팅 패턴 |
| Commands | `multi-platform` |

**추천 대상**
- Flutter 개발자
- 크로스플랫폼 앱 개발자
- iOS/Android 네이티브 개발자 (Flutter 전환)
- 모바일 앱 아키텍트

---

## 요구사항

- Claude Code 1.0.33+

## 로컬 테스트

```bash
claude --plugin-dir ./plugins/<plugin-name>
```

## Credits

- 기반 플러그인: [wshobson/agents](https://github.com/wshobson/agents)
- Freegrow Enterprise 확장 및 커스터마이징

## 라이선스

PROPRIETARY - Freegrow Enterprise 내부 사용 전용
