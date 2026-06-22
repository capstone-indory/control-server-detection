# Public Release Checklist

Use this checklist before publishing `control-server-detection`.

## Repository Scope

- Keep source code, API contracts, tests, and small documentation examples.
- Keep OCR benchmark images, GT annotations, review exports, generated datasets,
  checkpoints, and model weights outside git.
- Publish benchmark images through a separate Hugging Face dataset repository
  when the dataset license and visibility are decided.
- Keep `benchmark/dataset.py` in git; it packages and uploads the ignored local
  dataset folder to Hugging Face.

## Required Files

- `LICENSE` with Apache License 2.0.
- `README.md` with purpose, install, run, ports, API surfaces, safety notes, and license.
- `.env.example` with fake/default values only.
- `THIRD_PARTY_LICENSES.md` with dependency-family license notes.
- `SECURITY.md` with vulnerability reporting guidance.
- `docs/publish.ko.md` with GitHub and Hugging Face publication commands.

## Sensitive Data Scan

Check for local-only values before pushing:

```bash
git status --short --ignored
rg -n "(BEGIN .*KEY|token|password|secret|api[_-]?key|tailscale|100\\.|192\\.168\\.|10\\.)" .
rg -n "/home/" . --glob '!PUBLIC_RELEASE_CHECKLIST.md'
```

Recommended external scan:

```bash
trufflehog filesystem .
```

## Ignored Local Artifacts

The following should remain ignored:

- `.env`, `.env.*`
- `models/`, `*.gguf`, `*.safetensors`, `*.pt`, `*.pth`, `*.onnx`
- `benchmark/datasets/`, `benchmark/runs/`, `benchmark/ground_truth/`
- `benchmark/export_paddleocr_rec.py`
- camera media, rosbag files, map databases, logs, and generated artifacts

## GitHub Publication

Recommended first publish flow:

```bash
git init
git add .
git status --short
git commit -m "Prepare control-server-detection public release"
gh repo create Fnhid/control-server-detection --private --source=. --remote=origin --push
```

Use `--public` only after the sensitive data scan and dataset visibility review
are complete.

## Hugging Face Dataset Publication

Package and validate the current local waybill OCR dataset:

```bash
python3 benchmark/gt.py build --allow-unannotated
python3 benchmark/dataset.py export --overwrite
python3 benchmark/dataset.py validate
```

Upload to Hugging Face:

```bash
python3 benchmark/dataset.py upload \
  --repo-id Fnhid/indory-waybill-ocr-640x480 \
  --create-repo \
  --private
```

Keep the first dataset upload private until dataset license and visibility are
confirmed.
