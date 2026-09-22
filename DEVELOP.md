# 개발·배포 안내 (for developers)

사용자용 소개는 [README.md](./README.md)를 보세요. 이 문서는 개발·운영 전용입니다.

## 원칙

- 백엔드 없음 — 전부 정적 파일. 외부 CDN 금지, 라이브러리·모델 전부 셀프호스팅
- 최초 로드 후 오프라인 동작
- 코드·README에 계정명·절대 URL 하드코딩 금지 (조직 이전 대비, 전부 상대경로)
- 디자인은 자매 서비스(파이보 랩)와 동일 — 공용 규격은 [design/](./design/) 참고

## 실행

`file://` 로는 열 수 없습니다. 반드시 http 로 서빙하세요.

```bash
npx http-server -p 8080          # 또는
python3 -m http.server 8080
```

### MIME 주의

서버가 아래 타입을 내보내야 합니다. 틀리면 wasm 스트리밍 컴파일이 실패합니다.

| 확장자 | MIME |
|---|---|
| `.wasm` | `application/wasm` |
| `.task` / `.tflite` | `application/octet-stream` |
| `.mjs` | `text/javascript` |

Go 서버(`pibo-server` 계열) 사용 시 `mime.AddExtensionType` 로 명시 등록하세요.

## 구조

```
index.html / test.html / storage.html
css/
  maker-ui.css       공용 디자인 킷 (design/maker-ui.css 복사본 — 수정 금지)
  app.css            이 서비스 전용
lib/
  nav.js             헤더/탭/전체화면
  landmarker.js      MediaPipe 로드/전환/추론 (GPU 실패 시 CPU 폴백)
  sound.js           마이크 + YAMNet 소리 분류 (16kHz, 250ms 주기, 521차원, 한 창 15600 샘플)
  features.js        특징 벡터 전처리 + 내장 신호 (손 63 / 얼굴 52 / 포즈 75 / 소리 521)
  trainer.js         TF.js 분류기 학습 (Dense32-Dropout-Softmax, CPU 백엔드)
  store.js           IndexedDB 저장 + zip 내보내기/불러오기
  bgfx.js            배경 놀이 (셀피 세그멘테이션)
  i18n.js            한/영 토글 (파이보 랩과 같은 방식)
  tour.js            튜토리얼
  learn.js / exam.js / storage_page.js   페이지 로직
vendor/
  tasks-vision/      @mediapipe/tasks-vision wasm + vision_bundle.mjs
  tasks-audio/       @mediapipe/tasks-audio wasm + audio_bundle.mjs (소리)
  tfjs/              @tensorflow/tfjs tf.min.js
models/              hand / face / pose_lite .task + selfie_segmenter(배경) ·
                     yamnet(소리) .tflite
assets/fonts/        Pretendard (셀프호스팅)
assets/img/          캐릭터·로고·앱 아이콘
docs/                README 용 화면 캡처
design/              공용 디자인 킷 원본 (maker-ui.css + 미리보기)
                     css/maker-ui.css 는 이 파일의 복사본이다.
                     공통 규격을 바꿀 때는 design/ 을 고치고 css/ 로 복사한다.
                     자매 서비스(teach-lab 등)에도 같은 파일이 들어 있다.
```

## 소리: 창 길이를 반드시 15600 샘플로

YAMNet 한 창은 **15600 샘플(0.975초)** 이다. 1초(16000)를 통째로 넣으면
MediaPipe 가 창을 둘로 쪼개는데, 둘째 창은 400샘플만 진짜이고 나머지는 0으로
채운 꼬리라 **늘 "Silence" 가 1등**으로 나온다. 예전 코드는
`results[results.length - 1]` 로 **그 꼬리 창을 골라 썼다.**

```
220Hz 순음을 넣고 잰 것 (헤드리스 크로미움)
  고치기 전   Silence   0.148      ← 0으로 채운 꼬리 창
  고친 뒤     Sine wave 0.996      ← 진짜 내용
```

그래서 링 버퍼를 `0.975 × ctx.sampleRate` 로 잡고 `results[0]` 을 쓴다
(`lib/sound.js`). 샘플레이트를 16kHz 로 못 맞추는 기기가 있어 초 단위로 계산한다.
이 창 길이는 `features.js` 의 내장 소리 신호(박수·휘파람·말소리) 판정에도
그대로 영향을 준다 — 꼬리 창을 쓰면 점수가 전부 0 근처라 아무것도 안 잡힌다.

> 같은 코드가 **teach-lab** 의 `lib/sound.js` 에도 있다. 한쪽을 고치면 저쪽도 본다.

## 벤더 파일 갱신

```bash
npm i @mediapipe/tasks-vision @mediapipe/tasks-audio @tensorflow/tfjs
cp node_modules/@mediapipe/tasks-vision/wasm/*        vendor/tasks-vision/
cp node_modules/@mediapipe/tasks-vision/vision_bundle.mjs vendor/tasks-vision/
cp node_modules/@mediapipe/tasks-audio/wasm/audio_wasm*   vendor/tasks-audio/
cp node_modules/@mediapipe/tasks-audio/audio_bundle.mjs   vendor/tasks-audio/
cp node_modules/@tensorflow/tfjs/dist/tf.min.js       vendor/tfjs/
```

`.task`/`.tflite` 모델은 MediaPipe 공식 모델 저장소에서 받아 `models/` 에 둡니다
(hand_landmarker, face_landmarker, pose_landmarker_lite — 포즈는 lite 고정,
selfie_segmenter, yamnet).
`node_modules` 는 커밋하지 않고, 벤더 파일은 커밋합니다.

## 배포 (파이보 랩과 동일)

- `main` 에 작업 → GitHub Pages 테스트 → 통과하면 `release` 브랜치 머지 → Cloudflare 자동 배포
- Cloudflare 는 Workers 방식: `wrangler.toml` + `.assetsignore` (잠금 없음 — 자산만 서빙)
- 배포 명령: `npx wrangler deploy`
- 잠금이 다시 필요하면 파이보 랩의 시험 기간 커밋처럼
  `main`/`run_worker_first` + Basic Auth 워커(`src/index.js`)를 되살립니다
- 캐시 버스팅: 코드 파일만 `?v=N`. 모델·wasm 파일에는 붙이지 않습니다
- 모든 자산 경로는 상대경로 — 하위 경로(`/sense-lab/`) 서빙에서도 동작합니다

## 오프라인 exe (파이보 랩과 동일)

`v*` 태그를 푸시하면 GitHub Actions 가 사이트 전체를 담은 단일
`SenseLab.exe` 를 빌드해 Release 에 첨부합니다 (인터넷 없이 동작).

```bash
git tag v0.1.0 && git push origin v0.1.0
```

- 구성: `.github/workflows/build-exe.yml` + `tools/portable/` (Go embed 서버)
- 서버는 `.wasm`/`.task`/`.tflite`/`.mjs` MIME 을 명시 등록합니다
- Actions 탭에서 workflow_dispatch 로 수동 빌드도 가능합니다
