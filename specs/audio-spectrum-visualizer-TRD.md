# 오디오 스펙트럼 비주얼라이저 TRD (Technical Requirements Document)

## 1. 개요

### 1.1 문서 목적
- **PRD 참조**: `audio-spectrum-visualizer-PRD.md` v1.0
- **SPEC 참조**: `audio-spectrum-visualizer-SPEC.md` v2.0
- **기술적 범위**: MP3 → 스펙트럼 비주얼라이저 + 자막 → MP4 영상 생성 웹 애플리케이션

### 1.2 기술 원칙
- **클라이언트 중심 렌더링**: 서버 부하 최소화, 사용자 PC 자원 활용
- **심플 아키텍처**: 단일 백엔드 + SPA 프론트엔드
- **실시간 피드백**: 모든 편집 사항은 즉시 미리보기에 반영

## 2. 시스템 아키텍처

### 2.1 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Client (Browser)                            │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │ React App   │  │ Web Audio    │  │ Canvas API  │  │ MediaRecorder │  │
│  │ (UI/State)  │  │ API (분석)   │  │ (렌더링)    │  │ (MP4 생성)    │  │
│  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘  └───────┬───────┘  │
│         │                │                  │                  │         │
│         └────────────────┴──────────────────┴──────────────────┘         │
│                                    │                                     │
└────────────────────────────────────┼─────────────────────────────────────┘
                                     │ HTTP/REST
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Server (Python FastAPI)                          │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │
│  │ Queue Manager   │  │ Whisper Service │  │ Lyrics Matching Service │  │
│  │ (대기열 관리)    │  │ (음성 인식)      │  │ (가사 매칭)              │  │
│  └────────┬────────┘  └────────┬────────┘  └────────────┬────────────┘  │
│           │                    │                        │                │
│           └────────────────────┴────────────────────────┘                │
│                                │                                         │
└────────────────────────────────┼─────────────────────────────────────────┘
                                 │
                                 ▼
                    ┌───────────────────────┐
                    │   OpenAI Whisper API  │
                    └───────────────────────┘
```

### 2.2 아키텍처 결정 기록 (ADR)

| 결정 | 선택 | 대안 | 근거 |
|------|------|------|------|
| 아키텍처 패턴 | 모놀리식 | 마이크로서비스 | 초기 복잡도 최소화, 단일 개발자 운영 |
| 렌더링 위치 | 클라이언트 | 서버 | 서버 비용 절감, 동시 처리 부담 감소 |
| 백엔드 언어 | Python | Node.js | Whisper API 연동, 텍스트 처리 강점 |
| 영상 생성 | MediaRecorder | ffmpeg.wasm | 브라우저 네이티브, 라이브러리 의존성 최소화 |
| 상태 관리 | Zustand | Redux, Context | 가벼움, 보일러플레이트 최소화 |
| 호스팅 | Self-hosting | Vercel | Serverless 타임아웃 제한 회피, 장시간 Whisper 처리 가능 |

### 2.3 컴포넌트 구성

| 컴포넌트 | 역할 | 기술 스택 |
|----------|------|-----------|
| Frontend | UI/UX, 오디오 분석, 영상 렌더링 | React, TypeScript, Tailwind CSS |
| Backend | 대기열 관리, Whisper 연동, 가사 매칭 | Python FastAPI |
| Audio Analyzer | 오디오 주파수 분석 | Web Audio API |
| Video Renderer | 스펙트럼 + 자막 영상 생성 | Canvas API + MediaRecorder |
| Whisper Service | 음성 → 텍스트 + 타임스탬프 | OpenAI Whisper API |

## 3. 기술 스택

### 3.1 Frontend

```yaml
Framework: React 18.x
Language: TypeScript 5.x
Styling: Tailwind CSS 3.x
State Management: Zustand 4.x
Build Tool: Vite 5.x
Package Manager: pnpm

주요 라이브러리:
  - @tanstack/react-query: API 상태 관리
  - react-dropzone: 파일 업로드
  - @dnd-kit/core: 드래그 앤 드롭 (타임라인)
  - lucide-react: 아이콘
```

### 3.2 Backend

```yaml
Runtime: Python 3.11+
Framework: FastAPI 0.100+
ASGI Server: Uvicorn

주요 라이브러리:
  - openai: Whisper API 연동
  - python-multipart: 파일 업로드 처리
  - pydantic: 데이터 검증
  - difflib: 가사 유사도 매칭
```

### 3.3 Infrastructure

```yaml
Hosting:
  - Primary: Fly.io / Render / Railway (택1)
  - 요구사항: 장시간 실행 지원, WebSocket 지원

Container: Docker
CI/CD: GitHub Actions

환경:
  - Development: 로컬 Docker Compose
  - Production: 단일 인스턴스 (MVP)
```

## 4. API 설계

### 4.1 API 스타일

```yaml
스타일: REST
Base URL: /api/v1
인증: 없음 (비로그인 서비스)
Content-Type: application/json, multipart/form-data
```

### 4.2 주요 엔드포인트

| Method | Endpoint | 설명 | Request | Response |
|--------|----------|------|---------|----------|
| GET | `/queue/status` | 대기열 현황 조회 | - | `{ current: 15, max: 20, position?: 3 }` |
| POST | `/queue/join` | 대기열 참가 | - | `{ session_id, position }` |
| DELETE | `/queue/leave` | 대기열 이탈 | `{ session_id }` | `{ success: true }` |
| POST | `/whisper/transcribe` | 음성 인식 요청 | `FormData(audio)` | `{ segments: [...] }` |
| POST | `/lyrics/match` | 가사 매칭 요청 | `{ whisper_segments, original_lyrics }` | `{ matched_segments: [...] }` |
| GET | `/health` | 서버 상태 확인 | - | `{ status: "ok" }` |

### 4.3 Whisper Transcribe 상세

**Request**
```http
POST /api/v1/whisper/transcribe
Content-Type: multipart/form-data

audio: <MP3 file>
```

**Response**
```json
{
  "segments": [
    {
      "id": 1,
      "start": 0.0,
      "end": 2.5,
      "text": "너를 만나고"
    },
    {
      "id": 2,
      "start": 2.8,
      "end": 5.2,
      "text": "이렇게 좋은데"
    }
  ],
  "duration": 180.5,
  "language": "ko"
}
```

### 4.4 Lyrics Match 상세

**Request**
```json
{
  "whisper_segments": [
    { "id": 1, "start": 0.0, "end": 2.5, "text": "너를 만나고" }
  ],
  "original_lyrics": "너를 만나고\n이렇게 좋은데\n어쩌면 우린"
}
```

**Response**
```json
{
  "matched_segments": [
    {
      "id": 1,
      "start": 0.0,
      "end": 2.5,
      "whisper_text": "너를 만나고",
      "matched_text": "너를 만나고",
      "confidence": "exact",
      "needs_review": false
    },
    {
      "id": 2,
      "start": 2.8,
      "end": 5.2,
      "whisper_text": "이렇게 조은데",
      "matched_text": "이렇게 좋은데",
      "confidence": "similar",
      "similarity": 0.85,
      "needs_review": false
    },
    {
      "id": 3,
      "start": 5.5,
      "end": 8.0,
      "whisper_text": "아이 러브 유",
      "matched_text": null,
      "confidence": "failed",
      "needs_review": true
    }
  ]
}
```

### 4.5 에러 응답 형식

```json
{
  "error": {
    "code": "QUEUE_FULL",
    "message": "대기열이 가득 찼습니다. 잠시 후 다시 시도해주세요.",
    "details": {
      "current": 20,
      "max": 20
    }
  }
}
```

### 4.6 에러 코드

| Code | HTTP Status | 설명 |
|------|-------------|------|
| `QUEUE_FULL` | 503 | 대기열 초과 |
| `INVALID_FILE_TYPE` | 400 | 지원하지 않는 파일 형식 |
| `FILE_TOO_LARGE` | 413 | 파일 크기 초과 (50MB) |
| `WHISPER_API_ERROR` | 502 | Whisper API 오류 |
| `SESSION_EXPIRED` | 410 | 세션 만료 |

## 5. 데이터 모델

### 5.1 클라이언트 상태 (Zustand Store)

```typescript
// 프로젝트 상태
interface ProjectState {
  // 파일
  audioFile: File | null;
  backgroundImage: File | null;

  // 스펙트럼 설정
  spectrumStyle: 'bar' | 'waveform' | 'circular';
  spectrumColor: string;
  spectrumSize: number;

  // 가사 세그먼트
  segments: LyricSegment[];
  originalLyrics: string | null;

  // 자막 기본 스타일
  defaultSubtitleStyle: SubtitleStyle;

  // 대기열
  queuePosition: number | null;
  sessionId: string | null;
}

// 가사 세그먼트
interface LyricSegment {
  id: string;
  start: number;      // 초 단위
  end: number;
  text: string;
  needsReview: boolean;
  style?: Partial<SubtitleStyle>;  // 개별 오버라이드
}

// 자막 스타일
interface SubtitleStyle {
  position: 'top' | 'middle' | 'bottom';
  color: string;
  fontSize: 'small' | 'medium' | 'large';
  fontFamily: string;
}
```

### 5.2 서버 상태 (In-Memory)

```python
# 대기열 관리 (Redis 없이 메모리 사용 - MVP)
class QueueManager:
    sessions: dict[str, SessionInfo]  # session_id -> info
    queue: list[str]                   # 대기 순서
    active_count: int                  # 현재 활성 사용자
    max_active: int = 20               # 최대 동시 접속

class SessionInfo:
    session_id: str
    joined_at: datetime
    last_active: datetime
    status: Literal['waiting', 'active', 'expired']
```

## 6. 핵심 기능 구현

### 6.1 오디오 분석 (Web Audio API)

```typescript
// 주파수 분석 설정
const analyserConfig = {
  fftSize: 2048,           // FFT 크기 (주파수 해상도)
  smoothingTimeConstant: 0.8,  // 스무딩
  minDecibels: -90,
  maxDecibels: -10
};

// 스펙트럼 데이터 추출
function getFrequencyData(analyser: AnalyserNode): Uint8Array {
  const dataArray = new Uint8Array(analyser.frequencyBinCount);
  analyser.getByteFrequencyData(dataArray);
  return dataArray;
}
```

### 6.2 스펙트럼 렌더링 (Canvas API)

```typescript
// 렌더링 루프
function renderFrame(
  ctx: CanvasRenderingContext2D,
  frequencyData: Uint8Array,
  config: SpectrumConfig,
  currentTime: number,
  segments: LyricSegment[]
) {
  // 1. 배경 그리기
  drawBackground(ctx, config.backgroundImage);

  // 2. 스펙트럼 그리기
  switch (config.style) {
    case 'bar':
      drawBarSpectrum(ctx, frequencyData, config);
      break;
    case 'waveform':
      drawWaveformSpectrum(ctx, frequencyData, config);
      break;
    case 'circular':
      drawCircularSpectrum(ctx, frequencyData, config);
      break;
  }

  // 3. 현재 시간의 가사 그리기
  const activeSegment = findActiveSegment(segments, currentTime);
  if (activeSegment) {
    drawSubtitle(ctx, activeSegment);
  }
}
```

### 6.3 영상 생성 (MediaRecorder)

```typescript
async function renderVideo(
  audioFile: File,
  config: RenderConfig
): Promise<Blob> {
  const canvas = document.createElement('canvas');
  canvas.width = 1920;
  canvas.height = 1080;

  const stream = canvas.captureStream(30); // 30fps
  const audioContext = new AudioContext();
  const audioSource = await loadAudioToContext(audioContext, audioFile);

  // 오디오 트랙 추가
  const audioDestination = audioContext.createMediaStreamDestination();
  audioSource.connect(audioDestination);
  stream.addTrack(audioDestination.stream.getAudioTracks()[0]);

  const recorder = new MediaRecorder(stream, {
    mimeType: 'video/webm;codecs=vp9,opus',
    videoBitsPerSecond: 8000000
  });

  const chunks: Blob[] = [];
  recorder.ondataavailable = (e) => chunks.push(e.data);

  return new Promise((resolve) => {
    recorder.onstop = () => {
      resolve(new Blob(chunks, { type: 'video/webm' }));
    };

    recorder.start();
    audioSource.start();

    // 렌더링 루프 시작
    startRenderLoop(canvas, audioContext, config);
  });
}
```

### 6.4 가사 매칭 알고리즘

```python
from difflib import SequenceMatcher
import re

def match_lyrics(whisper_segments: list, original_lyrics: str) -> list:
    """Whisper 결과와 원본 가사 매칭"""

    # 원본 가사 정규화
    normalized_original = normalize_text(original_lyrics)
    position = 0  # 순서 기반 포인터

    results = []

    for segment in whisper_segments:
        whisper_text = normalize_text(segment['text'])

        # 1차: 정확 매칭
        match_pos = normalized_original.find(whisper_text, position)
        if match_pos != -1:
            results.append({
                **segment,
                'matched_text': get_original_text(original_lyrics, match_pos, len(whisper_text)),
                'confidence': 'exact',
                'needs_review': False
            })
            position = match_pos + len(whisper_text)
            continue

        # 2차: 유사도 매칭 (70% 이상)
        best_match, similarity = find_similar_match(
            whisper_text,
            normalized_original[position:],
            threshold=0.7
        )
        if best_match:
            results.append({
                **segment,
                'matched_text': best_match,
                'confidence': 'similar',
                'similarity': similarity,
                'needs_review': False
            })
            position += len(best_match)
            continue

        # 3차: 매칭 실패 → 사용자 확인 필요
        results.append({
            **segment,
            'matched_text': None,
            'confidence': 'failed',
            'needs_review': True
        })

    return results

def normalize_text(text: str) -> str:
    """텍스트 정규화: 특수문자 제거, 공백 통일"""
    text = re.sub(r'[.,!?~♪♩♫♬\-_]', '', text)
    text = re.sub(r'\s+', ' ', text)
    return text.strip().lower()
```

## 7. 대기열 시스템

### 7.1 동작 방식

```
┌─────────────────────────────────────────────────────────────┐
│                      대기열 시스템                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [활성 사용자 풀]  ←──────────────┐                         │
│  ┌───┬───┬───┬───┬───┐          │                         │
│  │ 1 │ 2 │ 3 │...│20 │   max=20 │                         │
│  └───┴───┴───┴───┴───┘          │                         │
│         ↑                        │                         │
│         │ 입장                   │ 퇴장/만료                │
│         │                        │                         │
│  [대기열]                        │                         │
│  ┌───┬───┬───┬───┬───┐          │                         │
│  │ A │ B │ C │ D │...│──────────┘                         │
│  └───┴───┴───┴───┴───┘                                     │
│    ↑                                                        │
│    │ 신규 접속                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 세션 관리

```python
# 세션 타임아웃 설정
SESSION_TIMEOUT = 30 * 60  # 30분 비활성 시 만료
HEARTBEAT_INTERVAL = 30    # 30초마다 하트비트

# 클라이언트 하트비트
@app.post("/api/v1/queue/heartbeat")
async def heartbeat(session_id: str):
    queue_manager.update_last_active(session_id)
    return {"status": "ok"}

# 만료 세션 정리 (백그라운드 태스크)
@app.on_event("startup")
async def start_cleanup_task():
    asyncio.create_task(cleanup_expired_sessions())

async def cleanup_expired_sessions():
    while True:
        await asyncio.sleep(60)  # 1분마다 체크
        queue_manager.remove_expired_sessions()
```

## 8. 보안 설계

### 8.1 보안 체크리스트

- [x] **파일 업로드 검증**
  - MIME 타입 확인 (audio/mpeg, image/*)
  - 파일 크기 제한 (MP3: 50MB, 이미지: 10MB)
  - 파일명 sanitization

- [x] **Rate Limiting**
  - Whisper API 호출: 분당 10회 / IP
  - 대기열 참가: 분당 5회 / IP

- [x] **CORS 설정**
  ```python
  app.add_middleware(
      CORSMiddleware,
      allow_origins=["https://yourdomain.com"],
      allow_methods=["GET", "POST", "DELETE"],
      allow_headers=["*"],
  )
  ```

- [x] **입력 검증**
  - Pydantic 모델로 모든 입력 검증
  - 가사 텍스트 길이 제한 (10,000자)

### 8.2 비로그인 서비스 보안

| 항목 | 적용 방법 |
|------|-----------|
| 세션 관리 | UUID 기반 임시 세션 (서버 메모리) |
| 데이터 저장 | 서버에 파일 저장 안 함 (클라이언트 처리) |
| API 남용 방지 | IP 기반 Rate Limiting |

## 9. 성능 요구사항

### 9.1 성능 목표

| 지표 | 목표 | 측정 방법 |
|------|------|-----------|
| 미리보기 반영 | < 500ms | 사용자 입력 → 화면 반영 |
| Whisper 처리 | < 30초 (5분 음악) | API 응답 시간 |
| 렌더링 속도 | 실시간 이상 | 3분 음악 → 3분 이하 |
| 초기 로드 | < 3초 | Lighthouse LCP |

### 9.2 최적화 전략

**Frontend**
```yaml
번들 최적화:
  - Code Splitting: 라우트 기반
  - Tree Shaking: 미사용 코드 제거
  - 이미지 최적화: WebP 변환

렌더링 최적화:
  - requestAnimationFrame 사용
  - OffscreenCanvas (Worker 활용, 지원 시)
  - 미리보기 해상도 다운스케일 (720p → 렌더링 시 1080p)
```

**Backend**
```yaml
Whisper 최적화:
  - 스트리밍 응답 (진행률 표시)
  - 오디오 전처리 (불필요한 무음 제거)

서버 최적화:
  - Uvicorn 워커 수 조절
  - 메모리 효율적인 파일 처리 (스트리밍)
```

## 10. 프로젝트 구조

### 10.1 Frontend 구조

```
frontend/
├── src/
│   ├── components/
│   │   ├── common/          # 공통 컴포넌트
│   │   ├── editor/          # 에디터 관련
│   │   │   ├── Preview.tsx
│   │   │   ├── Timeline.tsx
│   │   │   ├── LyricEditor.tsx
│   │   │   ├── OriginalLyricsPanel.tsx
│   │   │   └── SettingsPanel.tsx
│   │   ├── spectrum/        # 스펙트럼 렌더러
│   │   │   ├── BarSpectrum.tsx
│   │   │   ├── WaveformSpectrum.tsx
│   │   │   └── CircularSpectrum.tsx
│   │   └── queue/           # 대기열 UI
│   │       └── QueueStatus.tsx
│   ├── hooks/
│   │   ├── useAudioAnalyzer.ts
│   │   ├── useVideoRenderer.ts
│   │   └── useQueue.ts
│   ├── stores/
│   │   └── projectStore.ts
│   ├── services/
│   │   └── api.ts
│   ├── utils/
│   │   ├── srt.ts           # SRT 파싱/생성
│   │   └── audio.ts
│   └── types/
│       └── index.ts
├── package.json
└── vite.config.ts
```

### 10.2 Backend 구조

```
backend/
├── app/
│   ├── main.py              # FastAPI 앱
│   ├── routers/
│   │   ├── queue.py
│   │   ├── whisper.py
│   │   └── lyrics.py
│   ├── services/
│   │   ├── queue_manager.py
│   │   ├── whisper_service.py
│   │   └── lyrics_matcher.py
│   ├── models/
│   │   └── schemas.py       # Pydantic 모델
│   └── utils/
│       └── text_normalize.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

## 11. 배포 및 인프라

### 11.1 배포 환경

```yaml
Production:
  Platform: Fly.io (권장) / Render / Railway

  Fly.io 설정:
    - Region: nrt (Tokyo) - 한국 사용자 대상
    - Machine: shared-cpu-1x, 512MB RAM
    - Auto-scaling: 비활성화 (단일 인스턴스)

  환경 변수:
    - OPENAI_API_KEY: Whisper API 키
    - ALLOWED_ORIGINS: CORS 허용 도메인
```

### 11.2 Docker 설정

```dockerfile
# Backend Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY ./app ./app
EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 11.3 CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy Frontend
        # Vercel / Netlify 자동 배포

      - name: Deploy Backend
        uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

## 12. 테스트 전략

### 12.1 테스트 범위

| 레벨 | 도구 | 대상 | 커버리지 목표 |
|------|------|------|---------------|
| Unit | Vitest | 유틸리티, 상태 관리 | 80% |
| Integration | Vitest + MSW | API 연동 | 주요 흐름 |
| E2E | Playwright | 전체 사용자 플로우 | 핵심 시나리오 |

### 12.2 핵심 테스트 시나리오

```markdown
1. 파일 업로드 플로우
   - MP3 업로드 → 미리보기 재생 확인
   - 배경 이미지 업로드 → 미리보기 반영 확인

2. Whisper 연동
   - MP3 → Whisper → 타임라인 세그먼트 생성
   - 원본 가사 + Whisper → 매칭 결과 확인

3. 타임라인 편집
   - 가사 텍스트 수정 → 저장 확인
   - 드래그로 타이밍 조절 → 변경 반영 확인
   - 가사 순서 변경 → 정렬 확인

4. 영상 렌더링
   - 렌더링 시작 → 진행률 표시 → 완료 → 다운로드

5. 대기열 시스템
   - 20명 초과 시 → 대기열 진입 확인
   - 순서 대기 → 자동 입장 확인
```

## 13. 기술적 리스크 및 완화

| 리스크 | 영향 | 완화 방안 |
|--------|------|-----------|
| 브라우저 렌더링 성능 한계 | 높음 | 미리보기 해상도 다운스케일, Web Worker 활용 |
| MediaRecorder 호환성 | 중간 | 브라우저 지원 체크, ffmpeg.wasm 폴백 |
| Whisper API 응답 지연 | 중간 | 스트리밍 응답, 진행률 표시, 타임아웃 설정 |
| 메모리 부족 (긴 음악) | 중간 | 청크 단위 처리, 메모리 모니터링 |
| OpenAI API 비용 초과 | 낮음 | Rate Limiting, 일일 사용량 모니터링 |

---

**문서 버전**: v1.0
**작성일**: 2026-01-03
**상태**: 📝 초안
**기반 문서**:
- `audio-spectrum-visualizer-SPEC.md` v2.0
- `audio-spectrum-visualizer-PRD.md` v1.0
