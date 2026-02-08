# 빠른 시작 가이드 - GitHub Actions 자동 배포

master 브랜치에 머지하면 자동으로 NHN Cloud Object Storage에 배포되도록 설정하는 5분 가이드입니다.

## 1단계: NHN Cloud S3 자격 증명 발급 (2분)

1. [NHN Cloud Console](https://console.nhncloud.com/) 로그인
2. **Storage** > **Object Storage** 이동
3. **API 엔드포인트 설정** 탭 클릭
4. **S3 API 자격 증명 발급** 클릭
5. **Access Key**와 **Secret Key** 복사 (안전하게 보관!)

## 2단계: GitHub Secrets 추가 (2분)

1. GitHub Repository > **Settings** > **Secrets and variables** > **Actions**
2. **New repository secret** 클릭
3. 두 개의 Secret 추가:

```
Name: NHN_S3_ACCESS_KEY
Value: [복사한 Access Key]

Name: NHN_S3_SECRET_KEY
Value: [복사한 Secret Key]
```

## 3단계: Object Storage 컨테이너 생성 (1분)

1. NHN Cloud Console > **Storage** > **Object Storage**
2. **컨테이너 생성** 클릭
3. 설정:
   - 컨테이너명: `photo-frontend`
   - 액세스 정책: **퍼블릭 읽기**
4. 컨테이너 선택 > **컨테이너 설정**
5. **Static Website** 활성화:
   - Index Document: `index.html`
   - Error Document: `index.html`

## 4단계: 워크플로우 파일 확인

`.github/workflows/deploy-to-object-storage-s3.yml` 파일에서 컨테이너명 확인:

```yaml
env:
  BUCKET_NAME: 'photo-frontend'  # 3단계에서 만든 컨테이너명
```

다른 이름을 사용했다면 수정하세요.

## 5단계: 테스트

### 수동 배포 테스트

1. GitHub Repository > **Actions** 탭
2. **Deploy to NHN Cloud Object Storage (S3 API)** 선택
3. **Run workflow** 클릭
4. 완료 후 Object Storage URL로 접속

### 자동 배포 테스트

```bash
# 개발 브랜치에서 작업
git checkout -b test-deploy
echo "test" >> README.md
git add .
git commit -m "Test auto deploy"
git push origin test-deploy

# GitHub에서 Pull Request 생성 후 master로 머지
# → 자동으로 배포 시작!
```

## 완료! 🎉

이제 master 브랜치에 머지할 때마다 자동으로 배포됩니다.

**접속 URL**: NHN Cloud Console > Object Storage > 컨테이너 설정에서 확인

## 문제가 있나요?

- [상세 설정 가이드](.github/SETUP_SECRETS.md)
- [워크플로우 README](.github/workflows/README.md)
- [문제 해결 가이드](.github/workflows/README.md#문제-해결)
