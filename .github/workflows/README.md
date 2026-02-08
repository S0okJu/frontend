# GitHub Actions 배포 워크플로우

master 브랜치에 머지되면 자동으로 NHN Cloud Object Storage에 배포하는 GitHub Actions 워크플로우입니다.

## 워크플로우 선택

두 가지 방식 중 하나를 선택하여 사용하세요:

### 1. Swift API 방식 (deploy-to-object-storage.yml)
- NHN Cloud의 Swift API 사용
- Python Swift Client 사용
- 더 세밀한 제어 가능

### 2. S3 API 방식 (deploy-to-object-storage-s3.yml) ⭐ 권장
- AWS S3 호환 API 사용
- AWS CLI 사용 (더 친숙함)
- 더 간단하고 빠름

**권장**: S3 API 방식을 사용하세요. 더 간단하고 AWS CLI에 익숙하다면 사용하기 쉽습니다.

## 사용하지 않는 워크플로우 비활성화

하나만 사용할 경우, 사용하지 않는 워크플로우 파일을 삭제하거나 이름을 변경하세요:

```bash
# Swift API 방식만 사용하는 경우
mv .github/workflows/deploy-to-object-storage-s3.yml .github/workflows/deploy-to-object-storage-s3.yml.disabled

# S3 API 방식만 사용하는 경우 (권장)
mv .github/workflows/deploy-to-object-storage.yml .github/workflows/deploy-to-object-storage.yml.disabled
```

## GitHub Secrets 설정

GitHub Repository Settings > Secrets and variables > Actions에서 다음 Secrets를 추가하세요.

### Swift API 방식 사용 시

| Secret 이름 | 설명 | 예시 |
|------------|------|------|
| `NHN_AUTH_URL` | NHN Cloud Identity 인증 URL | `https://api-identity-infrastructure.nhncloudservice.com/v2.0` |
| `NHN_TENANT_ID` | NHN Cloud Tenant ID (프로젝트 ID) | `1234567890abcdef` |
| `NHN_API_PASSWORD` | NHN Cloud API Password | `your-api-password` |
| `NHN_REGION` | NHN Cloud 리전 | `KR1` |

### S3 API 방식 사용 시 ⭐

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
  VITE_API_BASE_URL: '/api'             # API URL (환경에 맞게 변경)
  AWS_REGION: 'kr1'                     # NHN Cloud 리전
```

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

```
https://kr1-api-object-storage.nhncloudservice.com/v1/AUTH_<TENANT_ID>/photo-frontend/index.html
```

### CDN 접속 (CDN 설정 시)

```
https://your-cdn-domain.com/
```

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
