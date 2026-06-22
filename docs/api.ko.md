# control-server-detection HTTP 인터페이스

이 레포는 `gz-nav-sim` 안의 OCR/LLM 실행 책임을 밖으로 빼기 위한 독립
FastAPI 서비스다. `gz-nav-sim`은 이미지만 보내고, 이 서비스가 OCR/LLM 결과를
표준 JSON으로 돌려준다.

기본 주소:

```text
http://127.0.0.1:8767
```

## 실행

```bash
cd ~/control-server-detection
./run.sh
```

`run.sh`는 새 AI 환경을 임의로 만들지 않는다. 기본으로 기존
`gz-nav-sim` adapter venv를 그대로 사용한다.

```text
~/gz-nav-sim/indoors-web/ros_adapter/venv/bin/python
```

새 머신이나 새 체크아웃에서 같은 형태의 venv를 만들 때는 lock 파일 기반 setup
스크립트를 쓴다.

```bash
./setup.sh
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python ./run.sh
```

PaddleOCR, Qwen GGUF LLM, Qwen-VL/transformers 설치와 검증 절차는
[OCR/LLM 설치 가이드](install.ko.md)에 정리되어 있다.

setup 마지막에는 PaddleOCR v3 preflight가 돈다. 이 preflight는 기존
`waybill_ocr_llm.ocr.build_paddleocr()`로 `PP-OCRv5_mobile_det`와
`korean_PP-OCRv5_mobile_rec`를 실제 로딩하고 synthetic 이미지 OCR까지 수행한다.
깨진 PaddleX partial cache가 있으면 `~/.paddlex/official_models/*.bad_<timestamp>`로
백업한 뒤 다시 다운로드해서 검증한다.

기본 provider는 `gz_compat`다. 이 provider가 현재 `gz-nav-sim`에서 쓰던
PaddleOCR 기반 OCR, waybill OCR+LLM, semantic VLM 경로를 보존한다.

`not_configured`는 계약 테스트용 mock provider다. 런타임에서는 쓰면 안 된다.

```bash
CONTROL_SERVER_DETECTION_PROVIDER=gz_compat \
./run.sh
```

provider 클래스는 `indory_ocr.providers.base.OcrLlmProvider`를
상속하고 `scan_waybill()`, `read_ocr()`, `read_room_signs()`,
`inspect_vlm()`, `health()`를 구현하면 된다.

## gz-nav-sim 연결

`gz-nav-sim` 원본 로컬 OCR/LLM 경로는 기본값으로 남아 있다. 아래 flag를 켜는
경우에만 이 독립 서비스로 camera snapshot을 보낸다.

`indoors-web/ros_adapter`:

```bash
INDORY_OCR_SERVICE_URL=http://127.0.0.1:8767 \
WAYBILL_OCR_USE_SERVICE=1 \
INIT_OCR_USE_SERVICE=1 \
~/gz-nav-sim/indoors-web/ros_adapter/run.sh
```

ROS semantic 노드:

```bash
ros2 launch gz_nav_sim sim_nav.launch.py \
  ocr_use_service:=true \
  vlm_use_service:=true \
  ocr_llm_service_url:=http://127.0.0.1:8767
```

## 환경변수

| 변수 | 기본값 | 설명 |
|---|---:|---|
| `CONTROL_SERVER_DETECTION_HOST` | `127.0.0.1` | bind host |
| `CONTROL_SERVER_DETECTION_PORT` | `8767` | bind port |
| `CONTROL_SERVER_DETECTION_PYTHON` | 기존 adapter venv python | 서비스 실행 Python |
| `CONTROL_SERVER_DETECTION_PROVIDER` | `gz_compat` | OCR/LLM/VLM provider |
| `GZ_NAV_SIM_ROOT` | `~/gz-nav-sim` | 기존 gz-nav-sim checkout |
| `GZ_NAV_HUMBLE_SETUP` | `~/micromamba/envs/gz-nav-humble/setup.bash` | gz-nav-sim micromamba setup |
| `CONTROL_SERVER_DETECTION_KEEP_ARTIFACTS` | `0` | 요청 이미지/중간 산출물 저장 여부 |
| `CONTROL_SERVER_DETECTION_INCLUDE_DEBUG` | `0` | 기본 debug payload 포함 여부 |
| `CONTROL_SERVER_DETECTION_MAX_IMAGE_MB` | `16` | 단일 이미지 최대 크기 |
| `WAYBILL_OCR_JUDGE_MODE` | `llama_cpp` | 송장 LLM judge 모드: `llama_cpp`, `openai`, `ollama` |
| `WAYBILL_OCR_DEFAULT_MODEL` | 로컬 Qwen GGUF 자동 탐색 | `llama_cpp` judge 모델 경로 |
| `WAYBILL_OCR_MODEL` | 빈 값 | Qwen/Ollama/OpenAI-compatible 모델 이름 |
| `WAYBILL_OCR_ENDPOINT` | 빈 값 | OpenAI-compatible 또는 Ollama endpoint |
| `WAYBILL_OCR_ROTATIONS` | `0` | 송장 OCR 회전 후보 |
| `WAYBILL_LLM_MAX_NEW_TOKENS` | `128` | Qwen/LLM judge 응답 최대 토큰 수 |
| `WAYBILL_LLM_CTX` | `4096` | Qwen/llama.cpp context window. OCR line이 많은 기존 benchmark 송장은 1024로 부족할 수 있음 |
| `WAYBILL_LLM_MAX_ATTEMPTS` | `10` | LLM JSON 검증 실패 시 재시도 횟수 |
| `WAYBILL_OCR_REQUIRE_PADDLE` | `1` | PaddleOCR 실패 시 다른 OCR fallback 금지 |
| `WAYBILL_ROOM_POLICY` | `any` | 송장 방번호 후보 허용 정책. 특수 테스트에서만 `5xx` 등으로 제한 |
| `WAYBILL_MAX_REASONABLE_FLOOR` | `40` | 긴 숫자가 방번호인지 도로명+호수인지 가르는 최대 일반 층수 휴리스틱 |
| `WAYBILL_ROOM_PREFIX_FLOOR_DIGIT` | 빈 값 | `T28-1` 같은 prefix OCR 오독을 `528-1`처럼 복원할 때만 명시적으로 설정 |
| `INDORY_OCR_REPAIR_CACHE` | `1` | legacy preflight cache repair flag |

Qwen을 LLM judge로 쓰는 예:

```bash
WAYBILL_OCR_JUDGE_MODE=llama_cpp \
WAYBILL_LLM_CTX=4096 \
./run.sh

WAYBILL_OCR_JUDGE_MODE=ollama \
WAYBILL_OCR_MODEL=qwen2.5 \
./run.sh

WAYBILL_OCR_JUDGE_MODE=openai \
WAYBILL_OCR_ENDPOINT=http://127.0.0.1:8000/v1/chat/completions \
WAYBILL_OCR_MODEL=Qwen/Qwen2.5-7B-Instruct \
./run.sh
```

`llama_cpp` 모드에서 `WAYBILL_OCR_DEFAULT_MODEL`을 지정하지 않으면
`~/waybill_ocr_llm/models/Qwen2.5-7B-Instruct-Q4_K_M.gguf`를 먼저 찾는다.

## Health

```http
GET /health
```

응답:

```json
{
  "ok": true,
  "service": "control_server_detection",
  "version": "0.1.0",
  "provider": "gz_compat",
  "provider_health": {
    "ok": true,
    "provider_ready": true,
    "availability": {
      "ocr.read": true,
      "semantic_ocr.room_signs": true,
      "waybill.scan": true,
      "vlm.inspect": true
    },
    "paddleocr_required": true,
    "paddleocr_version": "3.6.0",
    "python": "~/gz-nav-sim/indoors-web/ros_adapter/venv/bin/python"
  },
  "routes": {
    "waybill_scan": "/v1/waybill/scan",
    "ocr_read": "/v1/ocr/read",
    "semantic_ocr_room_signs": "/v1/semantic-ocr/room-signs",
    "vlm_inspect": "/v1/vlm/inspect"
  }
}
```

## 송장 OCR+LLM

```http
POST /v1/waybill/scan
Content-Type: application/json
```

요청:

```json
{
  "request_id": "task-123-waybill-001",
  "task_id": 123,
  "camera": "wrist_left",
  "source": "gz-nav-sim.ros_adapter",
  "image_b64": "...base64 jpeg...",
  "image_format": "jpg",
  "include_debug": true,
  "options": {
    "ocr_full_image_variants": true,
    "ocr_crop_variants": true,
    "confidence_threshold": 0.75
  }
}
```

허용 입력:

| 필드 | 필수 | 설명 |
|---|---:|---|
| `image_b64` 또는 `image_base64` | 예 | JPEG/PNG/WebP base64. data URL 허용 |
| `images` 또는 `frames` | 아니오 | 여러 프레임 consensus용. 문자열 또는 frame object 배열 |
| `request_id` | 아니오 | 추적 id. 없으면 서비스가 생성 |
| `task_id` | 아니오 | 메인 task id |
| `camera` | 아니오 | `head`, `floor`, `wrist_left` 등 |
| `include_debug` | 아니오 | OCR boxes, 후보, raw LLM 응답 포함 |
| `options` | 아니오 | provider별 옵션. 메인 adapter는 이해하지 않고 그대로 전달 가능 |

`waybill.scan`에서 자주 쓰는 provider 옵션은 `ocr_full_image_variants`,
`ocr_crop_variants`, `ocr_use_gpu`, `ocr_rec_batch_num`, `judge_mode`,
`model_path`, `llm_ctx`, `llm_gpu_layers`, `max_new_tokens`다.

응답 핵심:

```json
{
  "type": "result",
  "ok": true,
  "request_id": "task-123-waybill-001",
  "task_id": 123,
  "camera": "wrist_left",
  "destination": "5F 528-1호",
  "decision": {
    "destination_dong": null,
    "destination_floor": "5F",
    "destination_room": "528-1호",
    "confidence": 0.93,
    "evidence_indices": [0],
    "needs_manual_review": false,
    "auto_accept": true,
    "risk_reasons": []
  },
  "needs_manual_review": false,
  "auto_accept": true,
  "risk_reasons": [],
  "timing": {
    "ocr_seconds": 1.2,
    "llm_seconds": 3.4,
    "total_seconds": 4.6
  }
}
```

`gz-nav-sim` 자동 진행 규칙:

- `auto_accept=true`
- `needs_manual_review=false`
- 목적지 후보가 `destination` 또는 `decision.destination_room`에 존재

위 조건이 아니면 task는 자동 pass하지 않고 재촬영/수동 확인으로 간다.

송장 후보 추출은 특정 테스트 주소 문자열 목록에 의존하지 않는다. 예를 들어
`로봇대로` 같은 단어만으로 주소로 보지 않고, `~시/~군/~구`, `~대로/~로/~길 +
번지`, 건물 유형, `배송주소` 같은 라벨을 조합해 주소 맥락을 판단한다. 특정 층만
허용해야 하는 실험은 기본값에 넣지 않고 `WAYBILL_ROOM_POLICY`로 명시한다.

## 순수 OCR

초기 맵 스캔/표지판 OCR처럼 LLM 판단이 필요 없는 경우:

```http
POST /v1/ocr/read
Content-Type: application/json
```

요청:

```json
{
  "request_id": "init-ocr-001",
  "camera": "head",
  "source": "gz-nav-sim.init_ocr",
  "image_b64": "...base64 jpeg...",
  "image_format": "jpg",
  "ocr_rotations": [0]
}
```

응답:

```json
{
  "type": "ocr_result",
  "ok": true,
  "request_id": "init-ocr-001",
  "model": "provider-specific-model-name",
  "rotations": [0],
  "item_count": 1,
  "items": [
    {
      "text": "528",
      "confidence": 0.94,
      "box": [[120.0, 80.0], [170.0, 80.0], [170.0, 104.0], [120.0, 104.0]],
      "cx": 145.0,
      "cy": 92.0
    }
  ],
  "timing": {
    "ocr_seconds": 0.31,
    "llm_seconds": 0.0,
    "total_seconds": 0.31
  }
}
```

`gz-nav-sim`은 raw OCR이 필요할 때만 이 endpoint를 사용한다. 방번호 표지판
인식처럼 기존 semantic OCR 동작이 필요한 경우에는 아래 endpoint를 사용한다.

## Semantic OCR 방번호/표지판

이 endpoint는 일반 텍스트 검출기가 아니라 **방번호/표지판 전용 OCR 후처리**다.
`floor_hint`가 주어지면 반드시 현재 층 constraint로 사용한다.
`floor_prior_mode=reject`는 층 힌트와 맞지 않는 room-id 후보를 버리고,
`floor_prior_mode=complete`는 같은 필터를 적용하되 일부 누락된 층 prefix를 복원한다.

```http
POST /v1/semantic-ocr/room-signs
Content-Type: application/json
```

요청:

```json
{
  "request_id": "semantic-ocr-001",
  "camera": "head",
  "source": "gz-nav-sim.semantic_ocr_node",
  "image_b64": "...base64 jpeg...",
  "floor_hint": "5F",
  "floor_prior_mode": "reject",
  "min_confidence": 0.6,
  "options": {
    "ocr_backend": "paddle",
    "ocr_use_gpu": false,
    "ocr_max_side": 1280,
    "ocr_scales": [1.0, 2.0]
  }
}
```

`ocr_backend`은 기존 호출과의 호환 필드지만 현재 지원 backend는 PaddleOCR뿐이다.
다른 값을 보내면 fallback하지 않고 오류를 반환한다.

응답:

```json
{
  "type": "semantic_ocr_result",
  "ok": true,
  "task_mode": "ocr_room_ids",
  "has_text_object": true,
  "objects": [
    {
      "type": "room_id_sign",
      "room_id": "528",
      "text": "528",
      "raw_text": "528",
      "confidence": 0.94,
      "bbox_xyxy": [120, 80, 170, 104],
      "source": "paddleocr@1x"
    }
  ],
  "raw_ocr_output": [],
  "metadata": {
    "ocr_backend": "paddle",
    "floor_hint": "5F",
    "floor_prior_mode": "reject"
  }
}
```

`semantic_ocr_node.py`는 RGB/Depth/TF/tracking/publish를 담당하고, OCR 추론과
방번호 정규화는 이 서비스가 담당한다.

## Semantic VLM

```http
POST /v1/vlm/inspect
Content-Type: application/json
```

요청:

```json
{
  "request_id": "semantic-vlm-001",
  "camera": "head",
  "source": "gz-nav-sim.semantic_vlm_node",
  "task_mode": "scene_description",
  "image_b64": "...base64 jpeg...",
  "model_name": "Qwen/Qwen2.5-VL-3B-Instruct",
  "device": "auto",
  "torch_dtype": "auto",
  "max_new_tokens": 256
}
```

`task_mode`는 `scene_description` 또는 `text_object`를 사용한다.
`text_object`는 표지판, 문패, 배송 라벨처럼 **텍스트가 있는 물체**만 찾는 모드다.
기존 `object_detection` 값은 호환을 위해 `text_object` alias로 처리하지만,
일반 객체 검출 prompt는 아니다.

응답:

```json
{
  "type": "vlm_result",
  "ok": true,
  "task_mode": "scene_description",
  "observation": {
    "scene_description_ko": "현재 프레임에 복도와 문이 보입니다.",
    "objects": [],
    "control_summary_ko": "복도 장면이 확인됩니다.",
    "need_human_check": false
  },
  "raw_response": "{...}",
  "metadata": {
    "model": "Qwen/Qwen2.5-VL-3B-Instruct",
    "prompt_family": "scene_description"
  }
}
```

`semantic_vlm_node.py`는 카메라 sampling, Depth/TF, candidate tracking,
ROS/Foxglove publish를 유지하고, prompt/model inference는 이 서비스가 담당한다.
