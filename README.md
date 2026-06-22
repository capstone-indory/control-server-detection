# control-server-detection

Indory detection control server for camera-image understanding workflows.

This repository exposes OCR, waybill destination recognition, room-sign OCR,
and semantic text-object inspection as a FastAPI service. Other Indory control
servers send camera snapshots over HTTP; this service owns PaddleOCR, Qwen/LLM,
and VLM execution and returns stable JSON decisions.

The internal Python package is still named `indory_ocr` for compatibility with
the original gz-nav-sim adapter path. Public repository and distribution names
use `control-server-detection`.

## Interfaces

- `POST /v1/waybill/scan`: waybill OCR + destination decision
- `POST /v1/ocr/read`: OCR-only text and box extraction
- `POST /v1/semantic-ocr/room-signs`: room/sign OCR with floor prior
- `POST /v1/vlm/inspect`: semantic VLM scene/text-object inspection
- `GET /health`: service and provider health
- `GET /v1/contracts`: machine-readable route contracts

Default local URL:

```text
http://127.0.0.1:8767
```

## Quick Start

```bash
cd ~/control-server-detection
./setup.sh
CONTROL_SERVER_DETECTION_PYTHON=$PWD/.venv/bin/python ./run.sh
```

For compatibility with the original integrated stack, `run.sh` first looks for
the gz-nav-sim adapter virtual environment at:

```text
~/gz-nav-sim/indoors-web/ros_adapter/venv/bin/python
```

If that environment does not exist, set `CONTROL_SERVER_DETECTION_PYTHON` to a
compatible Python environment or run `./setup.sh` to create `.venv`.

PaddleOCR is a required runtime dependency. The service sets
`WAYBILL_OCR_REQUIRE_PADDLE=1` by default so broken OCR setup fails fast instead
of silently falling back to a different OCR engine.

## Runtime

```bash
CONTROL_SERVER_DETECTION_PROVIDER=gz_compat ./run.sh
curl -sf http://127.0.0.1:8767/health | python -m json.tool
```

Provider choices:

- `gz_compat`: production-compatible provider for PaddleOCR, waybill OCR+LLM,
  room-sign OCR, and semantic VLM.
- `not_configured`: contract-test provider; does not run model inference.
- `module:Class`: custom provider implementing `indory_ocr.providers.base.OcrLlmProvider`.

Important environment variables:

| Variable | Default | Description |
|---|---:|---|
| `CONTROL_SERVER_DETECTION_HOST` | `127.0.0.1` | FastAPI bind host |
| `CONTROL_SERVER_DETECTION_PORT` | `8767` | FastAPI bind port |
| `CONTROL_SERVER_DETECTION_PYTHON` | auto-detected | Python used by `run.sh` |
| `CONTROL_SERVER_DETECTION_PROVIDER` | `gz_compat` | provider implementation |
| `CONTROL_SERVER_DETECTION_ARTIFACT_ROOT` | `/tmp/control_server_detection` | local debug artifact directory |
| `CONTROL_SERVER_DETECTION_KEEP_ARTIFACTS` | `0` | retain request images/intermediate artifacts |
| `CONTROL_SERVER_DETECTION_INCLUDE_DEBUG` | `0` | include debug payloads by default |
| `CONTROL_SERVER_DETECTION_MAX_IMAGE_MB` | `16` | max accepted image payload size |
| `WAYBILL_OCR_JUDGE_MODE` | `llama_cpp` | `llama_cpp`, `openai`, or `ollama` |
| `WAYBILL_OCR_DEFAULT_MODEL` | auto-detected | local Qwen GGUF path for llama.cpp |
| `WAYBILL_OCR_MODEL` | empty | model name for Ollama/OpenAI-compatible mode |
| `WAYBILL_OCR_ENDPOINT` | empty | Ollama or OpenAI-compatible endpoint |
| `WAYBILL_OCR_REQUIRE_PADDLE` | `1` | require PaddleOCR instead of fallback OCR |

Legacy `INDORY_OCR_*` variables are still accepted so existing adapters can
move gradually.

## Benchmark

Benchmarks call the running HTTP service rather than internal functions.

Use a Hugging Face dataset snapshot:

```bash
python3 benchmark/run.py \
  --dataset hf:Fnhid/indory-waybill-ocr-640x480 \
  --ocr-full-image-variants \
  --ocr-crop-variants \
  --include-debug
```

Use a local manifest or image directory:

```bash
python3 benchmark/run.py --dataset /path/to/manifest.json
python3 benchmark/run.py /path/to/images --recursive --include-debug
```

Benchmark outputs are written under `benchmark/runs/`, which is intentionally
ignored by git. Raw benchmark images, generated datasets, review exports, and
ground-truth annotation files are also ignored and should be published through a
separate dataset channel when appropriate.

More details: [benchmark/README.ko.md](benchmark/README.ko.md)

## Dataset Publication

The repository should be published to GitHub without benchmark images or model
artifacts. Package the OCR benchmark images separately for Hugging Face:

```bash
python3 benchmark/dataset.py export --overwrite
python3 benchmark/dataset.py validate
python3 benchmark/dataset.py upload \
  --repo-id Fnhid/indory-waybill-ocr-640x480 \
  --create-repo \
  --private
```

After upload, the service benchmark can evaluate the dataset directly:

```bash
python3 benchmark/run.py --dataset hf:Fnhid/indory-waybill-ocr-640x480
```

More details: [docs/publish.ko.md](docs/publish.ko.md)

## Integration

The service is designed to be called by the broader Indory stack:

- `control-server-slam-navigation` or gz-nav-sim-compatible launchers can call
  `/v1/semantic-ocr/room-signs` for room-sign detection.
- `control-server-backend` can call `/v1/waybill/scan` when a task needs parcel
  destination recognition.
- Other adapters should send a base64 image and consume the JSON contracts
  documented in [docs/api.ko.md](docs/api.ko.md).

Existing gz-nav-sim adapter flags remain supported:

```bash
INDORY_OCR_SERVICE_URL=http://127.0.0.1:8767 \
WAYBILL_OCR_USE_SERVICE=1 \
INIT_OCR_USE_SERVICE=1 \
~/gz-nav-sim/indoors-web/ros_adapter/run.sh
```

## Security, Privacy, and Artifacts

This repository should contain code, contracts, and small examples only. Do not
commit:

- `.env` files, access tokens, private keys, or service credentials
- local IPs, Tailscale addresses, SSH usernames, or hardware serial IDs
- map databases, rosbag files, camera captures, OCR benchmark images, or logs
- model weights, checkpoints, GGUF files, PaddleOCR exports, or VLM caches

Use `.env.example` for documented configuration. Report security issues through
the project security policy in [SECURITY.md](SECURITY.md). Before public
release, run a secret scan such as `trufflehog filesystem .` or an equivalent
tool.

## License

This repository is released under the Apache License 2.0. See [LICENSE](LICENSE).
Third-party dependency notes are summarized in
[THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md).
