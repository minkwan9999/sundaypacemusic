# Kaggle 음악 생성 작업: 현황과 자동화 가능성

- 보고: Mac 세션 (sundaypacemusic-e0)
- 확인 시각: 2026-11-18 23:00 JST 무렵
- 점검 방식: 읽기만 함. 노트북 실행, 파일 다운로드·삭제, 라이브 송출 변경은 하지 않음

## 1. 지금까지 만든 방식

### 사용한 노트북과 모델
- 노트북: `SundayPace_Generator_Kaggle_v2.12.ipynb` (Mac의 `SundayPaceMusic` 폴더에 v2.4~v2.12 보관. 최신은 v2.12)
- Kaggle 노트북 링크: 확인 불가 (노트북 파일에 Kaggle 노트북 ID가 기록돼 있지 않음)
- 모델: **ACE-Step 1.5** (오픈소스 음악 생성 모델)
  - 코드: https://github.com/ace-step/ACE-Step-1.5
  - 가중치: https://huggingface.co/ACE-Step/Ace-Step1.5
  - 사용한 설정: `acestep-v15-turbo`, 8스텝, LM 끔. A/B 청취 시험에서 기본값이 가장 좋았다고 기록돼 있음
- Kaggle 실행 환경: GPU T4 x2 (실제 사용은 1장), 인터넷 켬. P100은 이 PyTorch 버전을 지원하지 않음

### 실행 순서 (노트북 설명 기준)
1. (최초 1회) Kaggle 가입과 전화번호 인증. 인증하지 않으면 GPU와 인터넷을 쓸 수 없음
2. 노트북을 열 때마다 Session options에서 GPU T4 x2와 Internet On 설정
3. 셀 1: GPU 확인
4. 셀 2: ACE-Step 1.5 설치, T4용 float16→float32 패치, 모델 다운로드 (세션마다 10~15분)
5. 셀 3B: 배치 생성. 예) `PRESET=xmas FROM=16 TO=20 uv run python batch_bgm.py` 로 5곡씩 생성
6. 셀 3C: 노트북 안에서 바로 듣고, 마음에 안 드는 곡은 번호를 지정해 다시 생성
7. 셀 6: 결과 폴더를 zip으로 묶음 → 브라우저에서 클릭해 다운로드 (세션이 끝나면 사라짐)
8. Mac: zip을 풀어 mp3를 에피소드 폴더에 넣고 `sundaypace_v1.22.py` 실행 → 믹스, 타임스탬프, 썸네일, 영상, YouTube 제목·설명을 한 번에 생성

### 결과물 (크리스마스 재즈 에피소드, 폴더 `SundayPaceMusic/001`)
- 곡: mp3 35곡 (`01_taxi-full-of-tinsel.mp3` ~ `35_home-by-candlelight.mp3`), 창작 영어 가사가 있는 1950년대풍 보컬 재즈
- Kaggle에서 받은 zip 7개 (`xmas_20261114_1829.zip` 등). 5곡씩 생성했고, 묶음마다 곡 구성을 적은 `tracks.json`이 있음
- Mac에서 만든 결과 (`001/_output`):
  - `mix.mp3` (약 291MB)
  - `video.mp4` (약 369MB). 지금 24시간 라이브에서 송출 중인 파일
  - `thumbnail_xmas_en.jpg`, `thumbnail_xmas_ko.jpg`, `timestamps.txt`, `youtube.txt`
- 생성 날짜: 2026-11-14 (zip 이름 기준), 영상 제작 2026-11-15

## 2. Mac에 있는 도구

| 항목 | 상태 |
|---|---|
| `kaggle` CLI | 설치 안 됨 |
| Kaggle 인증 파일 (`~/.kaggle/kaggle.json`, `~/.kaggle/access_token`, `~/.config/kaggle/kaggle.json`) | 없음 |
| Kaggle 인증 환경변수 | 없음 |
| Python | 3.9.6 (macOS 기본 `/usr/bin/python3`) |
| ffmpeg / ffprobe | 9.0.2, `~/streamtool/` 에 있음 (시스템 PATH에는 없음) |
| 브라우저 | Google Chrome, Safari |
| Mac 사양 | macOS 12.7.6, Intel Core i5 2코어, RAM 8GB, NVIDIA GPU 없음 |

→ 이 Mac 사양으로는 ACE-Step을 직접 돌려 곡을 만들 수 없음. 곡 생성은 Kaggle 같은 GPU 환경이 계속 필요함.

## 3. 이 Mac의 Claude Code로 가능한 방식

### (a) Kaggle CLI로 노트북 실행과 결과 다운로드 (추천)
흐름: 노트북을 배치 실행용으로 정리 → `kaggle kernels push`로 GPU·인터넷을 켜고 실행 → `kaggle kernels status`로 완료 확인 → `kaggle kernels output`으로 mp3·zip을 Mac에 받기 → `sundaypace.py`로 영상 제작.
- 장점
  - 브라우저 클릭 없이 처음부터 끝까지 명령어로 처리 가능
  - 세션이 끝나 결과가 사라질 걱정이 없음 (실행 결과가 Output으로 저장됨)
  - 곡 번호(FROM/TO)나 프리셋만 바꿔 반복 실행하기 쉬움
- 단점
  - 지금 노트북은 사람이 셀을 골라 누르는 방식임. 셀 3A(웹 UI)는 끝나지 않아서 그대로는 배치 실행이 안 됨 → 3A를 빼고 3B 설정을 변수로 받는 실행용 버전이 필요
  - 실행할 때마다 설치와 모델 다운로드에 10~15분이 걸림
  - 중간에 듣고 고르는 단계(3C)가 없음. 받은 뒤 Mac에서 듣고 다시 생성할 번호를 정해야 함
  - GPU 사용량이 주 30시간 내외로 제한됨
- 사용자 승인이 필요한 단계
  1. Mac에 `kaggle` CLI 설치 (pip)
  2. Kaggle 사이트에서 API 토큰을 직접 만들어 Mac에 넣기 (토큰은 사용자가 직접 넣고, Claude는 내용을 보지 않음)
  3. 노트북을 Kaggle에 올리거나 덮어쓰기
  4. 실행할 때마다 GPU 시간 사용
  5. 결과 다운로드

### (b) 브라우저 조작 (Claude in Chrome)
흐름: 사용자의 Chrome에서 Kaggle 노트북을 열고 셀을 차례로 실행 → 끝나면 zip 다운로드 링크 클릭.
- 장점
  - 지금 쓰는 노트북을 거의 그대로 쓸 수 있음
  - 이미 로그인된 브라우저를 쓰므로 API 토큰이 필요 없음
- 단점
  - 화면이 바뀌면 깨지기 쉽고 느림
  - 20분 넘게 아무 조작이 없으면 세션이 꺼질 수 있어 오래 지켜봐야 함
  - Mac이 켜져 있고 Chrome이 열려 있어야 함 (라이브 송출로 CPU를 이미 약 75% 쓰는 중)
  - Claude in Chrome 확장 설치와 연결 여부: 확인 불가
- 사용자 승인이 필요한 단계: 확장 설치와 Kaggle 사이트 접근 허용, GPU 세션 시작, 다운로드

### (c) 기타
- **Mac 쪽 후반 작업 자동화 (지금 바로 가능):** zip 풀기 → 곡 순서 정리 → `sundaypace.py` 실행 → 결과 확인까지는 Kaggle 없이 이 Mac의 Claude Code가 대신할 수 있음. 새 파일만 만들고 송출 중인 `001/_output/video.mp4`는 건드리지 않는 방식으로 진행 가능
- **다른 GPU 환경 (Colab, 유료 GPU 서버 등):** 가능은 하지만 새 계정과 비용이 필요함. 이 보고에서는 조사하지 않음

### 권장 순서
1. (c)의 후반 작업 자동화부터 시작 (승인만 받으면 바로 가능)
2. 이어서 (a): 배치 실행용 노트북을 만들고 CLI로 시험 실행 1회 (사용자가 토큰을 준비한 뒤)
3. (b)는 (a)가 막힐 때 쓰는 대안

## 4. 모델 라이선스
- **ACE-Step 1.5 코드:** MIT 라이선스 (GitHub 저장소 표기)
- **모델 가중치:** MIT 라이선스 (Hugging Face 모델 카드 표기)
- **상업적 사용:** 모델 카드에 "You can strictly use the generated music for commercial purposes."라고 적혀 있음 → 생성한 음악의 상업적 사용은 허용된다고 판단됨
- **저작권 관련 주의 (저장소 문구):** 기존 곡과 스타일이 비슷해 의도치 않게 저작권을 침해할 수 있다고 경고함. 생성물이 독창적인지 확인하고 AI를 썼다는 사실을 밝히라고 권함
- **YouTube 수익화 가능 여부:** 확인 불가. 모델 라이선스상으로는 막는 내용이 없음. 다만 수익화 승인은 YouTube 정책(반복적이거나 대량 생산된 콘텐츠 기준 등)과 채널 심사에 달려 있어 여기서 판단할 수 없음
- 법률 검토를 거친 결론은 아님

## 5. 확인하지 못한 것
- Kaggle 노트북 링크와 Kaggle 계정 쪽 상태 (남은 GPU 시간 등): 확인 불가
- Claude in Chrome 확장 설치와 연결 여부: 확인 불가
- YouTube 수익화 정책 적용 여부: 확인 불가
- `claude/sundaypace-builder.py`와 v1.22의 동기화 상태 (작업 기록의 남은 할 일): 이번 점검 범위가 아님

## 6. 사용자가 할 일
1. 원하는 방식을 고르기: (a) CLI 자동화 / (b) 브라우저 조작 / (c) Mac 후반 작업 자동화만
2. (a)를 고르면: Kaggle 사이트 Settings → API → 토큰을 만들고 직접 Mac에 넣기. 토큰 내용은 채팅이나 저장소에 붙여 넣지 않기
3. YouTube 수익화를 계획한다면 YouTube 파트너 프로그램 정책에서 AI 생성 음악과 반복 콘텐츠 기준 확인
