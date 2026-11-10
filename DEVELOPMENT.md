# DEVELOPMENT.md — 우리 반 AI 키우기 (가칭)

특수교육용 이미지 학습 도구의 개발 명세다. 티쳐블 머신과 같은 원리(공개된 작은 이미지 모델 + 브라우저 안 학습)를 쓰되, 지적장애·중도중복장애 학생이 쓸 수 있는 ‘반려 AI 키우기’ 게임으로 감싼다.

화면·캐릭터·색·글꼴·대사는 [DESIGN.md](./DESIGN.md)를 따른다. 두 문서는 같은 ID(화면 `S*`, 단계 `stage`, 표정 `mood`, 대사 ID, 토큰)를 쓴다.

---

## 1. 목표와 범위

### 1.1 꼭 되어야 하는 것

1. 태블릿·노트북 브라우저에서 카메라로 사진을 모아(먹이 주기) 3개 칸(캔·페트병·종이) + 빈 자리 칸을 학습시킨다.
2. 학습은 버튼(신호)으로 시작하고, 학습 과정이 눈에 보인다.
3. 시험 물건 6개로 퀴즈(시험)를 하고, 회차별 결과를 저장해 전·후를 비교한다.
4. ‘골고루 먹였는지’(장소 다양성)와 점수 향상으로 반려 AI가 성장한다.
5. 근무 모드에서 실시간으로 물건을 알아보고 큰 그림과 음성으로 안내한다.
6. 다 수준 학생이 버튼·키보드형 스위치·소리 신호로 참여한다.
7. 모든 데이터는 기기 안(IndexedDB)에 저장되고, 탭을 닫아도 남는다. 파일로 내보내고 가져올 수 있다.
8. **실행 중 외부 서버로 아무것도 보내지 않는다.** 사진, 소리, 기록 모두 기기 밖으로 나가지 않는다.

### 1.2 하지 않는 것

계정·로그인, 서버·클라우드 동기화, 사용 통계 수집, 얼굴 인식, 소리·동작 모델, 여러 기기 간 실시간 공유.

---

## 2. 기술 구성

| 영역 | 선택 | 이유 |
|---|---|---|
| 빌드 | Vite + React + TypeScript | 가볍고 코덱스가 다루기 쉬움 |
| 스타일 | CSS 변수(`src/styles/tokens.css`) + CSS Modules | DESIGN.md 토큰을 그대로 옮김. UI 라이브러리 쓰지 않음 |
| 화면 전환 | 라우터 없이 화면 상태(`useReducer` + Context) | 키오스크형 단일 화면 앱 |
| 머신러닝 | `@tensorflow/tfjs` | 브라우저 안 학습·추론 |
| 특징 추출 모델 | MobileNet v3 small 100 224 feature vector (출력 1024차원) | 티쳐블 머신·구글 코드랩과 같은 방식, 작고 빠름 |
| 저장 | `dexie` (IndexedDB) | 사진·특징값·기록 저장 |
| 학습된 모델 저장 | `tf.io` `indexeddb://` | 탭을 닫아도 유지 |
| 테스트 | `vitest` | 게임 규칙·판정 로직 단위 테스트 |
| 오프라인(마지막 단계) | `vite-plugin-pwa` | 학교 와이파이가 불안정해도 동작 |
| 배포 | Vercel (https 필수: 카메라·마이크는 https에서만 켜짐) | |

위 목록 밖의 의존성을 추가해야 하면 이유를 적고 먼저 묻는다.

---

## 3. 폴더 구조

```
public/
  models/mobilenet_v3_small/   # model.json + 가중치 파일 (로컬 제공, §5.1)
  audio/                       # 대사·효과음 mp3 (DESIGN.md §9 ID와 같은 파일명)
  fonts/                       # 학교안심 글꼴 woff2/otf + OFL.txt
scripts/
  fetch-model.mjs              # 모델 파일을 public/models/에 내려받는 1회용 스크립트
src/
  app/          App.tsx, ScreenContext.tsx(화면 상태), screens.ts(화면 ID)
  screens/      Setup, Egg, Hatch, Home, Feed, Bowls, Study, Quiz, Grow, Work, Album, Teacher
  components/   pet/(PetSvg, parts, expressions), BigButton, SignalButton, BowlCard,
                PlaceCard, CameraView, Countdown, ProgressBar, SpeechBubble,
                ScoreStamps, CompareBars, ConfidenceBars, HiddenCorner
  ml/           featureExtractor.ts, classifier.ts, capture.ts, predictLoop.ts, stabilizer.ts
  input/        useSignal.ts(버튼·키보드·소리 통합), voiceTrigger.ts
  audio/        sound.ts(대사·효과음 재생, 음성 합성 대체)
  data/         db.ts(Dexie 스키마), repo.ts, exportImport.ts
  game/         config.ts(규칙 상수), mood.ts, growth.ts, diversity.ts, events.ts
  copy/         lines.ts(DESIGN.md §9 대사 표를 그대로 옮긴 것)
  styles/       tokens.css, global.css, fonts.css
```

---

## 4. 데이터 모델 (Dexie)

```ts
// 하나의 project = 한 반(또는 한 모둠)이 키우는 반려 AI 하나
type Stage = 'egg' | 'baby' | 'learner' | 'grown' | 'worker';
type ClassKey = 'can' | 'pet' | 'paper' | 'empty';   // empty = 빈 자리(시스템 칸)

interface Project {
  id: string;
  title: string;            // 교사가 붙이는 이름 (예: "고1 직업반")
  petName: string | null;   // 부화 후 반이 지은 이름
  stage: Stage;
  dataVersion: number;      // 먹이(사진)가 바뀔 때마다 +1
  createdAt: number;
  settings: Settings;
}

interface Settings {
  displayMode: 'basic' | 'detail';             // 기본 / 자세히
  signal: { button: true; keyboard: boolean; voice: boolean; voiceThreshold: number };
  sound: { voice: boolean; sfx: boolean; volume: number };
  reduceMotion: boolean;                       // 기기 설정 prefers-reduced-motion도 반영
  previewTransitions: boolean;                 // 화면 전환 예고
  teacherPin: string | null;
  unlocked: { hatch: boolean; work: boolean }; // 교사가 여는 단계
}

interface PetClass {                 // 그릇
  projectId: string;
  key: ClassKey;
  label: string;                     // 캔, 페트병, 종이, 빈 자리
  color: string;                     // 토큰 기본값, 교사가 변경 가능
  photo: Blob | null;                // 교사가 찍은 실물 사진(그릇 아이콘)
  order: number;
}

interface Place {                    // 장소 카드
  id: string; projectId: string;
  label: string;                     // 책상, 바닥, 창가, 분리수거함 옆 …
  photo: Blob | null;
  order: number;
}

interface Sample {                   // 먹이 한 장
  id: string; projectId: string;
  classKey: ClassKey;
  placeId: string;
  thumb: Blob;                       // 224×224 JPEG(품질 0.8), 그릇 보기용
  embedding: ArrayBuffer;            // Float32Array(EMBED_DIM)
  batchId: string;                   // 한 번의 연속 촬영 묶음
  createdAt: number;
  removedAt: number | null;          // 빼기(되돌리기 가능하게 소프트 삭제)
}

interface ModelMeta {
  projectId: string;
  trainedAt: number;
  dataVersion: number;               // 학습 당시 project.dataVersion
  classKeys: ClassKey[];             // 출력 순서
  sampleCounts: Record<ClassKey, number>;
  finalLoss: number;
  finalAcc: number;
}

interface TestRun {                  // 퀴즈 한 회차
  id: string; projectId: string;
  round: number;                     // 1, 2, 3 …
  modelTrainedAt: number;            // 어떤 학습 결과로 본 시험인지
  startedAt: number; finishedAt: number | null;
  items: { n: number; predKey: ClassKey; trueKey: ClassKey; confidence: number; correct: boolean }[];
}

interface AlbumEvent {
  id: string; projectId: string;
  type: 'arrived' | 'hatched' | 'first_study' | 'confused_found' | 'quiz' | 'grew' | 'hired';
  at: number;
  thumb: Blob | null;                // 그 순간 카메라 장면 또는 캐릭터 스냅샷
  data: Record<string, unknown>;     // 점수, 사진 수 등
}

interface WorkLog {                  // 근무 모드 하루 기록
  projectId: string; date: string;   // YYYY-MM-DD
  guides: number;                    // 안내 횟수(칭찬 하트)
  greetings: number;                 // 맞이 신호 횟수
  byClass: Record<ClassKey, number>;
}
```

- 학습된 모델 가중치는 `tf.io`로 `indexeddb://pet-{projectId}`에 저장하고, 메타는 `ModelMeta` 테이블에 둔다.
- **퀴즈 장면은 Sample로 저장하지 않는다.** 시험 물건이 학습에 섞이면 시험이 의미가 없어진다.

---

## 5. 머신러닝

### 5.1 특징 추출 모델

- MobileNet v3 small 100 224 feature vector(TF Hub 형식 tfjs graph model, 출력 1024차원)를 쓴다.
- `scripts/fetch-model.mjs`로 모델 파일(model.json + 가중치)을 한 번 내려받아 `public/models/mobilenet_v3_small/`에 넣고, **앱은 로컬 경로에서만 불러온다.** 개발 환경에서 내려받을 수 없으면 TF Hub 주소에서 불러오는 임시 경로를 두되 `TODO(local-model)`로 표시하고 배포 전에 반드시 로컬로 바꾼다.
- 입력: 카메라 화면 가운데를 정사각형으로 잘라 224×224로 줄이고 0~1로 정규화.
- 앱 시작 시 모델을 불러오고 빈 입력으로 한 번 미리 실행(워밍업)한다. 불러오는 동안 알이 천천히 흔들리는 대기 화면.
- 백엔드는 WebGL 우선, 안 되면 wasm 또는 cpu.

### 5.2 먹이 주기(사진 모으기)

- `navigator.mediaDevices.getUserMedia`로 카메라를 연다. 태블릿은 뒤 카메라(`facingMode: 'environment'`)가 기본, 교사 설정에서 앞·뒤 전환.
- 연속 촬영: 3·2·1 카운트다운 후 `BURST_COUNT`장을 `BURST_INTERVAL_MS` 간격으로 찍는다.
- 한 장마다 즉시 특징값(embedding)을 계산해 썸네일과 함께 저장한다. 학습 때는 저장된 특징값만 쓰므로 학습이 빠르다.
- 저장이 끝나면 `project.dataVersion += 1`.
- 개발·준비용으로 교사 설정에 ‘사진 파일로 먹이 주기’(이미지 파일 여러 장 선택)를 둔다.

### 5.3 학습(공부하기)

- 분류기: `dense(128, relu)` → `dropout(0.2)` → `dense(클래스 수, softmax)`. 입력은 저장된 특징값.
- `adam(0.001)`, `categoricalCrossentropy`, `epochs = TRAIN_EPOCHS`, `batchSize = 16`, `shuffle: true`.
- `onEpochEnd`로 진행 막대를 갱신한다.
- 학습 가능 조건: 먹이 그릇 3개와 빈 자리 칸이 각각 `MIN_SAMPLES_TO_TRAIN`장 이상. 먹이 그릇이 모자라면 학습하지 않고 `hungry` 표정 + 부족한 그릇을 말풍선 아이콘으로 보여 준다. 빈 자리 칸은 교사가 `S0_setup`에서 찍는 것이므로, 모자라면 학생 화면 대신 상단 띠에 작은 교사용 알림을 띄운다.
- 칸 사이 사진 수가 2배 넘게 차이 나면 ‘자세히’ 모드에서 안내한다(학습은 진행).
- 끝나면 모델 저장, `ModelMeta` 기록, 첫 학습이면 `AlbumEvent('first_study')`, 단계가 `baby`면 `learner`로.
- 학습 중에는 다른 화면으로 갈 수 없다(집 버튼 비활성). 메모리 누수를 막기 위해 텐서는 `tf.tidy`와 `dispose`로 정리한다.

### 5.4 예측과 안정화

- 실시간 예측은 초당 `PREDICT_FPS`번으로 제한한다(`requestAnimationFrame` + 시간 간격 확인).
- 최근 5번 예측의 확률을 평균 내고, 같은 칸이 `STABLE_MS` 이상 1위이고 확신도가 `CONF_THRESHOLD` 이상이면 ‘안정된 답’으로 본다.
- 1위가 `empty`면 ‘아무것도 없음’으로 처리한다.
- 확신도가 `CONF_THRESHOLD`보다 낮게 `STABLE_MS` 이상 이어지면 `confused` 표정.

### 5.5 퀴즈(시험)

```
회차 시작 → round = 지난 회차 + 1
문제 n (1~TEST_ITEMS):
  "물건을 보여 주세요" → 실시간 예측
  안정된 답이 나오거나 신호(정답 맞혀 봐)가 들어오면 → 화면 멈춤, 답 발표(그림+음성)
  맞아요 → correct=true
  틀려요 → 정답 그릇 고르기 → correct=false
  기록 저장 → 다음 문제
회차 끝 → 점수(맞힌 수), 지난 회차와 비교, AlbumEvent('quiz'), 성장 판정(§6.3)
```

첫 오답이 나온 회차에는 `AlbumEvent('confused_found')`를 한 번 남긴다.

### 5.6 근무 모드

- 안정된 답이 `empty`가 아닌 칸으로 바뀌면 큰 그림 + `work_{can|pet|paper}` 음성, `WorkLog.guides += 1`.
- 같은 물건으로 반복 안내하지 않도록, 안내 뒤 `WORK_COOLDOWN_MS`가 지나거나 `empty`를 한 번 본 뒤에만 다음 안내.
- 맞이 신호가 들어오면 `work_greet` 음성 + `WorkLog.greetings += 1`.
- 처음 근무 모드를 열면 단계 `worker`, `AlbumEvent('hired')`.

---

## 6. 게임 규칙

### 6.1 규칙 상수 (`src/game/config.ts`)

| 이름 | 기본값 | 뜻 |
|---|---|---|
| `TARGET_SAMPLES` | 30 | 그릇이 ‘가득’으로 보이는 사진 수 |
| `MIN_SAMPLES_TO_TRAIN` | 10 | 칸마다 학습에 필요한 최소 사진 수 |
| `BURST_COUNT` | 20 | 한 번 먹이 줄 때 찍는 장수 |
| `BURST_INTERVAL_MS` | 150 | 연속 촬영 간격 |
| `TRAIN_EPOCHS` | 15 | 학습 반복 수 |
| `PREDICT_FPS` | 8 | 초당 예측 횟수 |
| `CONF_THRESHOLD` | 0.6 | 이보다 낮으면 헷갈림 |
| `STABLE_MS` | 800 | 안정된 답으로 보는 시간 |
| `TEST_ITEMS` | 6 | 퀴즈 문제 수 |
| `DIVERSITY_MIN_PLACES` | 2 | 그릇마다 있어야 하는 장소 수 |
| `DIVERSITY_MIN_PER_PLACE` | 5 | 장소 하나로 인정되는 최소 사진 수 |
| `GROW_MIN_SCORE` | 5 | 점수가 오르지 않아도 이 이상이면 성장 인정 |
| `PICKY_RATIO` | 0.9 | 한 그릇 사진의 90% 이상이 한 장소면 ‘편식’ 대사 |
| `WORK_COOLDOWN_MS` | 3000 | 근무 모드 안내 간격 |
| `VOICE_SUSTAIN_MS` | 300 | 소리 신호로 인정되는 지속 시간 |
| `VOICE_COOLDOWN_MS` | 2000 | 소리 신호 사이 최소 간격 |
| `TEACHER_LONGPRESS_MS` | 2000 | 숨은 모서리 길게 누르기 |
| `TRANSITION_PREVIEW_MS` | 1500 | 화면 전환 예고 시간 |

교사 설정에서 `CONF_THRESHOLD`, `TARGET_SAMPLES`, `BURST_COUNT`, 소리 신호 감도는 바꿀 수 있다.

### 6.2 표정 결정 (`mood.ts`)

우선순위가 높은 것부터 확인한다. 순수 함수로 만들고 단위 테스트를 단다.

1. 학습 중 → `studying`
2. 근무 모드 → `working`
3. 3초 안에 일어난 순간 이벤트(퀴즈 맞음·성장 → `happy`, 퀴즈 틀림·확신도 낮음 → `confused`)
4. 먹이 그릇 중 하나라도 `MIN_SAMPLES_TO_TRAIN` 미만 → `hungry`
5. `project.dataVersion > ModelMeta.dataVersion`(새 먹이를 아직 안 배움) 또는 모델 없음 → `sleepy`
6. 그 밖 → `idle`

### 6.3 성장 판정 (`growth.ts`)

- `egg → baby`: 교사가 부화를 열고(`unlocked.hatch`), 신호 3번으로 깨어난 뒤 이름이 정해지면.
- `baby → learner`: 첫 학습 완료.
- `learner → grown`: 퀴즈 회차가 끝났을 때 아래를 **모두** 만족하면.
  - 먹이 그릇 3개 모두 `DIVERSITY_MIN_PLACES`개 이상의 장소에서 각각 `DIVERSITY_MIN_PER_PLACE`장 이상(`diversity.ts`)
  - 이번 점수 > 지난 회차 점수, 또는 이번 점수 ≥ `GROW_MIN_SCORE`
  - 이번 퀴즈가 마지막 학습 이후에 본 것
- `grown → worker`: 교사가 근무 모드를 열고(`unlocked.work`) 처음 근무 모드에 들어가면.
- 단계는 내려가지 않는다. 성장하면 `S8_grow` 화면과 `AlbumEvent('grew')`.

### 6.4 편식 안내

먹이를 줄 때 그 그릇 사진의 `PICKY_RATIO` 이상이 한 장소라면 `feed_picky` 대사를 한 번 들려준다(같은 날 그릇마다 한 번). 이 안내는 4차시에서 일부러 한 장소만 쓰는 수업 의도를 방해하지 않도록 교사 설정에서 끌 수 있다.

---

## 7. 입력: 신호 (`useSignal.ts`)

세 입력을 하나의 `onSignal()` 이벤트로 합친다. 화면은 신호의 출처를 몰라도 된다.

| 입력 | 처리 |
|---|---|
| 신호 버튼 | `pointerdown`에서 바로 신호(손을 떼는 시점이 아님). 연속 입력은 400ms 안에 한 번만 |
| 키보드·스위치 | 스페이스, 엔터. 신호를 쓰는 화면에서만 받음. 입력란에 글자를 쓰는 중에는 무시 |
| 소리 | Web Audio `AnalyserNode`로 소리 크기(RMS)를 재서 `voiceThreshold`를 `VOICE_SUSTAIN_MS` 이상 넘으면 신호. `VOICE_COOLDOWN_MS` 동안 재입력 무시. 소리는 저장·전송하지 않음 |

- 소리 신호는 교사가 켰을 때만 마이크를 연다. 신호 버튼 둘레에 현재 소리 크기 링을 그린다.
- 교사 설정에 ‘소리 신호 맞추기’: 조용히 있을 때와 학생이 소리 낼 때를 각각 3초 재서 그 사이 값을 기준으로 제안.
- 신호를 받으면 0.2초 안에 버튼 눌림 모양 + `sfx_signal` + 다음 동작.

---

## 8. 소리 (`sound.ts`)

- `play(lineId)`: `public/audio/{lineId}.mp3`가 있으면 재생, 없으면 `speechSynthesis`(ko-KR)로 `copy/lines.ts`의 같은 문장을 읽는다.
- 대사는 한 번에 하나만. 새 대사가 오면 앞 대사를 멈춘다(효과음은 겹쳐도 됨).
- 브라우저 자동재생 제한 때문에 첫 터치(준비 화면 시작 버튼)에서 오디오를 깨운다.
- 볼륨·끄기는 `Settings.sound`를 따른다.

---

## 9. 화면 구현 메모

화면 ID와 배치는 DESIGN.md §6~7을 따른다.

- **화면 상태**: `{ screen: ScreenId, params }`. 활동 화면 사이 이동 때 `previewTransitions`가 켜져 있으면 다음 활동 예고(`next_*`)를 먼저 보여 준다.
- **첫 실행**: `S0_setup`으로 시작. 준비가 끝나면 `S1_egg`.
- **숨은 모서리**(`HiddenCorner`): 오른쪽 위 120×96, `TEACHER_LONGPRESS_MS` 길게 누르면 `S11_teacher`(PIN이 있으면 숫자 입력). 근무 모드에서 나가는 유일한 길.
- **반려 AI**(`PetSvg`): `stage`와 `mood`를 받아 SVG를 그린다. 부품(팔, 모자, 배지, 안테나 길이)은 단계로, 눈·입은 표정으로 바꾼다. 깜빡임·숨쉬기는 CSS 애니메이션이고 `reduceMotion`이면 끈다.
- **카메라 화면**: 정사각형으로 잘라 보여 준다. 화면에 보이는 영역 = AI가 보는 영역.
- **화면이 꺼지지 않게**: 근무 모드에서는 Screen Wake Lock API를 지원하면 쓴다.
- **전체 화면**: 근무 모드 진입 시 Fullscreen API 요청(거부돼도 동작).

---

## 10. 저장·내보내기

- 앱을 열면 마지막 project를 불러온다. 여러 project(반·모둠)를 교사 설정에서 고를 수 있다.
- 내보내기: project 하나를 JSON 파일 한 개로. 사진·특징값은 base64, 모델 가중치는 `tf.io.withSaveHandler`로 받은 내용을 base64로 넣는다. 파일 이름 `우리반AI_{petName 또는 title}_{YYYYMMDD}.json`.
- 가져오기: 새 project로 추가(기존 덮어쓰지 않음).
- 초기화: 교사 설정에서 project 삭제(두 번 확인).
- 사진 빼기는 `removedAt`으로 숨기고 ‘되돌리기’를 준다. 다음 학습 때 빠진 사진은 쓰지 않는다. 30일 지난 빠진 사진은 정리한다.

---

## 11. 개인정보·보안

- 런타임 외부 요청 0건: 모델, 글꼴, 소리 모두 로컬 파일. 분석 도구, 외부 CDN 글꼴·스크립트를 넣지 않는다.
- 카메라 화면·사진·소리는 기기 밖으로 보내지 않는다. 콘솔 로그에 이미지 데이터를 찍지 않는다.
- 학생 이름을 입력받는 곳은 없다(반려 AI 이름과 project 제목만).
- 수업 안내: 사진은 손만 나오게 찍고 얼굴은 넣지 않는다. 얼굴이 찍히면 그릇 보기에서 뺀다.
- 공개 배포 전에 `edu-app-launch` 점검(보안 헤더, 개인정보처리방침, 링크 미리보기)을 거친다.

---

## 12. 마일스톤과 수용 기준

한 번에 한 마일스톤씩 진행하고, 끝날 때 수용 기준을 하나씩 확인해 보고한다.

### M1. 학습 핵심 (꾸밈 없는 화면)
- 카메라 열기, 칸 4개(캔·페트병·종이·빈 자리)에 연속 촬영으로 사진 모으기
- 특징값 저장, 학습, 실시간 예측, 확신도 막대 표시
- IndexedDB 저장
- **수용 기준**: 새로고침해도 사진과 학습 결과가 남는다 / 칸마다 30장일 때 학습이 실제 태블릿에서 30초 안에 끝난다 / 예측이 초당 5번 이상 갱신된다 / 외부 요청이 없다(개발자 도구 네트워크 탭으로 확인)

### M2. 반려 AI 화면
- 토큰·글꼴 적용, `PetSvg`(단계 5개 × 표정 7개), `S1`~`S6` 화면, 대사·효과음, 표시 모드(기본·자세히)
- **수용 기준**: DESIGN.md 화면 배치와 위치가 같다 / 표정 규칙(§6.2) 단위 테스트 통과 / 소리를 꺼도 화면만으로 먹이 주기·공부하기가 된다

### M3. 퀴즈·기록·성장
- `S7_quiz`, 회차 기록, 전·후 비교, 다양성 계산, 성장 판정, `S8_grow`, 앨범 이벤트 기록
- **수용 기준**: 성장 판정(§6.3)·다양성 단위 테스트 통과 / 퀴즈 장면이 먹이로 저장되지 않는다 / 한 장소 사진만으로는 점수가 올라도 `grown`이 되지 않는다

### M4. 신호 입력·근무 모드·선생님 설정
- `useSignal`(버튼·키보드·소리), 소리 신호 맞추기, `S9_work`, `S10_album`, `S11_teacher`, 내보내기·가져오기, 감각 설정
- **수용 기준**: 세 입력 모두 같은 동작을 일으킨다 / 근무 모드에서 같은 물건을 계속 들고 있어도 3초 안에 반복 안내하지 않는다 / 내보낸 파일을 다른 기기에서 가져와 같은 결과가 나온다 / 학생 화면에서 실수로 설정에 들어갈 수 없다

### M5. 마무리
- 오프라인(PWA), 실제 태블릿 성능 점검, 접근성 점검표(DESIGN.md §12), `edu-app-launch` 점검, Vercel 배포
- **수용 기준**: 와이파이를 끊어도 앱이 열리고 학습·예측이 된다 / 접근성 점검표 모두 통과

---

## 13. 테스트

- 단위 테스트(vitest): `mood.ts`, `growth.ts`, `diversity.ts`, 예측 안정화(`stabilizer.ts`), 신호 쿨다운, 내보내기·가져오기 왕복.
- 수동 점검(실제 기기): 카메라 앞·뒤 전환, 연속 촬영 중 화면 멈춤 없음, 학습 시간, 근무 모드 1시간 연속 실행 시 메모리 증가 없음(`tf.memory()` 확인용 숨은 개발 표시).
- 확인 기기: 수업에 쓸 태블릿(미정, §14), 노트북 크롬.

---

## 14. 정해지지 않은 것

- [ ] 수업에 쓸 기기(태블릿 종류 또는 노트북)
- [ ] 다 학생이 실제로 쓸 신호 방식(버튼 대리 입력, 스위치, 소리)과 감도 — 특수교육지도사와 확인
- [ ] 쓸 학교안심 글꼴 종류와 파일
- [ ] 학교 분리수거함 색(그릇 색 기본값)
- [ ] 음성 파일을 미리 녹음·생성할지, 브라우저 음성으로 갈지
- [ ] 앱 정식 이름, 공개 배포 여부
