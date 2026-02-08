# Presigned URL 업로드 구현

photo-api의 presigned URL 기능을 frontend에 적용했습니다.

## 변경 사항

### 1. API 엔드포인트 추가 (`src/config/api.js`)

presigned URL 관련 엔드포인트를 추가했습니다:

```javascript
photosPresignedUrl: () => `${API_BASE_URL}/photos/presigned-url`
photosConfirm: () => `${API_BASE_URL}/photos/confirm`
```

### 2. AlbumContext 업로드 방식 변경 (`src/contexts/AlbumContext.jsx`)

기존 레거시 방식(서버 경유 업로드)에서 presigned URL 방식으로 변경했습니다.

#### 변경 전 (레거시 방식)
```
클라이언트 → [파일] → 서버 → [파일] → Object Storage
```

#### 변경 후 (Presigned URL 방식)
```
1. 클라이언트 → [메타데이터] → 서버 → [Presigned URL] → 클라이언트
2. 클라이언트 → [파일] → Object Storage (직접)
3. 클라이언트 → [완료 확인] → 서버
```

#### 주요 개선사항

- **3단계 업로드 프로세스**:
  1. **Presigned URL 발급**: 서버에 메타데이터(파일명, 크기, 형식 등)만 전송
  2. **직접 업로드**: Object Storage에 파일을 직접 업로드 (서버 경유 없음)
  3. **업로드 확인**: 서버에 업로드 완료를 알림

- **업로드 진행률 추적**: XMLHttpRequest를 사용하여 실시간 진행률 표시

- **파일 검증**: 
  - 최대 파일 크기: 10MB
  - 지원 형식: JPEG, PNG, GIF, WebP, HEIC

### 3. 업로드 진행률 UI 추가 (`src/components/Image/AddImageModal.jsx`, `AddImageModal.css`)

사용자에게 업로드 상태를 시각적으로 표시:

- **진행률 바**: 0%부터 100%까지 실시간 표시
- **단계별 진행**:
  - 10%: Presigned URL 요청 중
  - 20%: Presigned URL 발급 완료
  - 20-90%: Object Storage 업로드 중
  - 90%: 파일 업로드 완료
  - 100%: 업로드 확인 완료

## 장점

### 1. 서버 부하 감소
- 파일이 서버를 거치지 않고 클라이언트에서 Object Storage로 직접 업로드
- 서버는 메타데이터만 처리

### 2. 업로드 속도 향상
- 중간 경유 없이 직접 업로드하므로 속도가 빠름
- 네트워크 지연 최소화

### 3. 서버 대역폭 절약
- 서버의 네트워크 트래픽이 크게 줄어듬
- 비용 절감 효과

### 4. 확장성
- 서버 리소스에 영향을 주지 않고 대용량 파일 업로드 가능
- 동시 업로드 처리 능력 향상

### 5. 사용자 경험 개선
- 실시간 업로드 진행률 표시
- 더 빠른 업로드 속도

## 사용 방법

### 기본 업로드

사용자는 기존과 동일한 방식으로 사진을 업로드할 수 있습니다:

1. 앨범 상세 페이지에서 "사진 추가" 버튼 클릭
2. 파일 선택 또는 드래그 앤 드롭
3. 사진 제목 및 설명 입력
4. "사진 추가" 버튼 클릭
5. 업로드 진행률 확인

### 내부 동작

```javascript
// 1. Presigned URL 발급
const presignedData = await fetch('/photos/presigned-url', {
  method: 'POST',
  body: JSON.stringify({
    album_id: albumId,
    filename: file.name,
    content_type: file.type,
    file_size: file.size,
    title: '사진 제목',
    description: '설명'
  })
});

// 2. Object Storage에 직접 업로드
await fetch(presignedData.upload_url, {
  method: 'PUT',
  headers: { 'Content-Type': file.type },
  body: file
});

// 3. 업로드 완료 확인
await fetch('/photos/confirm', {
  method: 'POST',
  body: JSON.stringify({
    photo_id: presignedData.photo_id
  })
});
```

## 에러 처리

다음과 같은 에러 상황을 처리합니다:

- **파일 크기 초과**: "파일 크기는 10MB를 초과할 수 없습니다."
- **지원하지 않는 형식**: "지원하지 않는 파일 형식입니다."
- **Presigned URL 발급 실패**: "Presigned URL 발급에 실패했습니다."
- **업로드 실패**: "파일 업로드에 실패했습니다."
- **네트워크 오류**: "파일 업로드 중 네트워크 오류가 발생했습니다."
- **업로드 확인 실패**: "업로드 확인에 실패했습니다."

## 호환성

- **브라우저 지원**: XMLHttpRequest를 사용하는 모든 최신 브라우저
- **기존 기능**: URL 기반 이미지 추가는 기존 방식 유지
- **레거시 API**: 백엔드의 레거시 업로드 엔드포인트(`POST /photos/`)는 여전히 사용 가능

## 테스트

개발 환경에서 테스트하려면:

1. photo-api 서버 실행 확인
2. NHN Cloud S3 자격 증명 설정 확인 (`NHN_S3_ACCESS_KEY`, `NHN_S3_SECRET_KEY`)
3. frontend 개발 서버 실행
4. 브라우저에서 앨범에 사진 업로드 테스트

## 참고 문서

- [photo-api PRESIGNED_URL_GUIDE.md](../photo-api/PRESIGNED_URL_GUIDE.md)
- [NHN Cloud Object Storage S3 API 가이드](https://docs.nhncloud.com/ko/Storage/Object%20Storage/ko/s3-api-guide/)
