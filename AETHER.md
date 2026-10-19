# Termux-Aether runtime builds

This fork owns Android ARM64 CLI and Google Tensor dispatch builds for
[Termux-Aether](https://github.com/wallentx/termux-aether-app). It does not modify
LiteRT-LM inference code yet. The app consumes a separately built runtime;
the runtime is not currently bundled in its APK.

## Fork release

`AETHER_VERSION` is the release authority for this downstream fork. The first
fork release is `0.17.1.1000`, published under the reserved fork tag
`aether/v0.17.1.1000`. It packages the Android ARM64 runtime and provenance from
this fork; it is not an upstream LiteRT-LM `0.17.1.1000` release. The final
component is an Aether-only sequence and future fork patches increment it.
Upstream `v*` tags remain unmodified.

The upstream base remains LiteRT-LM `v0.17.1` at
`5e58e9a0aef7abf7091207a8b1d1063a1c800f08`. Release artifacts record both the
fork release/tag and that upstream base. A future upstream update must change the
pin and its documented upstream version independently of `AETHER_VERSION`.

## Source and build

The initial downstream changes are based on LiteRT-LM v0.17.1,
commit `5e58e9a0aef7abf7091207a8b1d1063a1c800f08`. Its WORKSPACE pins
LiteRT to `9fe5be45564c868408e6514c8aabb83e211a0911`; that dependency remains
upstream-owned unless a concrete source patch requires a separate fork.

Run the manual **Pixel TPU runtime** GitHub Actions workflow on the downstream
release revision. It validates `AETHER_VERSION`, builds the checked-out fork
revision using NDK r28b and the repository's Bazel version, and records both
versions in the artifact. When advancing the upstream base, update LITERT_LM_REF
and its documented upstream version together. No model weights or Hugging Face
tokens are required by CI.

The `pixel-tpu-arm64` artifact contains:

- `litert_lm_main`, the Android ARM64 CLI.
- `libLiteRtDispatch_GoogleTensor.so` and `libLiteRt.so`, built from the same
  pinned LiteRT dependency tree, plus the release's ARM64 prebuilt libraries.
- Fork and upstream-base versions, source revisions, toolchain versions, ELF
  metadata, licenses, and SHA256SUMS.

The runtime packaging queries `litert_runtime_c_api_so` directly. The similarly
named `litert_runtime_c_api_shared_lib` is a C++ wrapper target; querying its
default outputs is not a reliable way to collect the runtime shared object.
Build and query commands use identical Bazel JVM startup options to avoid
restarting the server during packaging.

## Validation status

The corrected fork workflow passed in [CI run 36987508784](https://github.com/wallentx/termux-aether-litert-lm/actions/runs/36987508784).
Its artifact and shell-side copies were checksum-verified on the Pixel 11 Pro XL.
LiteRT-LM v0.17.1 generated `READY` using the official Gemma4-E2B Tensor G6 model.
Logs confirm Google Tensor dispatch handled the main model's prefill/decode
subgraphs through `libedgetpu_litert.so`; auxiliary operations also use CPU kernels.

This was a short functional test (16 input tokens, 2 output tokens), not a sustained
performance benchmark. The earlier mask-element error did not reproduce with
this coherent runtime. Local evidence and a rerun script are saved under
`.local/benchmarks/tpu-20261002/` (ignored, not part of the release).

Verify SHA256SUMS before staging an artifact. Test with a model compiled for the
specific Tensor generation. Check generated output and completed prefill/decode,
not just exit status: an earlier CLI returned zero after an inference error.
TPU dispatch initialization alone does not demonstrate successful generation.

The tested older v0.11.0 runtime with dispatch v2.1.6 initialized the Pixel's
Tensor G6 TPU but rejected the Gemma4-E2B model's mask element type. This is why
we are building a coherent newer runtime instead of mixing prebuilt versions.
