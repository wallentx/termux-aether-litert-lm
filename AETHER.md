# Termux-Aether runtime builds

This fork owns Android ARM64 CLI and Google Tensor dispatch builds for
[Termux-Aether](https://github.com/wallentx/termux-aether-app). It does not modify
LiteRT-LM inference code yet. The app consumes a separately built runtime;
the runtime is not currently bundled in its APK.

## Source and build

The initial downstream changes are based on LiteRT-LM v0.17.1,
commit `5e58e9a0aef7abf7091207a8b1d1063a1c800f08`. Its WORKSPACE pins
LiteRT to `9fe5be45564c868408e6514c8aabb83e211a0911`; that dependency remains
upstream-owned unless a concrete source patch requires a separate fork.

Run the manual **Pixel TPU runtime** GitHub Actions workflow on the downstream
branch. It builds the checked-out fork revision using NDK r28b and the repository's
Bazel version. When advancing the upstream base, update LITERT_LM_REF and the
release label in the workflow together. No model weights or Hugging Face tokens
are required by CI.

The `pixel-tpu-arm64` artifact contains:

- `litert_lm_main`, the Android ARM64 CLI.
- `libLiteRtDispatch_GoogleTensor.so` and `libLiteRt.so`, built from the same
  pinned LiteRT dependency tree, plus the release's ARM64 prebuilt libraries.
- Source revisions, toolchain versions, ELF metadata, licenses, and SHA256SUMS.

The runtime packaging queries `litert_runtime_c_api_so` directly. The similarly
named `litert_runtime_c_api_shared_lib` is a C++ wrapper target; querying its
default outputs is not a reliable way to collect the runtime shared object.
Build and query commands use identical Bazel JVM startup options to avoid
restarting the server during packaging.

## Validation status

The original app-hosted CI build compiled the v0.17.1 ARM64 targets successfully,
but packaging failed. This fork carries the packaging correction; its CI artifact
and device inference still need validation. No successful TPU text generation
is claimed yet.

Verify SHA256SUMS before staging an artifact. Test with a model compiled for the
specific Tensor generation. Check generated output and completed prefill/decode,
not just exit status: an earlier CLI returned zero after an inference error.
TPU dispatch initialization alone does not demonstrate successful generation.

The tested older v0.11.0 runtime with dispatch v2.1.6 initialized the Pixel's
Tensor G6 TPU but rejected the Gemma4-E2B model's mask element type. This is why
we are building a coherent newer runtime instead of mixing prebuilt versions.
