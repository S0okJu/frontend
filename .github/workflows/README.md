# GitHub Actions 배포 워크플로우

master 브랜치에 머지되면 자동으로 NHN Cloud Object Storage에 배포하는 GitHub Actions 워크플로우입니다.

## 워크플로우 목록

| 워크플로우 | 트리거 | 용도 |
|------------|--------|------|
| **Deploy to NHN Cloud Object Storage** (`deploy-to-object-storage.yml`) | push to main, 수동 | 일상 배포 (선택 리전 OBS 업로드) |
| **OBS Failover Recovery** (`failover-recovery.yml`) | 수동만 | 단일 리전 OBS 장애 시 보조 리전 배포 + CDN 원본 전환 |

## 배포 워크플로우 선택 (일상 배포용)

두 가지 방식 중 하나를 선택하여 사용하세요:

### 1. Swift API 방식 (deploy-to-object-storage.yml) ⭐ 권장
- **NHN Cloud 네이티브 Object Storage API** 사용 (Identity 토큰 + Swift PUT)
- S3 서명(SignatureDoesNotMatch) 이슈 없음
- Identity 인증만 있으면 됨 (S3 API 자격 증명 불필요)

### 2. S3 API 방식 (deploy-to-object-storage-s3.yml)
- AWS S3 호환 API + AWS CLI
- NHN S3 자격 증명 필요. 서명/리전 설정에 따라 오류가 날 수 있음

## 사용하지 않는 워크플로우 비활성화

하나만 사용할 경우, 사용하지 않는 워크플로우 파일 이름을 변경하세요:

```bash
# Swift API 방식만 사용하는 경우 (권장)
mv .github/workflows/deploy-to-object-storage-s3.yml .github/workflows/deploy-to-object-storage-s3.yml.disabled

# S3 API 방식만 사용하는 경우
mv .github/workflows/deploy-to-object-storage.yml .github/workflows/deploy-to-object-storage.yml.disabled
```

## GitHub Secrets 설정

**Repository secrets**를 사용합니다.  
GitHub Repository Settings > **Secrets and variables** > **Actions** > **Repository secrets**에서 다음 Secrets를 추가하세요. (Environment secret이 아닌 Repository secret입니다.)

### Swift API 방식 사용 시 (권장)

아래 **4개만** 설정하면 됩니다.

| Secret 이름 | 설명 | 예시 |
|------------|------|------|
| `NHN_AUTH_URL` | NHN Cloud Identity 인증 URL | `https://api-identity-infrastructure.nhncloudservice.com/v2.0` |
| `NHN_TENANT_ID` | NHN Cloud Tenant ID (프로젝트 ID) | `1234567890abcdef` |
| `NHN_USERNAME` | NHN Cloud API 사용자(이메일 등) | `user@example.com` |
| `NHN_API_PASSWORD` | NHN Cloud API Password | `your-api-password` |

**API가 프록시가 아니라 별도 URL일 때** (선택): `API_BASE_URL` Secret에 백엔드 주소를 넣으면 빌드 시 적용됩니다.  
예: `https://api.example.com` (끝에 `/` 없이). 없으면 `/api`(프록시)로 빌드됩니다.

### S3 API 방식 사용 시

| Secret 이름 | 설명 | 예시 |
|------------|------|------|
| `NHN_S3_ACCESS_KEY` | NHN Cloud S3 API Access Key | `your-access-key` |
| `NHN_S3_SECRET_KEY` | NHN Cloud S3 API Secret Key | `your-secret-key` |

## NHN Cloud 자격 증명 발급 방법

### Swift API 자격 증명
1. NHN Cloud Console 로그인
2. **프로젝트 설정** > **API 보안 설정**
3. **User Access Key ID** 및 **Secret Access Key** 확인
4. Tenant ID는 프로젝트 설정에서 확인

참조: https://docs.nhncloud.com/ko/Compute/Instance/ko/api-guide/

### S3 API 자격 증명 ⭐

1. NHN Cloud Console 로그인
2. **Storage** > **Object Storage** 메뉴로 이동
3. **API 엔드포인트 설정** 탭
4. **S3 API 자격 증명** 발급
5. Access Key와 Secret Key 복사

참조: https://docs.nhncloud.com/ko/Storage/Object%20Storage/ko/s3-api-guide/#s3-api-s3-api-credential

## Object Storage 컨테이너 설정

### 1. 컨테이너 생성

```bash
# NHN Cloud Console에서 수행
# Storage > Object Storage > 컨테이너 생성
# - 컨테이너명: photo-frontend
# - 액세스 정책: 퍼블릭 읽기
```

### 2. 정적 웹사이트 설정

컨테이너를 정적 웹사이트로 설정:

1. Object Storage 컨테이너 선택
2. **컨테이너 설정** 클릭
3. **Static Website** 활성화
4. **Index Document**: `index.html`
5. **Error Document**: `index.html` (SPA 라우팅 지원)

### 3. CORS 설정 (선택사항)

API 호출이 필요한 경우 CORS 설정:

```json
[
  {
    "AllowedOrigins": ["*"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]
```

## 워크플로우 설정 변경

`.github/workflows/deploy-to-object-storage-s3.yml` (또는 Swift 버전) 파일에서 다음 값을 수정하세요:

```yaml
env:
  NODE_VERSION: '18'                    # Node.js 버전
  BUCKET_NAME: 'photo-frontend'         # Object Storage 컨테이너명 (변경 필요)
  VITE_API_BASE_URL: '/api'             # API URL (아래 설명 참고)
  AWS_REGION: 'kr1'                     # NHN Cloud 리전
```

### VITE_API_BASE_URL 상세

이 값은 **빌드 시점**에 프론트엔드 번들에 포함됩니다. 배포된 SPA가 API를 호출할 때 쓰는 **기준 URL**입니다.

- **지금처럼 같은 도메인에서 `/api`로 프록시**하는 구성이면 `'/api'` 그대로 두면 됩니다.
- **추후 API가 전혀 다른 URL**(다른 도메인·다른 서비스·다른 경로)을 가질 예정이면, 이 값을 **그 API의 기준 URL 전체**로 넣으세요. (예: `https://other-api.example.com/v1`, `https://api.another-service.com` 등) 그때는 백엔드 CORS에서 해당 오리진을 허용해야 합니다.

| 배포 구조 | 설정값 | 설명 |
|-----------|--------|------|
| **같은 도메인 + 프록시** | `'/api'` | 프론트와 API가 같은 도메인, `/api`로 프록시할 때. (현재 기본값) |
| **전혀 다른 API URL** | `'https://api.example.com'` 등 | API가 다른 도메인·다른 서비스일 때. 사용할 **그 URL 전체**를 넣음. CORS 허용 필요. |
| **로컬 개발** | (워크플로우에서 설정 안 함) | `src/config/api.js`는 개발 시 `http://localhost:8000` 기본값 사용. |

값을 바꾸면 **다음 빌드·배포부터** 적용됩니다.

## 사용 방법

### 자동 배포 (master 브랜치 머지 시)

```bash
# 개발 브랜치에서 작업
git checkout -b feature/new-feature
# ... 코드 수정 ...
git add .
git commit -m "Add new feature"
git push origin feature/new-feature

# GitHub에서 Pull Request 생성 및 master로 머지
# → 자동으로 배포 시작!
```

### 수동 배포

GitHub Repository > Actions > Deploy to NHN Cloud Object Storage > Run workflow

## 배포 프로세스

1. **코드 체크아웃**: master 브랜치 최신 코드 가져오기
2. **Node.js 설정**: Node.js 18 설치 및 npm 캐시 설정
3. **의존성 설치**: `npm ci` 실행
4. **빌드**: `npm run build` 실행 (dist/ 폴더 생성)
5. **빌드 검증**: dist/ 폴더 내용 확인
6. **자격 증명 설정**: NHN Cloud 자격 증명 구성
7. **업로드**: Object Storage에 파일 업로드
   - 정적 파일: 1년 캐시 (`max-age=31536000`)
   - HTML 파일: 5분 캐시 (`max-age=300`)
8. **배포 검증**: 업로드된 파일 목록 확인

## 캐시 전략

### 정적 파일 (JS, CSS, 이미지)
- **Cache-Control**: `public, max-age=31536000` (1년)
- Vite가 파일명에 해시를 포함하므로 안전하게 장기 캐시 가능

### HTML 파일
- **Cache-Control**: `public, max-age=300` (5분)
- 자주 변경되므로 짧은 캐시 시간 설정

## CDN 연동 (선택사항)

### 1. NHN Cloud CDN 생성

```bash
# NHN Cloud Console에서 수행
# CDN > CDN 서비스 생성
# - 원본 서버: Object Storage 컨테이너 URL
# - 도메인: photo.example.com (선택사항)
```

### 2. 캐시 무효화

배포 후 CDN 캐시를 무효화하려면 워크플로우에 추가:

```yaml
- name: Invalidate CDN cache
  run: |
    # NHN Cloud CDN API를 사용하여 캐시 무효화
    # 또는 수동으로 NHN Cloud Console에서 수행
```

## 접속 URL

### Object Storage 직접 접속

기본 리전(KR1) 기준:

```
https://kr1-api-object-storage.nhncloudservice.com/v1/AUTH_<TENANT_ID>/photo-frontend/index.html
```

다른 리전 사용 시 `kr1`을 해당 리전 코드(kr2, jp1 등)로 바꿉니다.

### CDN 접속 (CDN 설정 시)

```
https://your-cdn-domain.com/
```

## 장애 대응 (KR1 리전 장애 시)

NHN Cloud Object Storage는 **리전 단위**로 제공되며, **리전 간 복제는 단방향**입니다. 따라서 한 리전(OBS)에 장애가 나면:

1. **다른 리전 OBS로 트래픽을 전환**해야 하고  
2. **CDN의 원본 서버를 해당 리전 OBS URL로 변경**해야 합니다.  
3. 단방향 복제이므로, 장애 전에 보조 리전으로 복제된 데이터만 있다면 **최신 빌드가 아닐 수 있음** → 전환 시 **보조 리전에 현재 빌드물을 한 번 더 배포**하는 것이 안전합니다.

### 자동 복구 워크플로우 (Failover Recovery) ⭐

**OBS Failover Recovery (CDN 원본 전환)** 워크플로우(`failover-recovery.yml`)가 다음을 한 번에 수행합니다.

| 단계 | 설명 |
|------|------|
| 1. 빌드 | 현재 브랜치 기준으로 프론트엔드 빌드 |
| 2. OBS 배포 | 선택한 리전(kr1/kr2/jp1)의 Object Storage에 빌드물 업로드 (단방향 복제 갱신) |
| 3. CDN 원본 전환 | (선택) NHN CDN API로 해당 CDN 서비스의 원본을 위 리전 OBS URL로 변경 |

**실행 방법**

- GitHub **Actions** → **OBS Failover Recovery (CDN 원본 전환)** → **Run workflow**
- **failover_region**: 전환할 리전 선택 (`kr2` 또는 `jp1` = 장애 회피, `kr1` = 복구 후 원상 복귀)
- **update_cdn_origin**: `true`면 CDN 원본까지 자동 변경(아래 Secret 설정 시), `false`면 OBS 배포만 하고 CDN은 수동 변경

**CDN 자동 전환을 쓰려면** Repository Secrets에 다음을 추가하세요.

| Secret | 설명 |
|--------|------|
| `CDN_APP_KEY` | NHN Cloud 콘솔 우측 상단 **URL & Appkey** 에서 확인 |
| `CDN_SECRET_KEY` | 동일 메뉴에서 발급한 SecretKey (CDN API 인증용) |
| `CDN_DISTRIBUTION_ID` | CDN 서비스(배포) ID (CDN API로 조회 또는 콘솔에서 확인) |

이 세 가지가 없으면 워크플로우는 **OBS 배포만** 하고, CDN 원본 변경은 건너뛰며 경고만 출력합니다. 그때는 아래처럼 수동으로 CDN 원본을 바꾸면 됩니다.

### 사전 준비 (평상시)

1. **보조 리전에 컨테이너 미리 생성**
   - NHN Cloud Console에서 **KR2**(한국 평촌) 또는 **JP1**(일본 도쿄) 리전 선택
   - 동일한 이름의 컨테이너(`photo-frontend`) 생성, 퍼블릭 읽기, Static Website 설정
2. **Identity 자격 증명**
   - Failover Recovery는 기존 배포와 동일하게 **Swift API(Identity 토큰)** 를 사용합니다. `NHN_AUTH_URL`, `NHN_TENANT_ID`, `NHN_USERNAME`, `NHN_API_PASSWORD` 가 있으면 됩니다.
3. **(선택) CDN API 자격 증명**
   - CDN 원본 자동 전환을 쓰려면 `CDN_APP_KEY`, `CDN_SECRET_KEY`, `CDN_DISTRIBUTION_ID` 설정.

### 리전 전환 절차 (KR1 장애 발생 시)

1. **자동 복구 워크플로우 실행 (권장)**
   - Actions → **OBS Failover Recovery (CDN 원본 전환)** → Run workflow
   - **failover_region**: `kr2` 또는 `jp1` 선택
   - **update_cdn_origin**: `true` (CDN Secret 설정했을 때) 또는 `false` 후 수동 변경
2. **CDN 원본만 수동으로 바꾸는 경우**
   - NHN Cloud CDN 콘솔에서 해당 CDN 서비스의 **원본 서버**를 다음으로 변경:
     - KR2: `https://kr2-api-object-storage.nhncloudservice.com/v1/AUTH_<TENANT_ID>/photo-frontend/`
     - JP1: `https://jp1-api-object-storage.nhncloudservice.com/v1/AUTH_<TENANT_ID>/photo-frontend/`
   - 필요 시 캐시 무효화 수행.
3. **복구 후 (KR1 복구 시)**
   - Actions → **OBS Failover Recovery** → Run workflow → **failover_region**: `kr1` 선택 → 실행 (동일 워크플로우로 원상 복귀 가능).

### 요약

| 항목 | 내용 |
|------|------|
| 일반 배포 워크플로우 | 빌드 후 선택한 리전 OBS에 업로드 (CDN 원본 변경 없음) |
| **Failover Recovery** | 빌드 → 선택 리전 OBS 배포 → (선택) CDN 원본을 해당 리전 OBS로 변경 |
| OBS 복제 | 단방향 → 전환 시 보조 리전에 한 번 더 배포하는 것이 안전 |
| 리전 전환 방법 | Actions → **OBS Failover Recovery** → failover_region 선택 후 실행 |
| 사전 준비 | 보조 리전에 동일 컨테이너·설정 미리 생성, (선택) CDN API Secret 설정 |

## 문제 해결

### 워크플로우 실패 시

1. **GitHub Actions 로그 확인**
   - Repository > Actions > 실패한 워크플로우 클릭
   - 각 단계의 로그 확인

2. **자격 증명 확인**
   - GitHub Secrets가 올바르게 설정되었는지 확인
   - NHN Cloud Console에서 자격 증명 재발급

3. **컨테이너 권한 확인**
   - Object Storage 컨테이너가 퍼블릭 읽기로 설정되었는지 확인

4. **빌드 오류**
   - 로컬에서 `npm run build` 테스트
   - `package.json`의 의존성 확인

### 배포 후 사이트 접속 안 됨

1. **Object Storage URL 확인**
   - NHN Cloud Console에서 컨테이너 URL 확인
   - 브라우저에서 직접 접속 테스트

2. **정적 웹사이트 설정 확인**
   - Index Document가 `index.html`로 설정되었는지 확인
   - Error Document가 `index.html`로 설정되었는지 확인 (SPA)

3. **CORS 오류**
   - 백엔드 API에서 CORS 설정 확인
   - Object Storage CORS 설정 확인

## 로컬 테스트

배포 전 로컬에서 빌드를 테스트하세요:

```bash
# 빌드
npm run build

# 빌드 결과 미리보기
npm run preview

# 브라우저에서 http://localhost:4173 접속
```

## 보안 고려사항

1. **Secrets 관리**
   - GitHub Secrets에 민감한 정보 저장
   - 절대 코드에 직접 하드코딩하지 않기

2. **퍼블릭 읽기 권한**
   - Object Storage 컨테이너는 퍼블릭 읽기만 허용
   - 쓰기 권한은 GitHub Actions만 가짐

3. **브랜치 보호**
   - master 브랜치에 보호 규칙 설정
   - Pull Request 리뷰 필수로 설정

## 참고 자료

- [NHN Cloud Object Storage 문서](https://docs.nhncloud.com/ko/Storage/Object%20Storage/ko/overview/)
- [NHN Cloud S3 API 가이드](https://docs.nhncloud.com/ko/Storage/Object%20Storage/ko/s3-api-guide/)
- [GitHub Actions 문서](https://docs.github.com/en/actions)
- [AWS CLI S3 명령어](https://docs.aws.amazon.com/cli/latest/reference/s3/)
