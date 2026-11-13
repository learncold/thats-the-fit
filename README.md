# 대츠더핏 · THAT’S THE FIT

의류 실측과 핏 키워드 기반 데님 검색 서비스 프로젝트다

## 기술 스택

| 항목 | 설정 |
| --- | --- |
| Java | 21 |
| Spring Boot | 4.1.2 |
| Maven | 3.10.0 |

## 빌드·테스트

Docker를 실행한 상태에서 테스트한다. `contextLoads()`는 Testcontainers로 임시 PostgreSQL을 시작한다.

```powershell
# Windows PowerShell
.\mvnw.cmd clean verify
```

```sh
# Linux / macOS
sh ./mvnw clean verify
```

## 로컬 실행

Docker를 시작한 뒤 같은 터미널에서 DB 비밀번호를 입력하고, Compose로 PostgreSQL을 실행한다. 처음 만든 DB 볼륨을 재사용할 때는 초기화에 사용한 비밀번호를 입력한다.

```powershell
$dbPassword = Read-Host '로컬 PostgreSQL 비밀번호' -AsSecureString
$env:SPRING_DATASOURCE_PASSWORD = [pscredential]::new('thats_the_fit', $dbPassword).GetNetworkCredential().Password
docker compose up -d --wait db
.\mvnw.cmd spring-boot:run '-Dspring-boot.run.profiles=local'
```

```bash
read -rs -p '로컬 PostgreSQL 비밀번호: ' SPRING_DATASOURCE_PASSWORD
echo
export SPRING_DATASOURCE_PASSWORD
docker compose up -d --wait db
sh ./mvnw spring-boot:run -Dspring-boot.run.profiles=local
```

`local` 프로필은 `127.0.0.1:5432`의 `thats_the_fit` DB에 연결한다. 빌드한 JAR도 같은 환경변수와 프로필로 실행할 수 있다.

```sh
java -jar target/thats-the-fit-0.0.1-SNAPSHOT.jar --spring.profiles.active=local
```

DB를 멈추려면 같은 터미널에서 `docker compose stop db`를 실행한다. 데이터는 이름 있는 볼륨에 보관된다.

## 환경별 DB 연결

| 환경 | 연결 방식 |
| --- | --- |
| 로컬 | `local` 프로필의 `application-local.properties`가 `127.0.0.1:5432`의 `thats_the_fit` DB에 연결한다. Compose와 Spring Boot는 실행 터미널에서 `SPRING_DATASOURCE_PASSWORD`를 받는다. |
| 테스트 | Testcontainers가 임시 PostgreSQL을 실행하고 `@ServiceConnection`으로 테스트에 접속 정보를 전달한다. 로컬 DB와 별도로 실행된다. |
| 운영 | 배포 환경에서 아래 환경변수를 Spring Boot 프로세스에 주입한다. 로컬 프로필은 로컬 실행에 사용한다. |

운영에서 Spring Boot와 PostgreSQL을 같은 Docker Compose 네트워크에 두고 DB 서비스 이름을 `db`로 지정할 경우의 예시다.

| 환경변수 | 예시 값 | 용도 |
| --- | --- | --- |
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://db:5432/thats_the_fit` | 운영 DB의 JDBC 주소 |
| `SPRING_DATASOURCE_USERNAME` | `thats_the_fit` | 운영 DB 사용자명 |
| `SPRING_DATASOURCE_PASSWORD` | `<운영 DB 비밀번호>` | 배포 환경에서 주입하는 실제 비밀번호 |

실제 비밀번호와 토큰은 실행 환경에서 주입하고 코드나 Git에 기록하지 않는다.
