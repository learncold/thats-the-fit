# 기술 스택

| 영역 | 기술 | 용도 |
| --- | --- | --- |
| 언어·빌드 | Java 21, Maven Wrapper | 백엔드 개발·빌드 |
| 백엔드 | Spring Boot, Spring MVC | REST API |
| 데이터베이스 | PostgreSQL | 상품·사이즈·실측·수집 이력 저장 |
| 데이터 접근 | Spring Data JPA, 검색 전용 SQL | 데이터 저장·조회·실측 검색 |
| 스키마 관리 | Flyway | DB 변경 이력 관리 |
| 수집 | Spring RestClient, Playwright | HTTP 수집·브라우저 세션 기반 수집 |
| 입력 검증 | Bean Validation, 도메인 검증 | 실측·검색 조건 검증 |
| 프런트엔드 | React, TypeScript, Vite | 실측 입력·검색·비교 화면 |
| 테스트 | JUnit, Spring Boot Test, Testcontainers | 검색 규칙·DB 연동 검증 |
| 관측 | Actuator, 구조화 로그 | API·수집 작업 상태 확인 |
| 배포 | AWS EC2, Linux, Docker, Docker Compose, HTTPS | 서비스 실행·배포 |
| CI/CD | GitHub Actions | 빌드·테스트·배포 자동화 |

※ Spring Boot 세부 버전은 구현 시작 시 결정하고, RDS 도입은 예산 확인 후 검토한다.
