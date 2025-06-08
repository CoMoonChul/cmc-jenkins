# **배포 흐름**

---

전체 흐름도

```
GitHub (main)
   ↓
Jenkins Pipeline
   ↓
Gradle Build (.jar)
   ↓
Harbor 서버 전송 (Dockerfile + JAR)
   ↓
Docker 이미지 빌드 및 Harbor에 Push
   ↓
.env 파일 배포 서버 복사
   ↓
기존 컨테이너 제거
   ↓
신규 컨테이너 실행 (Harbor 이미지 기반)
```

---

## 배포 단계별 설명

### 1. Checkout Code

- GitHub에서 `main` 브랜치의 소스를 Jenkins 워크스페이스로 가져옵니다.
- `REPO_URL`, `BRANCH_NAME` 환경변수 사용

### 2. 환경설정 파일 로드

- `application-prod.yaml`을 읽어 YAML 데이터를 로드
- 현재는 변수에 적용하지 않음

### 3. Gradle 빌드 & OpenAPI 코드 생성

- `openApiGenerateAll` 태스크로 OpenAPI 기반 서버 코드 생성
- `bootJar`로 Spring Boot JAR 파일 생성

### 4. Harbor 서버로 JAR & Dockerfile 전송

- SCP를 통해 빌드된 `app.jar`과 `Dockerfile`을 Harbor 서버로 전송
- 대상 디렉토리: `${BUILD_DIR}`

### 5. Harbor 서버에서 Docker 이미지 빌드 & Push

- Harbor 서버에서 SSH  명령으로 이미지 빌드
- Harbor Registry에 login 후 `docker push`

### 6. OpenAPI Generator 결과를 AWS S3 업로드

- `${OAG_DIR}` 경로의 Swagger 생성 파일을 S3에 업로드
- AWS 자격증명: `aws-s3-deploy`

### 7. 배포 서버에 `.env` 파일 복사

- PEM 키를 이용해 `.env`가 배포 서버에 안전히 복사
- 복사 경로: `/home/${DEPLOY_USER}/.env`

### 8. 기존 컨테이너 중지 & 삭제

- 기존 실행 중인 컨테이너(`backend`)를 중지/삭제

### 9. 신규 컨테이너 실행

- Harbor에서 최신 이미지 Pull
- `.env` 파일을 `-env-file`로 적용하여 실행
- 네트워크: `cmc-net`

---

## Jenkins 환경변수

| 변수명 | 설명 |
| --- | --- |
| `HARBOR_REGISTRY` | 내부 Harbor Registry 주소 |
| `OUT_HARBOR_REGISTRY` | 외부 접근용 Harbor Registry |
| `HARBOR_ID`, `HARBOR_PW` | Harbor 로그인 자격 |
| `sshUser`, `sshPassword`, `sshHost`, `sshPort` | Harbor 서버 SSH 접근 |
| `DEPLOY_HOST`, `DEPLOY_USER`, `PEM_KEY` | 배포 서버 SSH (PEM) |
| `env_file` | `.env` 파일 경로 |
| `S3_BUCKET`, `AWS_REGION`, `OAG_DIR` | S3 업로드 가이드 |
