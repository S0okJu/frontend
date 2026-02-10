# GitHub Secrets 설정 가이드

GitHub Actions에서 NHN Cloud Object Storage에 배포하기 위한 Secrets 설정 방법입니다.

## 빠른 시작

### 1. NHN Cloud S3 API 자격 증명 발급 (권장)

1. [NHN Cloud Console](https://console.nhncloud.com/) 로그인
2. **Storage** > **Object Storage** 메뉴로 이동
3. **API 엔드포인트 설정** 탭 클릭
4. **S3 API 자격 증명** 섹션에서 **S3 API 자격 증명 발급** 클릭
5. **Access Key**와 **Secret Key** 복사 (한 번만 표시되므로 안전하게 보관)

### 2. GitHub Secrets 추가

1. GitHub Repository 페이지로 이동
2. **Settings** > **Secrets and variables** > **Actions** 클릭
3. **New repository secret** 클릭
4. 다음 Secrets 추가:

#### S3 API 방식 (권장)

| Name | Value | 설명 |
|------|-------|------|
| `NHN_S3_ACCESS_KEY` | `your-access-key` | NHN Cloud S3 API Access Key |
| `NHN_S3_SECRET_KEY` | `your-secret-key` | NHN Cloud S3 API Secret Key |

## 상세 설정 가이드

### S3 API 자격 증명 발급 (스크린샷 가이드)

#### Step 1: Object Storage 메뉴 이동

```
NHN Cloud Console
└── Storage
    └── Object Storage
        └── API 엔드포인트 설정 탭
```

#### Step 2: S3 API 자격 증명 발급

1. **S3 API 자격 증명** 섹션 찾기
2. **S3 API 자격 증명 발급** 버튼 클릭
3. 확인 대화상자에서 **확인** 클릭

⚠️ **중요**: Access Key와 Secret Key는 발급 시 한 번만 표시됩니다. 안전한 곳에 보관하세요.

#### Step 3: 자격 증명 복사

발급된 자격 증명을 복사합니다:

- **Access Key**: `AKIAIOSFODNN7EXAMPLE` (예시)
- **Secret Key**: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` (예시)

### GitHub Secrets 추가 방법

#### Step 1: Repository Settings 이동

```
GitHub Repository
└── Settings (상단 탭)
    └── Secrets and variables (왼쪽 메뉴)
        └── Actions
            └── New repository secret (버튼)
```

#### Step 2: Secret 추가

**NHN_S3_ACCESS_KEY 추가**:

1. **Name**: `NHN_S3_ACCESS_KEY`
2. **Secret**: NHN Cloud에서 복사한 Access Key 붙여넣기
3. **Add secret** 클릭

**NHN_S3_SECRET_KEY 추가**:

1. **Name**: `NHN_S3_SECRET_KEY`
2. **Secret**: NHN Cloud에서 복사한 Secret Key 붙여넣기
3. **Add secret** 클릭

#### Step 3: 추가 완료 확인

Secrets 목록에 다음 항목이 표시되어야 합니다:

- ✅ `NHN_S3_ACCESS_KEY`
- ✅ `NHN_S3_SECRET_KEY`

### Failover Recovery 시 CDN 원본 자동 전환 (선택)

**OBS Failover Recovery** 워크플로우에서 CDN 원본을 자동으로 바꾸려면 아래 CDN API용 Secret을 추가하세요. 없으면 워크플로우는 OBS 배포만 하고 CDN은 수동 변경해야 합니다.

| Secret 이름 | 설명 | 확인 방법 |
|-------------|------|-----------|
| `CDN_APP_KEY` | NHN Cloud CDN API AppKey | 콘솔 우측 상단 **URL & Appkey** 메뉴 |
| `CDN_SECRET_KEY` | CDN API 인증용 SecretKey | 동일 메뉴에서 발급 |
| `CDN_DISTRIBUTION_ID` | CDN 서비스(배포) ID | CDN 콘솔에서 해당 서비스 선택 후 URL 또는 API 조회로 확인 |

참고: [NHN Cloud CDN API v2.0 가이드](https://docs.nhncloud.com/ko/Contents%20Delivery/CDN/ko/api-guide-v2.0/)

## Object Storage 컨테이너 설정

### 1. 컨테이너 생성

1. NHN Cloud Console > **Storage** > **Object Storage**
2. **컨테이너 생성** 클릭
3. 설정:
   - **컨테이너명**: `photo-frontend`
   - **액세스 정책**: **퍼블릭 읽기** 선택
4. **생성** 클릭

### 2. 정적 웹사이트 설정

1. 생성한 컨테이너 선택
2. **컨테이너 설정** 버튼 클릭
3. **Static Website** 섹션:
   - **Static Website**: **활성화** 체크
   - **Index Document**: `index.html` 입력
   - **Error Document**: `index.html` 입력 (SPA 라우팅 지원)
4. **저장** 클릭

### 3. 컨테이너 URL 확인

컨테이너 설정에서 **Static Website URL** 확인:

```
https://kr1-api-object-storage.nhncloudservice.com/v1/AUTH_<TENANT_ID>/photo-frontend/index.html
```

이 URL이 배포 후 접속 URL입니다.

## 워크플로우 파일 수정

`.github/workflows/deploy-to-object-storage-s3.yml` 파일에서 컨테이너명 확인:

```yaml
env:
  BUCKET_NAME: 'photo-frontend'  # 생성한 컨테이너명과 일치해야 함
```

다른 컨테이너명을 사용한 경우 이 값을 수정하세요.

## 테스트

### 1. 로컬 테스트

```bash
# 빌드 테스트
npm run build

# 빌드 결과 확인
npm run preview
```

### 2. GitHub Actions 수동 실행

1. GitHub Repository > **Actions** 탭
2. **Deploy to NHN Cloud Object Storage (S3 API)** 워크플로우 선택
3. **Run workflow** 버튼 클릭
4. **Run workflow** 확인

### 3. 배포 확인

워크플로우 실행 후:

1. Actions 탭에서 실행 로그 확인
2. 성공 시 Object Storage URL로 접속 테스트
3. 브라우저에서 애플리케이션 동작 확인

## 문제 해결

### 자격 증명 오류

```
Error: The AWS Access Key Id you provided does not exist in our records.
```

**해결 방법**:
1. NHN Cloud Console에서 S3 API 자격 증명 재확인
2. GitHub Secrets에 올바르게 입력되었는지 확인
3. 공백이나 특수문자가 포함되지 않았는지 확인

### 컨테이너 접근 오류

```
Error: Access Denied
```

**해결 방법**:
1. Object Storage 컨테이너가 **퍼블릭 읽기**로 설정되었는지 확인
2. 컨테이너명이 워크플로우 파일과 일치하는지 확인

### 404 오류 (배포 후)

```
404 Not Found
```

**해결 방법**:
1. 정적 웹사이트 설정 확인
   - Index Document: `index.html`
   - Error Document: `index.html`
2. Object Storage URL이 올바른지 확인
3. 파일이 실제로 업로드되었는지 NHN Cloud Console에서 확인

### 워크플로우 실행 실패

1. **Actions 탭에서 로그 확인**
   - 각 단계의 상세 로그 확인
   - 에러 메시지 확인

2. **Secrets 재확인**
   - 모든 필수 Secrets가 추가되었는지 확인
   - 값이 올바른지 재확인

3. **권한 확인**
   - GitHub Actions에 필요한 권한이 있는지 확인
   - Repository Settings > Actions > General > Workflow permissions

## 보안 모범 사례

### 1. Secrets 관리

- ✅ GitHub Secrets에만 저장
- ✅ 정기적으로 자격 증명 갱신
- ❌ 코드에 직접 하드코딩 금지
- ❌ 로그에 출력 금지

### 2. 접근 제어

- ✅ Object Storage는 퍼블릭 읽기만 허용
- ✅ 쓰기 권한은 GitHub Actions만 가짐
- ✅ master 브랜치 보호 규칙 설정

### 3. 브랜치 보호

Repository Settings > Branches > Add rule:

- **Branch name pattern**: `master`
- **Require a pull request before merging**: 체크
- **Require approvals**: 1명 이상

## 추가 설정 (선택사항)

### CDN 연동

1. NHN Cloud Console > **CDN** > **CDN 서비스 생성**
2. **원본 서버**: Object Storage 컨테이너 URL
3. **도메인**: 커스텀 도메인 설정 (선택)

### 알림 설정

Slack, Discord 등으로 배포 알림 받기:

```yaml
# .github/workflows/deploy-to-object-storage-s3.yml에 추가
- name: Notify Slack
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 참고 자료

- [NHN Cloud S3 API 가이드](https://docs.nhncloud.com/ko/Storage/Object%20Storage/ko/s3-api-guide/)
- [GitHub Secrets 문서](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [AWS CLI S3 명령어](https://docs.aws.amazon.com/cli/latest/reference/s3/)
