# VisionEngine — Pre-Integration Materials

**Revision:** 1
**Last modified:** 2026-07-15T11:20:43Z
**Purpose:** Consolidated pre-integration materials (gate before any integration/deployment work).

> This document consolidates and verifies the pre-integration materials for the
> `VisionEngine` sub-system. Every statement below is grounded in the real repo
> at the path this document lives in; where the repo does not determine a fact,
> it is marked `UNKNOWN:` with the reason (§11.4.6 — no guessing). It
> cross-references the existing `ARCHITECTURE.md` / `API_REFERENCE.md` /
> `README.md` / `CLAUDE.md` rather than restating them.

---

## 1. Purpose / What it is

VisionEngine is a **standalone, project-not-aware Go toolkit for computer-vision
+ LLM-vision UI analysis and navigation-graph construction** (`README.md:1-6`,
`CLAUDE.md` §1). It is a **library-plus-CLI module, not a long-running network
service** — the codebase exposes reusable `pkg/*` APIs plus two thin CLI bridges
(`cmd/visiondescribe`, `cmd/videoverdict`); it opens no inbound listener (see
§5–§6).

Go module identity: `digital.vasic.visionengine` (`go.mod:1`).

It is designed to be **incorporated by a consuming project (and HelixQA) as an
equal-codebase submodule** and imports nothing from any consumer
(`README.md:4-6,110-112`; `helix-deps.yaml:16-17` → `deps: []`). In the Helix
stack it is the `§11.4.117` CV/OCR pixel-oracle / `§11.4.160` vision-verified
recording component: the CLI `videoverdict` is explicitly built to answer the
`§11.4.107` liveness, `§11.4.117` pixel-oracle, and `§11.4.137` subtitle-content
questions on captured video frames (`cmd/videoverdict/main.go:10-18`).

Version signal: `0.1.0` (from `CHANGELOG.md`; there is **no** `VERSION` file and
`go.mod` carries no semantic-version directive — `CLAUDE.md` "Versioning note").

---

## 2. Architecture overview

Full architecture lives in `ARCHITECTURE.md` and `docs/ARCHITECTURE.md` — not
restated here. Verified real package tree (`find pkg -maxdepth 2 -name '*.go'`):

| Package (real dir) | Role | Key source |
|---|---|---|
| `pkg/analyzer` | `Analyzer` / `VideoProcessor` interfaces + value types + `StubAnalyzer` reference impl (returns an explicit error, never a fabricated result, when no backend is wired) | `analyzer.go`, `stub.go`, `types.go` |
| `pkg/graph` | `NavigationGraph` — BFS `PathTo`, coverage, `ExportDOT`/`ExportJSON`/`ExportMermaid` | `graph.go`, `export.go` |
| `pkg/llmvision` | `VisionProvider` interface + **8 provider adapters** (OpenAI, Anthropic, Gemini, Qwen, Kimi, StepGUI/StepFun, Astica, Ollama) + `FallbackProvider` chain | `provider.go`, `openai.go`, `anthropic.go`, `gemini.go`, `qwen.go`, `kimi.go`, `stepgui.go`, `astica.go`, `ollama.go`, `fallback.go` |
| `pkg/opencv` | Mechanical CV layer — **stubs by default** (`//go:build !vision`, `factory.go:4`), real GoCV/OpenCV bindings behind `-tags vision` (`//go:build vision` on `detector_vision.go`, `differ_vision.go`, `color_vision.go`, `feature_vision.go`, `video_vision.go`, `factory_vision.go`) | `interfaces.go`, `stub.go`, `*_vision.go` |
| `pkg/config` | Env-var configuration loader + i18n-routed validation | `config.go` |
| `pkg/i18n` | Dependency-free `Translator` interface + `NoopTranslator` default + `bundles/` | `translator.go` |
| `pkg/remote` | SSH-driven remote/distributed Ollama + llama.cpp-RPC worker management (outbound SSH only) | `remote.go`, `ssh.go`, `deployer.go`, `distributed.go` |

Outside `pkg/`: `cmd/visiondescribe/` (screenshot → structured JSON UI
description CLI), `cmd/videoverdict/` (frame-dir → video-level SEE verdict CLI),
`internal/archdoc/` (internal arch-doc generator), `challenges/runner/` +
`challenges/scripts/` (anti-bluff evidence runner + shell Challenges).

**Vision/OCR pipeline (grounded):**
- **Two layers** (`ARCHITECTURE.md:55-69`): Layer 1 = mechanical GoCV/OpenCV
  (element detection, SSIM comparison, color analysis, ORB features, video —
  `pkg/opencv/*_vision.go`, `-tags vision` only); Layer 2 = LLM-vision
  (screen identification, UI comprehension, issue detection — `pkg/llmvision`).
- **OCR / text reading is done LLM-side, not by a bundled OCR engine.** The CLI
  prompts a vision model to READ on-screen text and burned-in subtitles verbatim
  (`cmd/videoverdict/main.go:64-75`; `cmd/visiondescribe/main.go:50-63`).
  `UNKNOWN:` — **no** Tesseract / gosseract / Leptonica / ffmpeg dependency
  exists anywhere in this module (verified: `grep -rniE 'tesseract|gosseract|ffmpeg|leptonica'`
  over `*.go` / `Dockerfile*` / `*.md` returns nothing). `videoverdict` consumes
  **already-extracted** frame images (a dir / glob / comma-list of PNG/JPG —
  `cmd/videoverdict/main.go:498-542`); frame extraction is the caller's job.

**Stack:** Go (`go 1.25.3`, `go.mod:3`). No database, cache, web framework, or
GUI-toolkit dependency (`CLAUDE.md` §2). Thread safety via `sync.RWMutex`
(`ARCHITECTURE.md:96-100`).

---

## 3. Dependencies

**Own-org submodule dependencies:** none. `helix-deps.yaml:16-17` declares
`schema_version: 1` / `deps: []` (a leaf Go submodule per its own comment,
`helix-deps.yaml:4-8`). There is **no** `.gitmodules` file and **no** compose
file in this repo (verified).

**Direct third-party Go modules (`go.mod:5,18-20`):**
- `github.com/stretchr/testify v1.11.1` — test assertions.
- `gocv.io/x/gocv v0.43.0` — OpenCV bindings, compiled **only** under `-tags vision`.
- `golang.org/x/crypto v0.51.0` — SSH client for `pkg/remote` (`pkg/remote/ssh.go`).
- Indirect (`go.mod:7-15`): `davecgh/go-spew`, `kr/pretty`, `pmezard/go-difflib`,
  `rogpeppe/go-internal`, `golang.org/x/sys`, `gopkg.in/check.v1`, `gopkg.in/yaml.v3`.

**System / native deps (only for the `-tags vision` path):** OpenCV 4.x dev
headers, installed by `Dockerfile.vision:8-14` — `build-essential`, `cmake`,
`pkg-config`, `libopencv-dev`, `libopencv-contrib-dev`. The default build path
(no `vision` tag) needs **none** of these and is buildable/testable on any
Go 1.25.3+ host (`README.md:28-30`).

**External runtime services (client-side, optional, config-driven — not bundled):**
LLM vision provider HTTP APIs (OpenAI, Anthropic, Google/Gemini, Qwen, Kimi,
StepFun, Astica) and/or a local/remote **Ollama** instance and **llama.cpp RPC**
workers (`ARCHITECTURE.md:20-53`; `docs/USAGE.md`). All reached **outbound** as a
client; none is a dependency the module hosts. Selection is honest / evidence-
driven — with no vision key configured the CLIs exit `3` OPERATOR-BLOCKED and
never fabricate a result (`cmd/visiondescribe/main.go:125-134`,
`cmd/videoverdict/main.go:188-192`).

---

## 4. Deploy / Distribution design

**Primary distribution = source-consumed Go submodule.** VisionEngine is added
by a consuming project as an equal-codebase submodule and imported via its
public `pkg/*` API (`README.md:4-6,64-75`; `CLAUDE.md` §6, §8). There is no
published binary or service artifact in this repo.

**Container image = self-contained build + test harness, not a service image.**
`Dockerfile.vision`:
- Base `golang:1.24-bookworm` (`Dockerfile.vision:5`). **Note (flagged, not
  resolved): this is Go 1.24, while `go.mod:3` pins `go 1.25.3`** — a
  version-skew worth reconciling before relying on the image for the
  `go 1.25.3` toolchain.
- Installs OpenCV 4.x dev deps and verifies via `pkg-config`
  (`Dockerfile.vision:7-17`).
- **Rootless-friendly:** creates an unprivileged `vision` user (uid/gid `10001`)
  that owns `/app` + the Go caches, and switches to it with `USER 10001:10001`
  before build — never root at runtime (`Dockerfile.vision:19-28`; composes the
  Helix rootless-container posture, §11.4.161).
- Builds with the vision tag (`RUN go build -tags vision ./...`,
  `Dockerfile.vision:36`).
- **Entrypoint / default command** = the test suite:
  `CMD ["go","test","-tags","vision","./...","-race","-count=1","-timeout","120s"]`
  (`Dockerfile.vision:39`). Running the image runs the OpenCV-backed test suite;
  it does **not** start a server or a persistent process.

**Distribution slice:** the module ships source (`pkg/`, `cmd/`, `internal/`,
`challenges/`) + docs + `Makefile` + `Dockerfile.vision`. The two CLIs
(`visiondescribe`, `videoverdict`) are built/run on demand by a consumer
(`go run ./cmd/...` or a `go build` binary); `Makefile` wires `build`,
`build-vision`, `test`, `test-race`, `test-vision`, etc. (`CLAUDE.md` §4).

---

## 5. Ports

`UNKNOWN:` — **VisionEngine exposes no inbound network port / listener.**
Verified: `Dockerfile.vision` has **no `EXPOSE`**; a repo-wide grep for
`EXPOSE|ListenAndServe|http.Listen|.Serve(|:8080|:9090` finds no inbound server
in non-test source. The only `net/http` usage is **client-side** in the
`pkg/llmvision` provider adapters (`gemini.go`, `openai.go`, `anthropic.go`,
`kimi.go`, `qwen.go`, `stepgui.go`, `astica.go`, `ollama.go`) plus `httptest`
in tests. `pkg/remote` makes **outbound** SSH connections
(`net.JoinHostPort` → `ssh.Dial("tcp", ...)`, `pkg/remote/ssh.go:162-168`).

Outbound-only port references (for firewall/egress planning, not listeners):
- Provider HTTPS endpoints (443) — reached by the `pkg/llmvision` adapters.
- SSH — default `22` for remote hosts (`docs/USAGE.md:85`).
- Ollama API — default `11434` (`docs/USAGE.md:86`, `ARCHITECTURE.md:42`).

---

## 6. Health

`UNKNOWN:` — **no HTTP/gRPC health endpoint exists** (no server; no
`/health`, `/healthz`, or `/ready` handler anywhere in non-test source —
verified by grep). Because the module is a library + short-lived CLIs + a
test-harness container, the applicable "is it working" signals are instead:
- **CLI exit-code contracts** — `visiondescribe`: `0` described / `2` bad input
  / `3` no provider (OPERATOR-BLOCKED) / `4` provider unreachable
  (`cmd/visiondescribe/main.go:28-29`); `videoverdict`: same `0/2/3/4` contract
  (`cmd/videoverdict/main.go:43-44`), plus a `-probe` flag reporting which vision
  providers are reachable (`cmd/videoverdict/main.go:146-149`).
- **Test suite** — the `Dockerfile.vision` CMD is itself the container's
  liveness proof (the `-tags vision` race test suite).
- **Anti-bluff Challenge** — `challenges/scripts/visionengine_describe_challenge.sh`
  (normal exit `0`, `--mutate` exit `99`) (`README.md:47-50,96-102`).

There is a shell Challenge named for "DDoS/health-flood" in
`challenges/scripts/` (`CLAUDE.md:110`, `AGENTS.md:173,265`) — that is a
resilience/stress Challenge script, **not** a served health endpoint; do not
mistake it for an HTTP `/health`.

---

## 7. How it boots

There is **no daemon boot** — the module does not start a service. Real entry
points:

1. **As a library (primary use):** a consumer imports `digital.vasic.visionengine/pkg/*`
   and constructs objects directly — e.g. `graph.NewNavigationGraph()` /
   `analyzer.NewStubAnalyzer()` / `llmvision.NewGeminiProvider(...)`
   (`README.md:64-75`, `docs/USAGE.md:30-53,116-134`). No process lifecycle.
2. **`cmd/visiondescribe`:** invoked directly —
   `visiondescribe -image shot.png [-focus ...] [-provider auto]` or
   `visiondescribe -probe`; reads env vars + a PNG, prints structured JSON to
   stdout, exits per the §6 code contract (`cmd/visiondescribe/main.go:23-30,95-171`).
3. **`cmd/videoverdict`:** invoked directly —
   `videoverdict -frames DIR [-sample 6] [-expect-subtitle "TEXT"] ...` or
   `-probe` / `-from-frames-json`; reads a frame set, runs one real vision call
   per sampled frame, prints the aggregate SEE verdict JSON, exits per the §6
   contract (`cmd/videoverdict/main.go:36-44,129-228`).
4. **`challenges/runner/`:** `go run ./challenges/runner/` — exercises the public
   API end-to-end with captured evidence (`README.md:44-45`).
5. **Container:** `docker/podman run` the `Dockerfile.vision` image → runs the
   `-tags vision` test suite once and exits (`Dockerfile.vision:39`); rootless
   (`USER 10001:10001`). Not a served process.

Configuration is entirely env-var driven at each invocation (`pkg/config`,
`.env.example`; `CLAUDE.md` §7) — e.g. `HELIX_VISION_PROVIDER`, the per-provider
`*_API_KEY` set, `HELIX_OLLAMA_URL`, `HELIX_VISION_HOSTS`, `HELIX_LLAMACPP_RPC_*`
(verified key list in `.env.example`).

---

## 8. Materials status (verify pass)

Verdict legend: **HAS-VERIFIED** = the source material exists in-repo and this
document cites it; **GAP-FILLED** = this document had to state `UNKNOWN:` /
derive the answer because the repo has no dedicated artifact for it.

| Pre-integration material | Present in repo? | Evidence | Verdict |
|---|---|---|---|
| Purpose / identity | Yes | `README.md:1-6`, `CLAUDE.md` §1, `go.mod:1` | HAS-VERIFIED |
| Architecture overview | Yes | `ARCHITECTURE.md`, `docs/ARCHITECTURE.md`, real `pkg/*` tree | HAS-VERIFIED |
| API surface | Yes | `API_REFERENCE.md`, `pkg/*` sources | HAS-VERIFIED |
| Dependency manifest (own-org) | Yes | `helix-deps.yaml` (`deps: []`) | HAS-VERIFIED |
| Go/third-party deps | Yes | `go.mod`, `go.sum` | HAS-VERIFIED |
| System/native deps | Yes | `Dockerfile.vision:7-17` (OpenCV 4.x) | HAS-VERIFIED |
| Container / build image | Yes | `Dockerfile.vision` | HAS-VERIFIED |
| Rootless posture | Yes | `Dockerfile.vision:19-28` (uid/gid 10001) | HAS-VERIFIED |
| Distribution model | Yes | `README.md:4-6`, `CLAUDE.md` §6/§8 (source submodule) | HAS-VERIFIED |
| Config / env vars | Yes | `.env.example`, `pkg/config`, `CLAUDE.md` §7 | HAS-VERIFIED |
| Boot / entrypoints | Yes | `cmd/*/main.go`, `Dockerfile.vision:39`, `Makefile` | HAS-VERIFIED |
| Anti-bluff / test evidence | Yes | `docs/test-coverage.md`, `challenges/`, `README.md:77-131` | HAS-VERIFIED |
| **Inbound ports** | No listener | grep: no `EXPOSE`/`ListenAndServe`; `net/http` client-only | GAP-FILLED (`UNKNOWN:` — none by design) |
| **Health endpoint** | No | grep: no `/health`\|`/healthz`\|`/ready` in non-test source | GAP-FILLED (`UNKNOWN:` — none; exit-code + test-suite signals instead) |
| **Compose / `.gitmodules`** | Absent | no compose file, no `.gitmodules` (verified) | GAP-FILLED (`UNKNOWN:` — none present) |

**Open items to reconcile before integration (facts, not blockers):**
1. **Go version skew** — `go.mod:3` pins `go 1.25.3` but `Dockerfile.vision:5`
   uses `golang:1.24-bookworm`.
2. **`-tags vision` host prerequisite** — the OpenCV path needs OpenCV 4.x dev
   headers on the host / in the image; the default (stub) path does not
   (`README.md:28-30`, `CLAUDE.md` §4).
3. **No inbound service surface** — any integration expecting VisionEngine to
   run as a networked service is mismatched; it integrates as a Go library and
   two CLIs (see §5–§7).

**Overall verify-pass verdict: HAS-VERIFIED** — every claimed material is
grounded in an in-repo artifact; the three GAP-FILLED rows are honest
`UNKNOWN:` findings (the sub-system genuinely has no inbound port, no health
endpoint, and no compose/`.gitmodules`), not missing documentation.
