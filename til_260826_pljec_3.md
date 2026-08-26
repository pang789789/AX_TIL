# 3차 프로젝트 개발 및 진행 일지 (4조)

## 1. 프로젝트 개요

* **프로젝트 회차**: 3차 프로젝트[cite: 2]
* **조 편성**: 4조[cite: 2]
* **담당 주제**: **주제 4 — 2차 프로젝트 결과(4조 개발) 고도화 및 1·2·3번 프로젝트 통합**[cite: 2]
* **내 담당 역할**: **담당자 6 — Project 3 통합 + DevOps (Secondary Reviewer: 담당자 5)**[cite: 2]
* **핵심 방향성**: 
  * 기존 AX Evaluation Console의 Core 평가 파이프라인 안정성 유지 및 보강[cite: 1, 2]
  * 타 조(Project 1·2·3) 기능과의 통합 인터페이스 구축 및 공통 UX 제공[cite: 1, 2]
  * 튜터 피드백 기반 확장 기능(핸디캡, 퀴즈, 가중치 비율 설정 등) 도입[cite: 1]
  * Windows 운영 서버 기반 CI/CD 파이프라인 및 로그/배포 체계 구축[cite: 2]

---

## 2. 전체 프로젝트 주제 목록 (튜터 제공)

1. **게임식 학습 프로그램**: 과목별 사전 레벨 테스트, 목표 설정(1~100등급), 모바일 최적화, 게임 스타일 레벨업 학습, ADSP/SQLD 문제은행[cite: 2]
2. **LMS 과제 및 평가**: 강의안, 학습 계획/진행, 과제 제출, 피드백/평가 체계화 (참고: Moodle)[cite: 2]
3. **Idea Developer**: Draft 아이디어를 구체적 PRD로 발전시키는 시스템 (마인드맵 스타일)[cite: 2]
4. **2차 프로젝트 결과(4조 개발) 고도화 및 1·2·3번 통합 (4조 담당)**: 학생 및 튜터 대상 실사용성 개선 및 타 조 산출물 통합[cite: 2]

---

## 3. 4조 PRD (Product Requirements Document) 요약

* **문서명**: AX Evaluation Console — 4조 Core 시스템 고도화[cite: 1]
* **버전**: v0.4 (Draft) / **작성일**: 2026-08-26[cite: 1]

### 3.1 문제 정의 (Problem Statement)
* **점수 계산의 투명성·추적성 부족**: 개별 평가 제출이 특정 학생의 점수에 영향을 미친 과정을 직관적으로 추적 불가[cite: 1]
* **핵심/확장 기능 경계 모호**: 유지해야 할 기존 기능과 신규 확장 기능의 경계 미비로 회귀 위험 존재[cite: 1]
* **상위 팀 고착화**: 시드 계산 시 우승 이력 보정이 없어 유사한 상위 팀 조합 지속 반복[cite: 1]
* **평가 반영 비율 유연성 부족**: 학생/튜터 평가 및 팀/개인 평가의 반영 비율을 독립적으로 설정 불가[cite: 1]

### 3.2 핵심 기능 요구사항 (Functional Requirements)
* **Core 보강 및 추적성 (Traceability)**
  * **FR-TRACE-001~004**: 채점 실행 시 평가 제출(`ReviewSubmission`) 및 퀴즈, 핸디캡 내역을 별도 기록(`ScoreInputTrace`)하여 튜터가 익명성을 유지한 채 드릴다운으로 점수 근거를 조회 가능하게 보강[cite: 1]
* **확장 기능 A: 우승 팀 핸디캡 (Handicap)**
  * **FR-EXT1-001~004**: 직전 회차 1위 팀 구성원 전원에게 감점 계수를 부여하여 다음 회차 시드 계산에만 반영 (원 평가 점수 및 공개 순위 미영향)[cite: 1]
* **확장 기능 B: 퀴즈 반영 (Quizzes)**
  * **FR-EXT2-001~006**: 회차별 퀴즈 응시 결과를 별도 점수화하여 최종점수 산식에 독립된 가중치로 반영 (`quizzes` 전용 App 분리)[cite: 1]
* **확장 기능 C: Project 1·2·3 통합 (Integration)**
  * **FR-EXT3-001~006**: Core의 User/Student/Team 공용 Service 참조를 의무화하여 회원/팀 데이터 중복 방지, 공통 디자인 토큰 적용 및 "오늘 할 일" / 통합 대시보드 구축[cite: 1]
* **평가 반영 비율 설정 개선 (Weight Configuration)**
  * **FR-WEIGHT-001~006**: 학생 평가와 튜터 평가의 최종 반영 비율 분리 설정, 각 평가 내부의 개인/팀 비율 독립 설정 및 계층별 가중치 검증 기능 추가[cite: 1]

---

## 4. 개인 담당 업무 및 상세 책임 (담당자 6)

### 4.1 담당 영역 및 역할
* **Primary 역할**: Project 3(Idea Developer) 통합 + DevOps[cite: 2]
* **Secondary Reviewer**: 담당자 5 (Project 2 통합 + QA)[cite: 2]
* **Secondary Review 담당**: 담당자 5의 작업물 검토 및 코드 리뷰 지원[cite: 2]

### 4.2 주요 책임 및 실행 과제
* **Project 3 (Idea Developer) 인수 및 Core 통합**[cite: 2]
  * Project 3 산출물 인수 체크리스트 작성 (PRD, ERD, Source, Env 등)[cite: 2]
  * Core 테이블과 중복되는 기능/테이블 Mapping표 작성[cite: 2]
  * DB 변경 요청 시 담당자 3(DB)과 협의하여 Migration 진행[cite: 2]
  * 공통 `User/Student/Team` 식별자를 사용하도록 Project 3 App 구현[cite: 2]
  * Project 3 전용 Unit 및 Integration Test 작성[cite: 2]
* **DevOps & CI/CD 구축**[cite: 2]
  * GitHub Actions 기반 CI 구축 (`check`, `lint`, `test`, `migration check` 자동화)[cite: 2]
  * Repository & 개발환경 표준화 (담당자 1과 협업)[cite: 2]
* **Windows 운영 서버 관리 및 배포**[cite: 2]
  * Windows 서버 사양, 계정, 고정 IP, Port, 방화벽 설정 협의[cite: 2]
  * 배포 방식 확정 (Docker Compose 또는 PowerShell 운영 Script)[cite: 2]
  * PC 재부팅 후 서비스 자동 시작 및 자동 복구 설정[cite: 2]
  * Application, Deploy, Backup 관련 로그 위치 확정 및 모니터링[cite: 2]
* **Release & Rollback Drill**[cite: 2]
  * 담당자 1, 3과 함께 Release 배포 1회 수행 및 Rollback Drill 실행[cite: 2]
  * `RUNBOOK.md`, `DEPLOYMENT.md` 등 운영 문서 작성[cite: 2]

---

## 5. 팀 전체 역할 분담표 (6인)

| 인원 | Primary 역할 | 주요 책임 | Secondary Reviewer |
| --- | --- | --- | --- |
| **담당자 1** | 통합 리드 / Architecture / Repository | 전체 구조·결정·Merge·Release 조정 | 담당자 5 |
| **담당자 2** | Accounts / Auth / Common Data | 회원가입·로그인·Role·공통 회원/학생 모델·공통 Service | 담당자 3 |
| **담당자 3** | DB / ERD / Migration / Seed | ERD·Migration·Backup·Restore·Data Quality | 담당자 2 |
| **담당자 4** | Project 1 통합 + UX | Project 1 인수·Core 연결·학생/튜터 UX 통일 | 담당자 1 |
| **담당자 5** | Project 2 통합 + QA | Project 2 인수·Core 연결·Integration/E2E Test 주도 | 담당자 6 |
| **담당자 6** | Project 3 통합 + DevOps | Project 3 인수·Core 연결·CI/CD·Windows 서버·로그 | 담당자 5 |

[cite: 2]