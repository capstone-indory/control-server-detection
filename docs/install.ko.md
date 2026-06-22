# OCR/LLM 설치 가이드

이 문서는 `control-server-detection`만 보고 새 환경에서 OCR/LLM 런타임을
재현하기 위한 설치 절차다. 기본 목표는 기존 `gz-nav-sim`에서 쓰던
PaddleOCR, Qwen GGUF LLM judge, Qwen-VL/transformers 런타임을 같은 계약으로
서비스에서 실행하는 것이다.

## 설치 방식

가장 안전한 기본값은 기존 `gz-nav-sim` adapter venv를 그대로 쓰는 것이다.

```bash
cd ~/control-server-detection
./run.sh
```

`run.sh`는 먼저 아래 Python을 찾는다.

```text
~/gz-nav-sim/indoors-web/ros_adapter/venv/bin/python
```

새 머신이나 새 체크아웃에서 같은 형태의 환경을 만들어야 하면 `setup.sh`를
쓴다.

```bash
cd ~/control-server-detection
./setup.sh
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python ./run.sh
```

`setup.sh`는 `requirements/runtime.txt`를 설치한다. 이 lock에는 FastAPI와
함께 OCR/LLM/VLM 런타임이 포함되어 있다.

- OCR: `paddleocr`, `paddlepaddle`, `paddlex`, `opencv-python`
- 송장 LLM judge: `llama-cpp-python`, `huggingface_hub`
- semantic VLM: `torch`, `transformers`, `accelerate`, `safetensors`
- 서비스: `fastapi`, `uvicorn`, `pydantic`

## OCR 설치와 검증

PaddleOCR은 필수 런타임이다. 기본값으로 Tesseract fallback을 쓰지 않는다.

```bash
cd ~/control-server-detection
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python ./preflight.sh
```

preflight는 실제로 아래 모델을 로딩하고 synthetic 이미지 OCR까지 수행한다.

- `PP-OCRv5_mobile_det`
- `korean_PP-OCRv5_mobile_rec`

PaddleX 모델 캐시가 깨졌으면 `~/.paddlex/official_models/*.bad_<timestamp>`로
백업한 뒤 다시 다운로드해서 검증한다.

실행 중 OCR fallback을 허용하지 않는 기본값:

```bash
export WAYBILL_OCR_REQUIRE_PADDLE=1
```

## Qwen GGUF LLM 설치

송장 OCR+LLM 기본 judge는 `llama_cpp`다.

서비스는 Qwen GGUF 모델을 아래 순서로 찾는다.

1. `WAYBILL_OCR_DEFAULT_MODEL`
2. `$WAYBILL_OCR_ROOT/models/Qwen2.5-7B-Instruct-Q4_K_M.gguf`
3. `~/waybill_ocr_llm/models/Qwen2.5-7B-Instruct-Q4_K_M.gguf`
4. 없으면 `huggingface_hub`로 한 번 다운로드 시도

명시적으로 모델 파일을 준비하려면:

```bash
mkdir -p ~/waybill_ocr_llm/models
~/control-server-detection/.venv/bin/python - <<'PY'
from huggingface_hub import hf_hub_download
from pathlib import Path

hf_hub_download(
    repo_id="bartowski/Qwen2.5-7B-Instruct-GGUF",
    filename="Qwen2.5-7B-Instruct-Q4_K_M.gguf",
    local_dir=str(Path.home() / "waybill_ocr_llm" / "models"),
    local_dir_use_symlinks=False,
)
PY
```

다른 GGUF 파일을 쓸 때는 경로를 직접 지정한다.

```bash
WAYBILL_OCR_JUDGE_MODE=llama_cpp \
WAYBILL_OCR_DEFAULT_MODEL=/path/to/Qwen2.5-7B-Instruct-Q4_K_M.gguf \
WAYBILL_LLM_CTX=4096 \
WAYBILL_LLM_MAX_NEW_TOKENS=128 \
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python \
./run.sh
```

Qwen 응답 JSON이 중간에서 잘리면 `WAYBILL_LLM_MAX_NEW_TOKENS`를 먼저 올린다.
기본값은 `128`이다. OCR line이 많아 prompt 자체가 1024 context를 넘는 경우가
있어서 서비스 기본 context는 기존 benchmark 흐름과 맞춰 `WAYBILL_LLM_CTX=4096`이다.

Ollama나 OpenAI-compatible 서버를 쓰면 로컬 GGUF 파일은 필요 없다.

```bash
WAYBILL_OCR_JUDGE_MODE=ollama \
WAYBILL_OCR_MODEL=qwen2.5 \
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python \
./run.sh

WAYBILL_OCR_JUDGE_MODE=openai \
WAYBILL_OCR_ENDPOINT=http://127.0.0.1:8000/v1/chat/completions \
WAYBILL_OCR_MODEL=Qwen/Qwen2.5-7B-Instruct \
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python \
./run.sh
```

## Qwen-VL 설치

`/v1/vlm/inspect`는 `torch`와 `transformers`를 사용한다. 기본 모델은
`Qwen/Qwen2.5-VL-3B-Instruct`다.

첫 VLM 요청 때 Hugging Face cache에 모델이 없으면 다운로드가 발생한다.
사전에 캐시를 채우고 싶으면 같은 Python으로 한 번 로딩한다.

```bash
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python
$CONTROL_SERVER_DETECTION_PYTHON - <<'PY'
from transformers import AutoProcessor

AutoProcessor.from_pretrained("Qwen/Qwen2.5-VL-3B-Instruct", trust_remote_code=True)
print("Qwen-VL processor cache ok")
PY
```

실제 모델 가중치 로딩은 메모리와 시간이 많이 들 수 있으므로 운영 장비에서
서비스 health와 첫 VLM 요청으로 확인한다.

## 설치 확인

패키지 import와 PaddleOCR preflight:

```bash
cd ~/control-server-detection
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python ./preflight.sh
$PWD/.venv/bin/python -c "import paddleocr, paddle, llama_cpp, torch, transformers; print('ocr/llm imports ok')"
```

서비스 health:

```bash
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python ./run.sh
curl -sf http://127.0.0.1:8767/health | python -m json.tool
```

`gz_compat` provider 기준으로 기대하는 capability:

- `ocr.read`: PaddleOCR + PaddlePaddle import 가능
- `semantic_ocr.room_signs`: PaddleOCR + PaddlePaddle import 가능
- `waybill.scan`: PaddleOCR + PaddlePaddle + `llama_cpp` import 가능
- `vlm.inspect`: `torch` + `transformers` import 가능

## gz-nav-sim 연결

서비스를 켠 뒤 main adapter에서 필요한 기능만 service mode로 켠다.

```bash
INDORY_OCR_SERVICE_URL=http://127.0.0.1:8767 \
WAYBILL_OCR_USE_SERVICE=1 \
INIT_OCR_USE_SERVICE=1 \
~/gz-nav-sim/indoors-web/ros_adapter/run.sh
```

ROS semantic node는 launch argument를 사용한다.

```bash
ros2 launch gz_nav_sim sim_nav.launch.py \
  ocr_use_service:=true \
  vlm_use_service:=true \
  ocr_llm_service_url:=http://127.0.0.1:8767
```
