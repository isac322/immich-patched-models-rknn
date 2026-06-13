# immich-patched-models-rknn

Pre-compiled RKNN model artifacts for [Immich](https://immich.app/) on Rockchip
NPUs, built from [immich-app/ml-models][ml-models] with the unmerged [PR #32][pr32]
applied (the CumSum-via-MatMul rewrite needed for textual encoders such as
`nllb-clip-large-siglip__mrl` and the XLM-RoBERTa family — without it
`librknnrt.so` aborts with `Unsupport CPU op: CumSum`).

The resulting OCI image is consumed as an `initContainer` that seeds the Immich
machine-learning cache PVC. It pairs with the matching ML server image at
[isac322/immich-machine-learning-rknn][ml-server].

[ml-models]: https://github.com/immich-app/ml-models
[pr32]: https://github.com/immich-app/ml-models/pull/32
[ml-server]: https://github.com/isac322/immich-machine-learning-rknn

## What gets built

| Item | Value |
| --- | --- |
| Image | `ghcr.io/isac322/immich-patched-models:<immich-tag>-rknn` |
| Base | `busybox:1.37` (arm64 digest-pinned) |
| Target SoC | `rk3588` |
| Build runner | `ubuntu-24.04-arm` (GH-hosted) |
| Tooling | `rknn-toolkit2>=2.3.0`, Python 3.12 |

The image is a single layer containing only the compiled `.rknn` files, laid out
to mirror the Immich ML cache directly:

```
clip/<model_name>/textual/rknpu/rk3588/model.rknn
clip/<model_name>/visual/rknpu/rk3588/model.rknn
facial-recognition/<model_name>/detection/rknpu/rk3588/model.rknn
facial-recognition/<model_name>/recognition/rknpu/rk3588/model.rknn
```

ONNX / tokenizer / preprocessor artifacts are intentionally stripped: Immich
already downloads those from Hugging Face at runtime, and RKNN is the only piece
that needs offline compilation.

## Repo layout

```
config/models.yaml          # models to export, one per line
model-patches/*.patch       # applied in lexical order against ml-models @ ML_MODELS_PIN
.github/workflows/          # GH Actions pipeline
```

### Adding a model

Append to `config/models.yaml`:

```yaml
models:
  - nllb-clip-large-siglip__mrl openclip
  - buffalo_l insightface
```

Format per line: `- <model_name> <model_source>` where `model_source` is one of
`openclip`, `mclip`, `insightface` (the `ModelSource` enum upstream).

A push to `main` touching `config/models.yaml`, `model-patches/**`, or the
workflow file rebuilds the next image automatically.

### Refreshing the patch series

`ML_MODELS_PIN` in the workflow pins the ml-models commit the patches were
authored against. When you bump the pin:

1. Re-generate each `model-patches/*.patch` against the new pin.
2. Confirm `git apply --check` is clean locally.
3. Push.

The workflow runs `git apply --check` before applying and fails fast with a
clear error if the patch series drifted.

## How it builds

1. `resolve-version` reads `inputs.immich_version` (defaulting to the latest
   `immich-app/immich` release tag), computes `<version>-rknn`, and skips the
   build if that tag already exists in the GHCR package (unless
   `force_rebuild=true`).
2. `export-and-push` runs on `ubuntu-24.04-arm`:
   - frees ~10 GiB on `/` and adds 16 GiB of swap (the large NLLB textual
     export peaks in the 15-20 GiB range; the runner only has 16 GiB RAM);
   - clones `ml-models` at `ML_MODELS_PIN`, applies every patch under
     `model-patches/` in order;
   - installs `uv` pinned to Python 3.12 (ml-models' `.python-version` is 3.14,
     but `rknn-toolkit2` only publishes cp310/cp311/cp312 wheels);
   - regenerates `uv.lock` against the patched `pyproject.toml`, runs
     `uv sync --no-dev --extra rknn`;
   - for each model in `config/models.yaml`, runs
     `python -m immich_model_exporter export <name> <source> --target-platform rk3588`;
   - strips everything but `*.rknn`, re-roots the tree under `clip/` /
     `facial-recognition/`, tarballs it;
   - `crane append`s the tarball onto an arm64-pinned `busybox:1.37` and
     pushes to GHCR with provenance labels.

## Triggering a build

| Event | Behavior |
| --- | --- |
| `workflow_dispatch` | Manual run. `immich_version` overrides the auto-detected latest tag. `force_rebuild=true` skips the "tag already exists" check. |
| `schedule` (`0 5 * * *` UTC) | Daily check against the latest Immich release; no-op when the image is already current. |
| `push` to `main` touching patches/config/workflow | Always rebuilds the latest release tag. |

## Consuming the image

Use it as an `initContainer` on the Immich machine-learning pod. The image's
single layer is rooted at `/`, so an `initContainer` that mounts the ML cache
PVC at `/cache` can simply `cp -rn /clip /facial-recognition /cache/` (with a
per-file `sha256sum` check if you want idempotent refresh on tag bump).

For a working example, see the [homelab Immich values][homelab-values] in
`isac322/homelab`.

[homelab-values]: https://github.com/isac322/homelab/blob/master/values/immich/backbone.yaml

## License

The patches and CI tooling in this repo are released under AGPL-3.0 (matching
Immich). The exported `.rknn` artifacts derive from the upstream HuggingFace
models published under their respective licenses by `immich-app/*`; this repo
only recompiles them for the RKNPU.
