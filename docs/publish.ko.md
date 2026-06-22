# 공개 및 업로드 가이드

이 문서는 `control-server-detection` 코드 레포를 GitHub에 올리고, 송장 OCR
benchmark dataset을 Hugging Face dataset repo에 따로 올리기 위한 절차다.

## 현재 이름 구조

- 로컬 레포 디렉토리: `~/control-server-detection`
- 공개 GitHub 레포명: `control-server-detection`
- Python 배포명: `control-server-detection`
- Python 내부 패키지명: `indory_ocr`
- 레거시 CLI alias: `indory-ocr`

`indory_ocr` 내부 패키지명은 기존 adapter/import 호환을 위해 유지한다. 디렉토리와
공개 레포 이름만 `control-server-detection`으로 사용한다.

## GitHub 코드 레포 준비

코드 레포에는 source, docs, tests, scripts, API contract만 포함한다. 아래 항목은
git에 넣지 않는다.

- `.env`, token, key, credential
- camera capture, rosbag, map database, runtime log
- benchmark image dataset, review export, generated dataset folder
- model weights, checkpoints, GGUF, PaddleOCR export, VLM cache

출시 전 확인:

```bash
cd ~/control-server-detection
git status --short --ignored
python3 -m pytest -q
CONTROL_SERVER_DETECTION_TEST_PYTHON=~/gz-nav-sim/indoors-web/ros_adapter/venv/bin/python \
  CONTROL_SERVER_DETECTION_BENCH_LIMIT=1 \
  benchmark/test.sh
```

민감정보 scan:

```bash
rg -n "(BEGIN .*KEY|token|password|secret|api[_-]?key|tailscale|100\\.|192\\.168\\.|10\\.)" .
rg -n "/home/" . --glob '!PUBLIC_RELEASE_CHECKLIST.md' --glob '!docs/publish.ko.md'
```

GitHub에 처음 올릴 때:

```bash
git add .
git status --short
git commit -m "Prepare control-server-detection public release"
gh repo create Fnhid/control-server-detection --private --source=. --remote=origin --push
```

처음에는 `--private`를 권장한다. 공개 전에는 GitHub secret scanning 또는
`trufflehog filesystem .` 같은 외부 scan을 한 번 더 돌린다.

## Hugging Face Dataset 준비

현재 로컬 benchmark source는 기본적으로 아래 manifest를 사용한다.

```text
~/data/benchmarks/waybill_ocr/run_full_640x480/current_manifest.json
```

manifest를 갱신하고 HF-friendly 폴더로 포장한다.

```bash
cd ~/control-server-detection
python3 benchmark/gt.py build --allow-unannotated
python3 benchmark/dataset.py export --overwrite
python3 benchmark/dataset.py validate
```

생성 위치:

```text
benchmark/datasets/indory_waybill_ocr_640x480/
```

이 폴더는 git에는 들어가지 않고, Hugging Face dataset repo에만 업로드한다. 현재
export 구조는 다음 파일을 만든다.

- `README.md`
- `manifest.json`
- `metadata.jsonl`
- `dataset_summary.json`
- `checksums.sha256`
- `images/test/*.jpg`

업로드:

```bash
python3 benchmark/dataset.py upload \
  --repo-id Fnhid/indory-waybill-ocr-640x480 \
  --create-repo \
  --private
```

공개 전에는 dataset license, 이미지 공개 가능 여부, 개인정보/운송장 정보 노출 여부를
다시 확인한다.

업로드 후 재검증:

```bash
python3 benchmark/dataset.py download \
  --repo-id Fnhid/indory-waybill-ocr-640x480 \
  --out-dir benchmark/datasets/hf_download \
  --force

python3 benchmark/run.py --dataset hf:Fnhid/indory-waybill-ocr-640x480
```
