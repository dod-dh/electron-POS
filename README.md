# electron-POS
- **개발 기간**
    - 2026.04 ~ 2026.05 (2026.06 ~ 테스트기간)
- **프로젝트 인원**
    - 개발팀 2명, 디자인팀
- **담당 역할 (전체 기획 및 설계, 로직 구현)**
    - Electron main/renderer 이중 프로세스 구조 설계 및 Context Isolation·IPC 채널 화이트리스트 기반 보안 아키텍처 구현
    - 서비스–IPC–API 3계층 분리 설계로 기능 확장 및 유지보수 용이성 확보
    - 2단계 결제 큐 시스템 구현 — VAN 단말 승인(1단계)과 백엔드 확정(2단계)을 분리하여 앱 재시작 시 중복 결제 없이 자동 복구
    - SQLite WAL 모드 + 마이그레이션 버전 관리 시스템 구축, 트랜잭션 기반 스키마 무결성 보장
    - 지수 백오프 재시도 정책(2→4→8초, 최대 3회) 및 Dead Letter Queue 도입으로 결제 유실 방지
    - KSCAT VAN 단말 프로토콜 통합 — EUC-KR 인코딩, 60바이트 전문 파싱, 재시도 가능/불가능 오류 분류 처리
    - PHP REST API 연동 (인증, 결제 확정, 취소, 주문 조회), Deep Link(ezchainpos://) 기반 SSO 흐름 구현
    - 복합 결제(카드+현금), 할부, 면세, 결제 취소 등 결제 시나리오 전반 구현
    - Winston 기반 날짜별 로그 파일 출력, IPC 핸들러 공통에러 로깅 래퍼 구현
- **사용 기술**
    - Electron 29, React 18, Vite, Zustand, SQLite (better-sqlite3), PHP, REST API, iconv-lite, Winston, electron-builder (NSIS)
- **프로젝트 개요**
    - 소매·유통 매장 전용 데스크톱 POS 애플리케이션.
    Electron + React 기반으로 결제 단말기(VAN) 연동, 오프라인 내성을 갖춘 결제 큐 처리, 재고·주문·쿠폰 관리 기능을 제공하며, Windows 전용 NSIS 인스톨러로 배포.
- **기술적 문제 & 해결**
    - 문제: 결제 승인 중 앱 강제 종료 시 이중 청구 위험 
    해결: 결제 payload를 DB에 먼저 저장 후 큐 처리 → 재시작 시 미완료 건 자동 복구, VAN 재호출 없이 백엔드 확정만 재시도
    - 문제: VAN 단말 오류 종류에 따라 재시도 여부가 달라야함
    해결: VanError 클래스에 retryable 플래그 도입, 재시도. 불가 오류는 즉시 Dead Letter로 이동해 불필요한 단말 재호출 차단
    - 문제: Renderer에서 Node.js API 직접 접근 시 보안 취약점
    해결: Context Isolation + 명시적 IPC 채널 화이트리스트(preload.js)로 Renderer의 Node 접근
    완전 차단
    - 문제: 복수 VAN사 지원 필요 (KSCAT·DAOU·KPN)
    해결: VAN Provider Factory 패턴으로 추상화, 설정값만 변경하면 단말사 교체 가능한 플러그인 구조 구현
    - 문제: electron-updater 라이브러리가 자사 배포 서버 구조와 맞지 않아 사용 불가
    해결: 자동 업데이트 모듈 직접 구현 — HTTP 스트림 다운로드(리다이렉트·타임아웃 처리), 진행률 IPC전송, 설치 완료 후 app.exit() 강제 종료로 구 버전 잔존 방지
    - 문제: 웹 관리자가 POS 앱에 토큰을 안전하게 전달할 방법 필요
    해결: pos://auth?token=xxx 커스텀 프로토콜 등록, 앱 실행 중이면 second-instance 이벤트로, 미실행이면 process.argv 파싱으로 동일한 딥링크 처리


----------------------------

## Project Duration

* Apr 2026 – May 2026 (Testing period: Jun 2026 – Present)

## Team Size

* 2 Software Engineers, Design Team

## Responsibilities (Project Planning, Architecture Design, and Core Development)

* Designed and implemented a secure Electron architecture based on a dual-process model (Main/Renderer), utilizing Context Isolation and IPC channel whitelisting.
* Established a three-layer architecture (Service → IPC → API) to improve maintainability, scalability, and feature extensibility.
* Developed a two-phase payment queue system that separates VAN terminal authorization (Phase 1) from backend confirmation (Phase 2), enabling automatic recovery after application restarts without duplicate charges.
* Built an SQLite WAL-based persistence layer with schema migration versioning and transaction-based integrity management.
* Implemented exponential backoff retry policies (2s → 4s → 8s, up to 3 attempts) and a Dead Letter Queue (DLQ) mechanism to prevent payment loss.
* Integrated KSCAT VAN terminal protocols, including EUC-KR encoding support, 60-byte message parsing, and classification of retryable vs. non-retryable errors.
* Connected to PHP REST APIs for authentication, payment confirmation/cancellation, and order retrieval, while implementing Deep Link (ezchainpos://) based Single Sign-On (SSO) workflows.
* Implemented comprehensive payment scenarios, including mixed payments (card + cash), installment transactions, tax-exempt sales, and payment cancellations.
* Developed a Winston-based logging framework with daily log rotation and a reusable IPC error-handling wrapper for centralized exception logging.

## Technology Stack

* Electron 29
* React 18
* Vite
* Zustand
* SQLite (better-sqlite3)
* PHP
* REST API
* iconv-lite
* Winston
* electron-builder (NSIS)

## Project Overview

A desktop Point-of-Sale (POS) application designed for retail and distribution businesses.

Built with Electron and React, the system provides VAN terminal integration, offline-resilient payment processing through a durable payment queue, and management features including inventory, orders, and coupons. The application is distributed through a Windows-based NSIS installer.

## Technical Challenges & Solutions

### Preventing Duplicate Charges After Unexpected Application Termination

**Problem:** If the application was forcibly terminated during payment processing, duplicate billing could occur.

**Solution:** Payment payloads were persisted to SQLite before queue execution. Upon restart, incomplete transactions were automatically recovered and only backend confirmation was retried, eliminating unnecessary VAN terminal reauthorization.

### Retry Logic Based on VAN Error Types

**Problem:** Different VAN terminal errors required different retry strategies.

**Solution:** Introduced a `VanError` class with a `retryable` flag. Retryable errors were processed through the retry queue, while non-retryable errors were immediately routed to the Dead Letter Queue to prevent unnecessary terminal requests.

### Securing Renderer Processes

**Problem:** Direct access to Node.js APIs from the Renderer process created security vulnerabilities.

**Solution:** Enforced Context Isolation and implemented an explicit IPC channel whitelist through `preload.js`, completely preventing unauthorized Node.js access from Renderer processes.

### Supporting Multiple VAN Providers

**Problem:** The POS system needed to support multiple VAN providers such as KSCAT, DAOU, and KPN.

**Solution:** Implemented a VAN Provider Factory pattern that abstracts terminal integrations, enabling provider replacement through configuration changes without modifying business logic.

### Custom Auto-Update System

**Problem:** The `electron-updater` library was incompatible with the company's deployment server architecture.

**Solution:** Built a custom auto-update module supporting HTTP stream downloads, redirect and timeout handling, IPC-based progress reporting, and forced application shutdown after installation to prevent legacy version conflicts.

### Secure Token Delivery from Web Admin to POS Application

**Problem:** A secure mechanism was required to transfer authentication tokens from the web administration system to the POS application.

**Solution:** Registered a custom protocol (`ezchainpos://auth?token=xxx`) and implemented unified deep-link handling through Electron's `second-instance` event when running, and `process.argv` parsing when launched via protocol.

