# CDN 배포 가이드

## 개요

frontend 애플리케이션을 CDN에 배포하는 방법을 설명합니다.

## 빌드 필수!

⚠️ **CDN에 배포하기 전에 반드시 빌드를 수행해야 합니다.**

개발 환경의 소스 코드(JSX, TypeScript 등)는 브라우저에서 직접 실행할 수 없으므로, 빌드 과정을 통해 정적 파일로 변환해야 합니다.

## 빌드 방법
.
### 1. 의존성 설치

```bash
cd frontend
npm install
```

### 2. 환경 변수 설정

프로덕션 환경의 백엔드 API URL을 설정합니다:

**방법 A: 빌드 시 환경 변수 지정**

```bash
# nginx proxy를 사용하는 경우 (/api로 프록시)
VITE_API_BASE_URL="/api" npm run build

# 백엔드 직접 호출
VITE_API_BASE_URL="https://api.example.com" npm run build
```

**방법 B: .env.production 파일 생성**

```bash
# frontend/.env.production
VITE_API_BASE_URL=/api
```

### 3. 빌드 실행

```bash
npm run build
```

빌드가 완료되면 `dist/` 폴더가 생성됩니다:

```
dist/
├── index.html
├── share-preview.html
├── assets/
│   ├── index-[hash].js
│   ├── index-[hash].css
│   └── [이미지 파일들]
└── ...
```

### 4. 빌드 결과 확인

로컬에서 빌드 결과를 미리 확인할 수 있습니다:

```bash
npm run preview
```

## CDN 배포 시나리오

### 시나리오 1: NHN Cloud Object Storage + CDN (권장)

**장점**: 정적 파일 호스팅에 최적화, 저렴한 비용, 높은 가용성

#### 1.1. Object Storage 버킷 생성

```bash
# NHN Cloud Console에서 수행
# Storage > Object Storage > 컨테이너 생성
# - 컨테이너명: photo-frontend
# - 액세스 정책: 퍼블릭 읽기
```

#### 1.2. 빌드 파일 업로드

**방법 A: NHN Cloud Console 사용**

1. Object Storage 컨테이너로 이동
2. `dist/` 폴더의 모든 파일 업로드
3. 폴더 구조 유지 필수

**방법 B: swift CLI 사용**

```bash
# Swift CLI 설치 및 인증 설정 후
cd dist
swift upload photo-frontend .
```

**방법 C: AWS S3 CLI 사용 (S3 호환 API)**

```bash
# AWS CLI 설치 및 NHN Cloud S3 자격 증명 설정 후
aws s3 sync dist/ s3://photo-frontend/ \
  --endpoint-url https://kr1-api-object-storage.nhncloudservice.com \
  --acl public-read
```

#### 1.3. CDN 연동

```bash
# NHN Cloud Console에서 수행
# CDN > CDN 서비스 생성
# - 원본 서버: Object Storage 컨테이너 URL
# - 도메인: photo.example.com (선택사항)
```

#### 1.4. 정적 웹사이트 설정

Object Storage에서 정적 웹사이트 기능 활성화:

```bash
# Container 설정
# - Static Website: 활성화
# - Index Document: index.html
# - Error Document: index.html (SPA 라우팅 지원)
```

### 시나리오 2: VM에 nginx로 배포

**장점**: 백엔드와 함께 배포, API 프록시 설정 용이

#### 2.1. 빌드

```bash
VITE_API_BASE_URL="/api" npm run build
```

#### 2.2. 서버에 업로드

```bash
# 빌드 결과를 압축
cd frontend
tar -czf ../photo-frontend.tar.gz dist/ nginx.conf deploy.sh

# 서버에 업로드
scp photo-frontend.tar.gz user@server:/opt/photo-frontend/
```

#### 2.3. 배포 스크립트 실행

```bash
# 서버에 SSH 접속
ssh user@server

# 배포 디렉토리로 이동
cd /opt/photo-frontend

# 압축 해제
tar -xzf photo-frontend.tar.gz

# 배포 스크립트 실행 (자동으로 nginx 설정)
sudo ./deploy.sh
```

**deploy.sh가 수행하는 작업**:
1. nginx 설치 확인
2. 빌드 파일 확인 (없으면 자동 빌드)
3. nginx 설정 (API 프록시 포함)
4. 파일 배포 (`/var/www/photo-album/`)
5. nginx 재시작

### 시나리오 3: 다른 CDN 서비스 (Cloudflare, AWS CloudFront 등)

#### Cloudflare Pages

```bash
# 빌드
npm run build

# Cloudflare Pages에 배포
# 1. Cloudflare Dashboard에서 Pages 생성
# 2. dist/ 폴더 업로드
# 3. Build 설정:
#    - Build command: npm run build
#    - Build output directory: dist
#    - Environment variables: VITE_API_BASE_URL=/api
```

#### AWS CloudFront + S3

```bash
# 빌드
npm run build

# S3 업로드
aws s3 sync dist/ s3://your-bucket-name/ --acl public-read

# CloudFront 배포 생성
aws cloudfront create-distribution \
  --origin-domain-name your-bucket-name.s3.amazonaws.com
```

## CORS 설정 (중요!)

CDN에서 호스팅하는 경우, 백엔드 API에서 CORS를 허용해야 합니다:

```python
# photo-api/app/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://your-cdn-domain.com",  # CDN 도메인 추가
        "http://localhost:5173",         # 개발 환경
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## SPA 라우팅 설정 (공유 링크 `/share/xxx` 등)

React Router를 사용하므로, **직접 URL 접근**(예: `/share/abc123`, `/albums`, `/login`) 시에도 `index.html`이 반환되어야 합니다. 그렇지 않으면 서버가 404를 주고 SPA가 로드되지 않습니다.

### CDN만 사용할 때 (nginx 없이) — 필수 설정

프론트를 **Object Storage + CDN**으로만 서빙하면, `/share/토큰` 같은 경로는 **실제 파일이 없어 404**가 납니다. 아래 중 하나를 반드시 해두어야 공유 링크가 동작합니다.

#### 1) Object Storage 원본에서 Error Document 설정 (권장)

원본(Object Storage)에서 **404일 때 `index.html`을 반환**하도록 설정합니다. CDN은 그 응답을 그대로 캐시·전달합니다.

- **NHN Cloud Console**  
  **Storage > Object Storage** → 해당 컨테이너 선택 → **컨테이너 설정** (또는 정적 웹사이트 설정)  
  - **Index Document**: `index.html`  
  - **Error Document**: `index.html`  ← 이 설정이 있어야 `/share/xxx` 등이 SPA로 열림  

NHN에서 “정적 웹사이트” 또는 “Error Document” 옵션이 없다면, 콘솔 메뉴 이름이 다를 수 있으니 Object Storage 문서를 참고하거나 지원에 문의하세요.

#### 2) CDN에서 404 시 index.html 반환 (CDN이 지원하는 경우)

일부 CDN은 **404일 때 커스텀 페이지(예: `/index.html`)를 200으로 반환**하는 기능을 제공합니다.  
NHN CDN에 해당 기능이 있으면, 404 시 `index.html` 내용을 반환하도록 설정하면 됩니다.

#### 3) 대안: Hash 라우팅 사용

Object Storage / CDN에서 위 설정을 할 수 없으면, **HashRouter**로 공유 링크를 `https://도메인/#/share/토큰` 형태로 쓰는 방법이 있습니다. 서버는 항상 `/`만 보므로 별도 설정 없이 동작합니다. 단, URL에 `#`이 들어갑니다.

- 공유 URL 형식: `https://your-cdn.com/#/share/토큰`
- 적용 방법: [아래 HashRouter 옵션](#hashrouter-대안) 참고

---

### 환경별 요약

| 환경 | 설정 |
|------|------|
| **Object Storage + CDN** | 컨테이너 **Error Document = index.html** (필수). CDN 404 커스텀 페이지 지원 시 거기서도 설정 가능. |
| **nginx** | `try_files $uri $uri/ /index.html;` (프론트용 nginx.conf에 이미 있음) |
| **Cloudflare Pages** | 기본 지원 |
| **AWS CloudFront + S3** | S3 버킷 정적 웹사이트 호스팅 사용 후, Error Document = index.html. CloudFront에서 403/404 Custom Error Response → 200, Response Page Path = `/index.html` |

### HashRouter 대안 (Error Document 설정 불가할 때)

CDN/Object Storage에서 Error Document를 쓸 수 없을 때만 사용하세요. 공유 URL이 `https://도메인/#/share/토큰` 형태가 됩니다.

1. **빌드 시** 환경 변수로 Hash 라우팅 사용:
   ```bash
   VITE_USE_HASH_ROUTER=true npm run build
   ```
2. 앱에서 복사하는 공유 링크는 자동으로 **`/#/share/토큰`** 형식으로 생성됩니다.
3. 이미 퍼뜨린 **`/share/토큰`** (슬래시 경로) 링크는 Error Document를 설정해야 동작합니다. HashRouter는 **새로 만드는 공유 링크**만 `#` 형식으로 바꿔 줍니다.

## 배포 체크리스트

- [ ] 빌드 수행 (`npm run build`)
- [ ] 환경 변수 설정 (`VITE_API_BASE_URL`)
- [ ] dist/ 폴더 생성 확인
- [ ] 로컬에서 빌드 결과 테스트 (`npm run preview`)
- [ ] CDN/서버에 업로드
- [ ] 백엔드 CORS 설정 확인
- [ ] SPA 라우팅 설정 확인 (**CDN만 쓸 때**: Object Storage Error Document = `index.html` 또는 HashRouter 사용)
- [ ] **공유 링크 직접 열기** 테스트: `https://도메인/share/토큰` 또는 `https://도메인/#/share/토큰` (HashRouter 사용 시)
- [ ] 브라우저에서 배포된 사이트 테스트
- [ ] API 호출 테스트
- [ ] 이미지 업로드 테스트 (presigned URL)

## 자동 배포 (CI/CD)

### GitHub Actions (이미 구현됨!)

이 프로젝트에는 이미 GitHub Actions 워크플로우가 구현되어 있습니다.

**위치**: `.github/workflows/deploy-to-object-storage-s3.yml`

**사용 방법**:
1. [빠른 시작 가이드](.github/QUICKSTART.md) 참조
2. GitHub Secrets 설정 (NHN_S3_ACCESS_KEY, NHN_S3_SECRET_KEY)
3. master 브랜치에 머지하면 자동 배포

**상세 가이드**:
- [GitHub Secrets 설정](.github/SETUP_SECRETS.md)
- [워크플로우 README](.github/workflows/README.md)

## 문제 해결

### 빌드 오류

```bash
# node_modules 삭제 후 재설치
rm -rf node_modules package-lock.json
npm install
npm run build
```

### API 호출 실패

1. 브라우저 개발자 도구에서 네트워크 탭 확인
2. CORS 오류 확인 → 백엔드 CORS 설정 수정
3. API URL 확인 → `VITE_API_BASE_URL` 환경 변수 확인

### 라우팅 오류 (404)

- SPA 라우팅 설정 확인
- 모든 경로를 `index.html`로 리다이렉트해야 함

### 캐시 문제

```bash
# CDN 캐시 무효화
# NHN Cloud Console > CDN > 캐시 재배포
# 또는 파일명에 해시가 포함되어 자동으로 캐시 무효화됨 (Vite 기본 동작)
```

## 참고 자료

- [Vite 프로덕션 빌드 가이드](https://vitejs.dev/guide/build.html)
- [NHN Cloud Object Storage 문서](https://docs.nhncloud.com/ko/Storage/Object%20Storage/ko/overview/)
- [NHN Cloud CDN 문서](https://docs.nhncloud.com/ko/Contents%20Delivery/CDN/ko/overview/)
