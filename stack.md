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

## 프런트엔드와 API 제공 방식

- 운영에서는 React 빌드 파일을 Spring Boot 애플리케이션에 포함한다. Spring Boot가 화면과 `/api`를 같은 HTTPS 출처에서 제공한다.
- 로컬 개발에서는 Vite와 Spring Boot를 각각 실행한다. Vite의 `server.proxy`가 `/api` 요청을 로컬 Spring Boot로 전달한다.
- 프런트엔드는 두 환경에서 `/api/products`처럼 상대 경로로 호출한다. 브라우저에서 화면과 API를 동일 출처로 사용하므로 별도 CORS 허용 설정 없이 연결한다.

※ Spring Boot 세부 버전은 구현 시작 시 결정하고, RDS 도입은 예산 확인 후 검토한다.
