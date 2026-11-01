# FPF 工位与环境快速初始化参考模版 (Station & Infrastructure Templates)

本文件提供快速初始化 FPF 开发与审查环境的可复用模版代码与配置文件。

---

## 模版 1：自动化生成 Manifest 与校验和脚本 (`generate_manifest.py`)

用于对工位制品包遍历计算 SHA-256 并生成权威清单：

```python
import os, sys, json, hashlib
from pathlib import Path

def generate_station_manifest(target_dir: Path, output_file: Path, schema: str = "fpf-stations-manifest-v1"):
    items = []
    for root, _, files in os.walk(target_dir):
        for file in sorted(files):
            if file in ("manifest.json", "manifest.sha256"):
                continue
            fp = Path(root) / file
            rel_path = fp.relative_to(target_dir).as_posix()
            data = fp.read_bytes()
            items.append({
                "path": rel_path,
                "size_bytes": len(data),
                "sha256": hashlib.sha256(data).hexdigest()
            })

    items.sort(key=lambda x: x["path"])
    manifest = {
        "schema_version": schema,
        "item_count": len(items),
        "items": items
    }
    manifest_bytes = json.dumps(manifest, indent=2, ensure_ascii=False).encode("utf-8")
    output_file.write_bytes(manifest_bytes)
    
    sha256_hash = hashlib.sha256(manifest_bytes).hexdigest()
    output_file.with_suffix(".sha256").write_text(f"{sha256_hash}  manifest.json\n", encoding="utf-8")
    print(f"Generated manifest: {len(items)} items, sha256={sha256_hash}")

if __name__ == "__main__":
    workdir = Path(sys.argv[1]) if len(sys.argv) > 1 else Path.cwd()
    generate_station_manifest(workdir, workdir / "manifest.json")
```

---

## 模版 2：硬化版 GitHub Actions 3-Job 独立 QA 工作流

部署于 `.github/workflows/fpf-independent-qa.yml`：

```yaml
name: FPF Independent QA (Hardened 3-Job Sandbox)

on:
  workflow_dispatch:
    inputs:
      candidate_repo:
        description: "Candidate Repository (e.g. huangzuomin/five-powers-flow)"
        required: true
        type: string
      tested_sha:
        description: "Exact Candidate Commit SHA (40-char hex)"
        required: true
        type: string
      task_id:
        description: "FPF Task ID"
        required: true
        type: string

permissions: {}

jobs:
  admission:
    name: "Job 1 · Trusted Controller Admission"
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    permissions:
      contents: read
    outputs:
      tested_sha: ${{ steps.sanitize.outputs.tested_sha }}
      task_id: ${{ steps.sanitize.outputs.task_id }}
    steps:
      - name: Validate Inputs and Controller Authority
        id: sanitize
        env:
          TESTED_SHA: ${{ inputs.tested_sha }}
          TASK_ID: ${{ inputs.task_id }}
        run: |
          if ! [[ "$TESTED_SHA" =~ ^[0-9a-f]{40}$ ]]; then exit 1; fi
          echo "tested_sha=$TESTED_SHA" >> "$GITHUB_OUTPUT"
          echo "task_id=$TASK_ID" >> "$GITHUB_OUTPUT"

      - name: Fetch Candidate Read-Only Snapshot
        env:
          CANDIDATE_REPO: ${{ inputs.candidate_repo }}
          TESTED_SHA: ${{ steps.sanitize.outputs.tested_sha }}
        run: |
          mkdir -p staging/candidate
          cd staging/candidate
          git init && git remote add origin "https://github.com/${CANDIDATE_REPO}.git"
          git fetch --depth 1 origin "$TESTED_SHA"
          git checkout "$TESTED_SHA"
          rm -rf .git
          cd .. && tar -czf candidate-source.tar.gz candidate/
          mkdir -p ../admission-package && mv candidate-source.tar.gz ../admission-package/

      - name: Upload Trusted Admission Package
        uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2b6cb773ac77
        with:
          name: admission-${{ steps.sanitize.outputs.tested_sha }}
          path: admission-package/*
          retention-days: 1

  sandboxed_execution:
    name: "Job 2 · Hardened Sandbox Execution"
    needs: admission
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    permissions: {} # Zero GitHub Tokens
    steps:
      - name: Download Admission Package
        uses: actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16
        with:
          name: admission-${{ needs.admission.outputs.tested_sha }}
          path: admission-package

      - name: Setup Isolated Workspace
        run: |
          mkdir -p workspace/app-ro workspace/scratch host-monitor
          tar -xzf admission-package/candidate-source.tar.gz -C workspace/app-ro
          chmod -R 777 workspace/scratch

      - name: Execute in Hardened Sandbox
        env:
          TESTED_SHA: ${{ needs.admission.outputs.tested_sha }}
          IMAGE_DIGEST: "python:3.11-slim@sha256:7bc030026e6f1f45df7456d9be70ec9ff416df98b093399b24da1c54b2383827"
        run: |
          set +e
          docker run --rm \
            --name "fpf-sandbox-${TESTED_SHA:0:8}" \
            --network none \
            --user 1001:1001 \
            --read-only \
            --cap-drop=ALL \
            --security-opt=no-new-privileges \
            --pids-limit=100 \
            --cpus=2 \
            --memory=2g \
            --tmpfs /tmp:rw,noexec,nosuid,size=64m \
            -v "$(pwd)/workspace/app-ro/candidate:/app:ro" \
            -v "$(pwd)/workspace/scratch:/scratch:rw" \
            -w /app \
            "$IMAGE_DIGEST" \
            bash -c "python -m unittest discover -s tests -p 'test_*.py' 2>&1" > host-monitor/stdout.log
          EXIT_CODE=$?
          set -e
          echo "$EXIT_CODE" > host-monitor/exit_code.txt
          mkdir -p raw-evidence && cp -r host-monitor raw-evidence/

      - name: Upload Raw Evidence
        uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2b6cb773ac77
        with:
          name: raw-evidence-${{ needs.admission.outputs.tested_sha }}
          path: raw-evidence/*
          retention-days: 1

  independent_evaluator:
    name: "Job 3 · Specification-Driven Evaluator"
    needs: [admission, sandboxed_execution]
    runs-on: ubuntu-24.04
    permissions:
      id-token: write
      attestations: write
      contents: read
    steps:
      - name: Download Raw Evidence
        uses: actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16
        with:
          name: raw-evidence-${{ needs.admission.outputs.tested_sha }}
          path: raw-evidence

      - name: Audit Test Exit Code, Specific Counts and Non-Skip
        run: |
          python3 -c "
          import sys, re
          with open('raw-evidence/host-monitor/exit_code.txt') as f:
              code = int(f.read().strip())
          if code != 0: sys.exit(1)

          with open('raw-evidence/host-monitor/stdout.log') as f:
              out = f.read()
          match = re.search(r'Ran (\d+) test', out)
          if not match or int(match.group(1)) < 1: sys.exit(1)
          if 'FAILED' in out or 'ERROR' in out or 'skipped=' in out: sys.exit(1)
          print('Passed Specification Audit')
          "

      - name: Attest Build Provenance
        uses: actions/attest-build-provenance@c074443f1a5fb4a482973331d3a1f4e046f4471a
        with:
          subject-path: "raw-evidence/host-monitor/exit_code.txt"
```
