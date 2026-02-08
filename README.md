# Photo Album Sharing - Frontend

React + Vite 기반의 사진 앨범 공유 웹 애플리케이션 프론트엔드입니다.

## 주요 기능

- ✅ 사용자 인증 (회원가입, 로그인)
- ✅ 앨범 생성 및 관리
- ✅ **Presigned URL 방식 사진 업로드** (서버 부하 감소, 속도 향상)
- ✅ 업로드 진행률 실시간 표시
- ✅ 앨범 공유 링크 생성
- ✅ 반응형 디자인

## 최근 업데이트

### Presigned URL 업로드 구현
- NHN Cloud Object Storage에 직접 업로드
- 서버 부하 감소 및 업로드 속도 향상
- 실시간 진행률 표시 (0% ~ 100%)

자세한 내용: [PRESIGNED_UPLOAD_IMPLEMENTATION.md](PRESIGNED_UPLOAD_IMPLEMENTATION.md)

### GitHub Actions 자동 배포
- master 브랜치 머지 시 자동 배포
- NHN Cloud Object Storage + CDN 배포

빠른 시작: [.github/QUICKSTART.md](.github/QUICKSTART.md)

## 기술 스택

- **React** 18.2.0 - UI 라이브러리
- **React Router** 6.20.0 - 라우팅
- **Vite** 5.0.0 - 빌드 도구
- **NHN Cloud Object Storage** - 정적 파일 호스팅

## 개발 환경 설정

### 1. 의존성 설치

```bash
npm install
```

### 2. 환경 변수 설정 (선택사항)

개발 환경에서는 기본적으로 `http://localhost:8000`을 사용합니다.

다른 API URL을 사용하려면:

```bash
# .env.local 파일 생성
echo "VITE_API_BASE_URL=http://your-api-url:8000" > .env.local
```

### 3. 개발 서버 실행

```bash
npm run dev
```

브라우저에서 http://localhost:5173 접속

## 빌드 및 배포

### 로컬 빌드

```bash
# 간편 빌드 (스크립트 사용)
./build.sh prod

# 또는 수동 빌드
VITE_API_BASE_URL="/api" npm run build

# 빌드 결과 미리보기
npm run preview
```

### 자동 배포 (GitHub Actions)

**master 브랜치에 머지하면 자동으로 NHN Cloud Object Storage에 배포됩니다.**

#### 초기 설정 (5분)

1. [빠른 시작 가이드](.github/QUICKSTART.md) 참조
2. GitHub Secrets 설정
3. Object Storage 컨테이너 생성
4. 완료!

#### 상세 가이드

- [GitHub Secrets 설정](.github/SETUP_SECRETS.md)
- [워크플로우 README](.github/workflows/README.md)
- [CDN 배포 가이드](DEPLOY_TO_CDN.md)

### 수동 배포 (VM + nginx)

```bash
# 빌드 및 압축
./build.sh prod
tar -czf photo-frontend.tar.gz dist/ nginx.conf deploy.sh

# 서버에 업로드
scp photo-frontend.tar.gz user@server:/opt/photo-frontend/

# 서버에서 배포
ssh user@server
cd /opt/photo-frontend
sudo ./deploy.sh
```

## 프로젝트 구조

```
frontend/
├── .github/
│   ├── workflows/
│   │   ├── deploy-to-object-storage.yml       # Swift API 배포
│   │   └── deploy-to-object-storage-s3.yml    # S3 API 배포 (권장)
│   ├── QUICKSTART.md                          # 빠른 시작 가이드
│   └── SETUP_SECRETS.md                       # Secrets 설정 가이드
├── src/
│   ├── components/
│   │   ├── Album/                             # 앨범 관련 컴포넌트
│   │   ├── Common/                            # 공통 컴포넌트
│   │   ├── Image/                             # 이미지 관련 컴포넌트
│   │   ├── Layout/                            # 레이아웃
│   │   └── ShareLink/                         # 공유 링크
│   ├── config/
│   │   └── api.js                             # API 설정 (presigned URL 포함)
│   ├── contexts/
│   │   ├── AlbumContext.jsx                   # 앨범 상태 관리
│   │   └── AuthContext.jsx                    # 인증 상태 관리
│   ├── pages/                                 # 페이지 컴포넌트
│   ├── styles/                                # 전역 스타일
│   ├── App.jsx                                # 앱 루트
│   └── main.jsx                               # 엔트리 포인트
├── build.sh                                   # 빌드 스크립트
├── deploy.sh                                  # VM 배포 스크립트
├── nginx.conf                                 # nginx 설정
├── vite.config.js                             # Vite 설정
└── package.json                               # 의존성 관리
```

## API 통신

### 기본 설정

`src/config/api.js`에서 API URL을 관리합니다:

```javascript
// 개발 환경: http://localhost:8000
// 프로덕션: /api (nginx proxy)
export const API_BASE_URL = getApiBaseUrl();
```

### Presigned URL 업로드

사진 업로드는 3단계로 진행됩니다:

1. **Presigned URL 발급**: `POST /photos/presigned-url`
2. **Object Storage 직접 업로드**: `PUT {presigned_url}`
3. **업로드 확인**: `POST /photos/confirm`

자세한 내용: [PRESIGNED_UPLOAD_IMPLEMENTATION.md](PRESIGNED_UPLOAD_IMPLEMENTATION.md)

## 환경 변수

| 변수 | 설명 | 기본값 |
|------|------|--------|
| `VITE_API_BASE_URL` | 백엔드 API URL | 개발: `http://localhost:8000`<br>프로덕션: `/api` |

## 스크립트

| 명령어 | 설명 |
|--------|------|
| `npm run dev` | 개발 서버 실행 (포트 5173) |
| `npm run build` | 프로덕션 빌드 |
| `npm run preview` | 빌드 결과 미리보기 |
| `./build.sh prod` | 프로덕션 빌드 (스크립트) |
| `./build.sh dev` | 개발 빌드 (스크립트) |

## 배포 체크리스트

### 초기 배포

- [ ] NHN Cloud S3 자격 증명 발급
- [ ] GitHub Secrets 설정
- [ ] Object Storage 컨테이너 생성 (퍼블릭 읽기)
- [ ] 정적 웹사이트 설정 (Index/Error Document)
- [ ] 로컬 빌드 테스트 (`npm run build`)
- [ ] GitHub Actions 수동 실행 테스트
- [ ] 배포된 사이트 접속 확인
- [ ] API 호출 테스트
- [ ] 사진 업로드 테스트

### 업데이트 배포

- [ ] 로컬에서 기능 테스트
- [ ] Pull Request 생성
- [ ] 코드 리뷰
- [ ] master 브랜치로 머지
- [ ] GitHub Actions 자동 배포 확인
- [ ] 배포된 사이트 테스트

## 문제 해결

### 개발 서버 실행 오류

```bash
# node_modules 재설치
rm -rf node_modules package-lock.json
npm install
```

### 빌드 오류

```bash
# 캐시 삭제 후 재빌드
rm -rf dist node_modules .vite
npm install
npm run build
```

### API 호출 실패

1. 백엔드 서버가 실행 중인지 확인
2. CORS 설정 확인 (백엔드)
3. API URL 확인 (`src/config/api.js`)
4. 브라우저 개발자 도구 네트워크 탭 확인

### 배포 후 404 오류

1. Object Storage 정적 웹사이트 설정 확인
   - Index Document: `index.html`
   - Error Document: `index.html`
2. 파일이 실제로 업로드되었는지 확인
3. 컨테이너가 퍼블릭 읽기로 설정되었는지 확인

## 브라우저 지원

- Chrome (최신 버전)
- Firefox (최신 버전)
- Safari (최신 버전)
- Edge (최신 버전)

## 라이선스

MIT

## 관련 문서

- [Presigned URL 업로드 구현](PRESIGNED_UPLOAD_IMPLEMENTATION.md)
- [CDN 배포 가이드](DEPLOY_TO_CDN.md)
- [GitHub Actions 빠른 시작](.github/QUICKSTART.md)
- [GitHub Secrets 설정](.github/SETUP_SECRETS.md)
- [워크플로우 상세 가이드](.github/workflows/README.md)

## 백엔드 연동

이 프론트엔드는 `photo-api` 백엔드와 함께 사용됩니다.

백엔드 설정: [../photo-api/README.md](../photo-api/README.md)
