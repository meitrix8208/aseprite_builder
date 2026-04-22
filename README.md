# 🎨 Aseprite Builder

> Build [Aseprite](https://www.aseprite.org/) from source for **Windows**, **Linux**, and **macOS** using GitHub Actions — no manual compilation, no shady binaries, no EULA violations.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub Actions](https://img.shields.io/badge/CI-GitHub%20Actions-2088FF?logo=github-actions&logoColor=white)](https://github.com/meitrix8208/aseprite_builder/actions)
[![Platforms](https://img.shields.io/badge/Platforms-Windows%20%7C%20Linux%20%7C%20macOS-blue)](https://github.com/meitrix8208/aseprite_builder/actions)

---

## What is this?

An automated GitHub Actions workflow that compiles Aseprite from its official source code on demand. You pick a version (or just say `latest`), trigger the workflow, and minutes later you have a fresh build as a **draft release** in your repo.

**Why use it?**

- 🔒 **Safe** — compiled directly from the official [aseprite/aseprite](https://github.com/aseprite/aseprite) source on GitHub runners
- 📦 **EULA-friendly** — binaries are published as **draft releases** (only visible to the repo owner), not as public artifacts
- 🎯 **Version-pinned** — build any released tag (`v1.3.7`, `v1.3.14.2`, etc.) or grab the latest one automatically
- ⚡ **Cached** — Skia dependencies are cached per-OS for faster subsequent builds

---

## 🚀 Usage

### 1. Fork or clone this repo

```bash
git clone https://github.com/meitrix8208/aseprite_builder.git
cd aseprite_builder
```

### 2. (Optional) Pick your target platforms

Edit `.github/workflows/aseprite_build_deploy.yml` and adjust the `os` matrix to only the platforms you need:

```yaml
strategy:
  matrix:
    os: [windows-latest, ubuntu-latest, macOS-latest]
```

> 💡 Skip this if you want all three. See [build times](#-build-times) for the cost breakdown.

### 3. Trigger the build

You have two options:

**Option A — Manual dispatch (recommended):**

1. Go to the **Actions** tab on your fork
2. Select **Build and deploy Aseprite new version**
3. Click **Run workflow**
4. Enter a version tag (e.g. `v1.3.7`) or leave it as `latest`
5. Hit **Run workflow**

**Option B — Push to `master`:** any push to the `master` branch triggers a build of the latest Aseprite release.

### 4. Grab your build

Once the workflow finishes, a **draft release** appears in your repo's Releases tab containing the `.zip` package for each OS you built. Only you (the repo owner) can see it.

---

## ⚙️ How it works

The workflow is split into **three jobs** that run in sequence:

```text
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  prepare-build   │ ──▶ │  build-aseprite  │ ──▶ │  create-release  │
│  (Ubuntu)        │     │  (matrix: 3 OS)  │     │  (Ubuntu)        │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

### Job 1 — `prepare-build`

Resolves which Aseprite version to build and extracts its metadata.

- Reads the `aseprite_version` input (defaults to `latest`)
- Queries the GitHub API for either the latest release or the specified tag
- Extracts the source zip URL and the release notes
- Fails early if the requested tag doesn't exist
- Exposes `selected_tag`, `selected_download_url`, and `version_info` as outputs for downstream jobs

### Job 2 — `build-aseprite`

Runs in parallel across the OS matrix (`fail-fast: false`, so one platform's failure doesn't kill the others).

1. **Install OS-specific dependencies**
   - **Windows:** Ninja (via `gha-setup-ninja`), and strips the preinstalled OpenSSL from `PATH` to avoid conflicts
   - **Ubuntu:** `g++`, `clang`, `cmake`, `ninja-build`, and X11/GL/fontconfig dev libraries
   - **macOS:** `ninja` and `p7zip` via Homebrew
2. **Restore Skia from cache** (key: `skia-<os>-m124-cache`)
3. **Download Skia** (release `m124-08a5439a6b` from `aseprite/skia`) if the cache missed
4. **Download and unzip the Aseprite source** using the URL resolved in job 1
5. **Configure the Visual Studio environment** (Windows only, via `gha-setup-vsdevenv`, x64)
6. **Run CMake** with `-G Ninja`, `LAF_BACKEND=skia`, and the cached Skia paths
   - macOS pins `x86_64` with deployment target `10.9` against the Xcode SDK
   - Windows ignores Chocolatey/Strawberry Perl paths to avoid picking up stray toolchains
7. **Build with `ninja aseprite`**
8. **Clean up generator binaries** (`gen`, `modp_b64_gen`, and their `.exe` variants)
9. **Make it portable on Windows** by dropping an empty `aseprite.ini` next to the binary
10. **Package** the `bin/` directory as `Aseprite-<tag>-<OS>.zip` using 7-Zip
11. **Upload the zip as a GitHub artifact** (30-day retention)

### Job 3 — `create-release`

Runs only if `prepare-build` succeeded (builds can partially fail and you'll still get releases for the successful ones).

- Downloads all build artifacts
- Creates a **draft release** tagged with the Aseprite version, containing:
  - All `Aseprite-*.zip` packages
  - The original Aseprite release notes as the body
- Uses `allowUpdates: true`, so re-running for the same tag updates the existing draft instead of failing

---

## 🔧 Configuration at a glance

| Setting | Value | Where |
| :--- | :--- | :--- |
| Build type | `Release` | `env.BUILD_TYPE` |
| Skia version | `m124-08a5439a6b` | Skia download step |
| LAF backend | `skia` | CMake flag |
| Windows arch | `x64` | `VsDevCmd.bat -arch=x64` |
| macOS arch | `x86_64` | `CMAKE_OSX_ARCHITECTURES` |
| macOS min target | `10.9` | `CMAKE_OSX_DEPLOYMENT_TARGET` |
| Artifact retention | 30 days | `upload-artifact` |
| Release visibility | Draft | `ncipollo/release-action` |

---

## ⏱ Build times

GitHub gives you **2,000 free Actions minutes per month** on public repos with private forks. Each OS burns minutes at a different rate:

| Operating System | Real minutes | Multiplier | Billed minutes |
| :--- | :--- | :--- | :--- |
| 🐧 Ubuntu | ~10 | ×1 | **10** |
| 🪟 Windows | ~20 | ×2 | **40** |
| 🍎 macOS | ~8 | ×10 | **80** |
| | | **Total (all 3)** | **130** |

> ⚠️ A full three-OS build costs ~130 billed minutes. If you only need one platform, trim the matrix and you'll get ~15× more builds out of the same quota. See the [GitHub Actions billing docs](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) for details.

---

## 📁 Project structure

```text
aseprite_builder/
├── .github/
│   └── workflows/
│       └── aseprite_build_deploy.yml   # The entire build pipeline
├── LICENSE                              # MIT
└── README.md
```

---

## 💚 Support Aseprite

This workflow compiles Aseprite from source, which is perfectly legal thanks to its open source code — but Aseprite is an amazing tool made by people who deserve your support. If you use it regularly, please buy a license:

👉 **[aseprite.org/#buy](https://aseprite.org/#buy)**

---

## 📄 License

MIT — see [LICENSE](LICENSE).
