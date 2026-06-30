# Design: YuyanIme Offline APK Build & Release Documentation

**Date:** 2026-06-30
**Author:** Chris Ripple
**Status:** Approved

## Problem

The `ifit/YuyanIme` repo (the China "语燕 / Yuyan" keyboard, a fork of `gurecn/YuyanIme`)
has no documentation for how iFIT actually builds and ships the offline release APK. The
only README is the upstream Chinese one, which describes a different signing setup
(`/YuyanIme/keystore/keystore.properties`) than the fork actually uses
(`~/.gradle/yuyanime.properties`). There is no `CLAUDE.md`. Each time someone needs a
build (e.g. the 2026-06 request from Junmei Fu / Don Jordan after the TSCH-847 fix), the
process has to be rediscovered from memory.

## Goals

1. Capture the **verified** offline-APK build recipe and the full distribution path so any
   engineer or agent can reproduce it.
2. Document signing-credential retrieval (1Password Valinor Vault) and both upload targets
   (Admin Portal + China S3/CDN).
3. Track the work (docs + the 2026-06 build already produced) under a FRO Task assigned to
   Chris Ripple.

## Non-Goals

- No CI/automation of the build (it remains a manual local Gradle build).
- No changes to the build logic, signing config, or version scheme.
- No rewrite of the upstream Chinese README content.

## Deliverables

### 1. `CLAUDE.md` (new, repo root) — authoritative runbook

Source of truth for build + release. Written for both a human and a future agent. Sections:

- **Repo overview**: fork of `gurecn/YuyanIme`; `app` module + `yuyansdk` git submodule;
  IME engine `ifit/yuyansdk`; package `com.yuyan.pinyin.{online,offline}.release`
  (offline is the variant iFIT ships); app label 语燕输入法; default branch `main`; no CI.
- **Prerequisites**: JDK **17** (gotcha — JDK 21 fails `kspOnlineReleaseKotlin` with
  "Inconsistent JVM-target compatibility (17 vs 21)"; set `org.gradle.java.home` to a
  JDK 17); Android SDK (`local.properties` `sdk.dir`); clone with `--recurse-submodules`.
- **Signing setup**: keystore file + `yuyanime.properties` live in the **Valinor Vault in
  1Password** — download **both manually** (no `op` automation). Place the keystore, then
  put `yuyanime.properties` at `~/.gradle/yuyanime.properties` with an **absolute**
  `storeFile` path. Keys: `yuyanime_app_keystore_file`, `_password`, `_key_alias`
  (`ifit_yuyanime`), `_key_password`. Verify with `keytool -list` (O=iFIT cert).
- **Build**: `./gradlew :app:assembleOfflineRelease`. Gotcha — `build.gradle` lists the
  Aliyun maven mirror first; it throws intermittent 502s that disable the repo mid-build,
  so retry. Output: `app/build/outputs/apk/offline/release/yuyanIme_<versionCode>_offline_release.apk`.
  Version auto-derived from build time in GMT+8 (`versionName yyyyMMdd.HH`,
  `versionCode yyyyMMddHH`).
- **Verify (post-build sanity check)**: `apksigner verify --verbose` (expect v1+v2 true)
  and `aapt dump badging` (confirm package + versionCode).
- **Rename for distribution**: `com.yuyan.pinyin.offline.release-<versionName>.<versionCode>.apk`
  — e.g. `com.yuyan.pinyin.offline.release-20260701.05.2026070105.apk`.
- **Distribution Part A — Admin Portal** (from WOLF doc "How To Update 3rd Party Apps",
  page 3664511006): Admin Portal → `Wolf Updates` → `App Updates` → `Create` →
  `Upload File` tab (upload APK; filename must be `com.xx.xxx-versionname.versioncode`) →
  `General` tab (FQN + versionCode must match the upload) → `Save`.
- **Distribution Part B — China S3 / CDN** (served at
  `ifit-wolf.svc.ifit.cn/android/builds/public/`; steps 1-7 adapted from VC1 page
  3824386272 "Pulling Customer Logs from China S3"):
  1. Get AWS permissions from Platform.
  2. **Do not be on the VPN.**
  3. Visit `https://ifitsso.awsapps.com/start/#/?tab=applications`.
  4. Click **AWS China SSO Portal**.
  5. Click **gateway-cn-production** to expand.
  6. Click the `ifit-mobile-team-role-production` url.
  7. Find the **S3** service and click into it.
  8. Click into `ifit-china-proxy-svc-production-ifit-wolf`.
  9. Click into `android` → `builds` → `public`.
  10. Click **Upload** and upload the renamed APK.

### 2. `README.md` (update) — ifit-specific section

Append an English **"## iFIT Build & Release"** section near the top of the existing
README, leaving the upstream Chinese content intact. The section gives a short orientation
(this is the iFIT fork; we ship the offline release APK) and points to `CLAUDE.md` as the
authoritative runbook, so the steps live in exactly one place.

### 3. FRO Task

- **Project** FRO, **type** Task, **assignee** Chris Ripple
  (`557058:1f54dd3a-3e1f-4842-8445-0b5ce70a5535`).
- **Summary**: `Document YuyanIme offline APK build & release process + produce 2026-06 offline build`
- **Description**: context (Junmei Fu / Don Jordan request; TSCH-847 fix merged to `main`
  @ `8f038a9`), deliverables (README section + CLAUDE.md runbook), and the build produced
  (`com.yuyan.pinyin.offline.release-20260701.05.2026070105.apk`, signed iFIT cert, v1+v2).
  Links: YuyanIme repo, WOLF page 3664511006, VC1 page 3824386272.
- No labels / components / fix version (none requested).

## Key facts (verified this session)

- Build succeeds with JDK 17 via `org.gradle.java.home`; fails on JDK 21.
- `.jks` and `yuyanime_keystore` are already in `.gitignore` — no secret-commit risk.
- Today's verified build: versionName `20260701.05`, versionCode `2026070105`, package
  `com.yuyan.pinyin.offline.release`, signed alias `ifit_yuyanime` (O=iFIT), v1+v2 verified.

## Risks / Open Questions

- The China S3 bucket `ifit-china-proxy-svc-production-ifit-wolf` is the documented backing
  store for `ifit-wolf.svc.ifit.cn`; documented as the manual upload target per user
  direction.
- 1Password retrieval is manual (no `op` CLI on the build machine), so docs describe the
  manual download rather than `op://` references.
