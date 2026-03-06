# MSX2 MV2 Video Encoder (Rust Native)

## 개요
이 프로그램은 일반적인 동영상 파일(MP4, AVI 등)을 MSX2의 **MV2 포맷**으로 변환하는 고성능 비디오 인코더입니다. Rust로 작성되었으며, WebGPU를 통한 컴퓨트 셰이더(Compute Shader) 가속과 FFmpeg 기반의 비디오/오디오 전처리를 지원합니다.

## 필수 요구 사항
- **FFmpeg**: 동영상 렌더링 및 오디오(PCM/MP3) 전처리를 위해 시스템 환경 변수(PATH)에 등록되어 있어야 합니다.
- **지원되는 GPU (선택)**: `--shader` 옵션 사용 시 WebGPU/Vulkan 기반 셰이더 가속을 지원하는 GPU가 필요합니다.
- **NVIDIA GPU (선택)**: `--nv` 옵션 사용 시 NVENC 하드웨어 가속 인코딩을 위해 필요합니다.

---

## 사용법
명령줄에서 다음과 같은 형식으로 실행합니다.

```bash
msx_encoder [OPTIONS] <INPUT> <OUTPUT>
```

### 필수 인수
- `<INPUT>`: 변환할 원본 동영상 파일의 경로 (예: `input.mp4`)
- `<OUTPUT>`: 저장할 MV2 파일의 경로 (예: `output.MV2`)

---

### 주요 옵션 (Options)

#### 디더링 (Dithering)
- `--dither <DITHER>`: 영상 렌더링 시 사용할 디더링 알고리즘을 선택합니다. [기본값: `none`]
  - **지원 모드**: 
    - `none`: 디더링 없음
    - `fs`: Floyd-Steinberg (에러 확산, 자연스러운 혼합)
    - `sfs`: Stucki
    - `jjn`: Jarvis-Judice-Ninke
    - `bayer4`: 4x4 베이어 매트릭스 (고전적인 체크무늬 패턴, 매우 빠름)
    - `bayer8`: 8x8 베이어 매트릭스

#### 화면 비율 및 크롭 (Aspect Ratio & Crop)
- `--aspect <ASPECT>`: 화면 비율을 조정하는 방식을 선택합니다. [기본값: `pad`]
  - `pad`: 원본 비율을 유지하며, 남는 상하좌우 공간을 검은색으로 채웁니다. (레터박스)
  - `crop`: 원본 비율을 유지하면서 MSX 화면(256x192)을 꽉 채우도록 넘치는 부분을 잘라냅니다.
  - `force`: 비율을 무시하고 화면을 강제로 늘리거나 압축하여 256x192로 맞춥니다.
- `--crop-up <CROP_UP>`: 상단 크롭 세부 조정 (0.0 ~ 1.0) [기본값: `0.0`]
- `--crop-left <CROP_LEFT>`: 좌측 크롭 세부 조정 (0.0 ~ 1.0) [기본값: `0.0`]
- `--skip-prescale`: FFmpeg 전처리 단계에서 스케일링 및 패딩 작업을 건너뜁니다. (이미 256x192로 맞춰진 영상에 유용)

#### 성능 가속 및 최적화
- `--shader`: GPU 컴퓨트 셰이더를 사용하여 초고속으로 팔레트를 추출합니다. (사용 권장)
- `--nv`: FFmpeg 전처리 시 NVIDIA 하드웨어 인코더(NVENC)를 사용하여 작업 속도를 크게 높입니다. (NVIDIA 그래픽카드 전용)
- `--workers <WORKERS>`: CPU 병렬 처리에 사용할 스레드 개수입니다. [기본값: `1`] (시스템의 코어 수에 맞게 올리는 것을 권장합니다.)

#### 고급 품질 설정 (Advanced)
- `--luma` / `--luma false`: Luma-Guided(밝기 기반) GPU 팔레트 추출을 활성화하여 명암 대비 색상 정확도를 극대화합니다. [기본값: `true`]
- `--shift` / `--shift false`: 팔레트 계산 시 기본 8픽셀 단위 블록 외에 4픽셀 어긋난 중간 블록을 추가로 분석하여 디테일과 경계선 품질을 향상시킵니다. [기본값: `false`]
- `--complex` / `--complex false`: 원본 버전의 복잡한 수학적 "선 투영(Segment Projection)" 블록 매칭 알고리즘을 사용합니다. 그라데이션 렌더링 품질이 미세하게 향상되지만 CPU 연산량이 크게 증가합니다. [기본값: `false`]
- `--anchor <ANCHOR>`: 프레임 당 초기 기준(Seed)으로 잡을 앵커 색상의 최대 개수입니다. (1~15) [기본값: `6`]

#### 기타 옵션
- `-s, --ss <START>`: 인코딩 시작 시간을 지정합니다. (예: `00:00:10` 또는 `10`)
- `-t, --to <END>`: 인코딩 종료 시간을 지정합니다. (예: `00:01:00` 또는 `60`)
- `-v, --verbose`: 세부 진행 상황과 FFmpeg 처리 로그를 화면에 출력합니다.

---

## 실행 예시

**1. 빠르고 안정적인 기본 변환 (Floyd-Steinberg 디더링, GPU 셰이더 활성화)**
```bash
cargo run --release -- input.mp4 output.MV2 --dither fs --shader --workers 16
```

**2. 초고속 변환 (Bayer4 디더링, NVIDIA 하드웨어 가속 사용)**
```bash
cargo run --release -- input.mp4 output.MV2 --dither bayer4 --shader --workers 16 --nv
```

**3. 최고 화질 (복잡한 매칭 연산, Shift 분석 추가, Floyd-Steinberg 디더링)**
```bash
cargo run --release -- input.mp4 output.MV2 --dither fs --shader --workers 16 --complex --shift
```

**4. 꽉 찬 화면으로 잘라서 변환 (Aspect: Crop)**
```bash
cargo run --release -- input.mp4 output.MV2 --aspect crop --dither fs --shader --workers 16
```

**5. 특정 구간만 변환 (10초부터 20초까지 10초 분량)**
```bash
cargo run --release -- input.mp4 output.MV2 -s 00:00:10 -t 00:00:20 --dither fs --shader --workers 16
```
