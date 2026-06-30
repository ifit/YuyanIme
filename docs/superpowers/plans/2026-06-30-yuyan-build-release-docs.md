# YuyanIme Build & Release Documentation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Document the verified offline-APK build + distribution process for `ifit/YuyanIme` (a new `CLAUDE.md` runbook and an iFIT section in `README.md`), and open a FRO Task tracking the docs plus the 2026-06 build already produced.

**Architecture:** Two doc deliverables in the repo on branch `FRO-yuyan-build-docs` (CLAUDE.md as the single source of truth; README links to it), plus one FRO Task created via the Atlassian MCP. No code or build-logic changes.

**Tech Stack:** Markdown docs; git; Atlassian (Jira) MCP for the ticket.

## Global Constraints

- Branch: all repo changes go on `FRO-yuyan-build-docs` (already created; design doc already committed there). Never commit to `main`.
- Never commit secrets: `*.jks`, `yuyanime_keystore`, `yuyanime.properties`, `local.properties` are gitignored — keep it that way; do not add credential values to any doc.
- Shipped variant is **offline**: `com.yuyan.pinyin.offline.release`.
- Build requires **JDK 17** (JDK 21 fails `kspOnlineReleaseKotlin` JVM-target check).
- Distribution filename format: `com.yuyan.pinyin.offline.release-<versionName>.<versionCode>.apk` (e.g. `com.yuyan.pinyin.offline.release-20260701.05.2026070105.apk`).
- Verified build this session: versionName `20260701.05`, versionCode `2026070105`, signed alias `ifit_yuyanime` (O=iFIT), v1+v2 verified.
- Commit message footer: `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
- Confluence refs: WOLF page 3664511006 (Admin Portal upload); VC1 page 3824386272 (China S3 access steps 1-7).
- Jira: project FRO, type Task, assignee Chris Ripple (`557058:1f54dd3a-3e1f-4842-8445-0b5ce70a5535`), cloudId `9d239a8a-1dba-4e4c-94d6-3e096b3985ce`.

---

## File Structure

- **Create** `CLAUDE.md` (repo root) — authoritative build + release runbook.
- **Modify** `README.md` (repo root) — add English "## iFIT Build & Release" section near top, pointing to CLAUDE.md; leave upstream Chinese content intact.
- **External** — one FRO Task created via Atlassian MCP (no repo file).

---

### Task 1: Write `CLAUDE.md` runbook

**Files:**
- Create: `CLAUDE.md`

**Interfaces:**
- Produces: the canonical runbook that README will link to (anchor target: file `CLAUDE.md` at repo root).

- [ ] **Step 1: Create `CLAUDE.md` with the full runbook**

Write the file with exactly these sections and content:

````markdown
# CLAUDE.md — YuyanIme (语燕输入法) Build & Release

This is iFIT's fork of [`gurecn/YuyanIme`](https://github.com/gurecn/YuyanIme), the China
"语燕 / Yuyan" Rime-based input method (keyboard) shipped on China consoles. This file is the
authoritative runbook for building and releasing the **offline** APK. (The upstream Chinese
`README.md` describes the original project and a *different* signing layout — follow this
file, not the README, for iFIT builds.)

## Repo overview

- Modules: `app` (the IME app) + `yuyansdk` (the IME engine, a **git submodule** → `ifit/yuyansdk`).
- Package: `com.yuyan.pinyin.<flavor>.release`. Flavors: `online`, `offline`. **iFIT ships `offline`.**
- App label: 语燕输入法. Default branch: `main`. No CI — this is a manual local Gradle build.
- Version is auto-derived from build time in **GMT+8**: `versionName = yyyyMMdd.HH`,
  `versionCode = yyyyMMddHH` (see `app/build.gradle`).

## Prerequisites

- **JDK 17.** Building with JDK 21 fails with
  `Inconsistent JVM-target compatibility detected ... (17) and ... (21)` on
  `kspOnlineReleaseKotlin`. If your default JDK is not 17, point Gradle at a JDK 17, e.g.
  add to `gradle.properties` (do not commit this line):
  `org.gradle.java.home=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home`
- **Android SDK** installed; create `local.properties` with `sdk.dir=/path/to/Android/sdk`.
- Clone with submodules:
  ```sh
  git clone --recurse-submodules git@github.com:ifit/YuyanIme.git
  # or, in an existing clone:
  git submodule update --init --recursive
  ```

## Signing setup (one-time per machine)

The signing keystore and its properties file are in **1Password → Valinor Vault**. There is
no `op` automation on the build machine — **download both items manually**.

1. From the Valinor Vault, download the Yuyan keystore file and the `yuyanime.properties` file.
2. Place the keystore somewhere stable, e.g. `~/.gradle/yuyanime_keystore`.
3. Put the properties at **`~/.gradle/yuyanime.properties`** (the non-CI path `app/build.gradle`
   reads). Set `yuyanime_app_keystore_file` to the **absolute** path of the keystore:
   ```properties
   yuyanime_app_keystore_file=/Users/<you>/.gradle/yuyanime_keystore
   yuyanime_app_keystore_password=<from 1Password>
   yuyanime_app_keystore_key_alias=ifit_yuyanime
   yuyanime_app_keystore_key_password=<from 1Password>
   ```
   > Never commit the keystore or properties. `*.jks`, `yuyanime_keystore`, and
   > `local.properties` are gitignored — keep credentials out of the repo.
4. (Optional) verify the keystore opens:
   ```sh
   keytool -list -keystore ~/.gradle/yuyanime_keystore -alias ifit_yuyanime
   ```
   Expect a `PrivateKeyEntry` with an O=iFIT certificate.

## Build the offline release APK

```sh
./gradlew :app:assembleOfflineRelease
```

> **Gotcha:** `build.gradle` lists the Aliyun maven mirror first. It throws intermittent
> `502 Bad Gateway` errors that disable the repo for the rest of the build. If you hit a
> dependency-resolution failure mentioning `maven.aliyun.com`, just re-run — it's transient.

Output:
```
app/build/outputs/apk/offline/release/yuyanIme_<versionCode>_offline_release.apk
```

## Verify the build (sanity check)

```sh
# signature — expect v1 + v2 = true
$ANDROID_HOME/build-tools/<latest>/apksigner verify --verbose <apk>

# package + version — expect com.yuyan.pinyin.offline.release and the versionCode in the filename
$ANDROID_HOME/build-tools/<latest>/aapt2 dump badging <apk> | grep -E "^package:"
```

## Rename for distribution

Admin/CDN require the filename format `com.xx.xxx-versionname.versioncode.apk`:

```sh
cp app/build/outputs/apk/offline/release/yuyanIme_<versionCode>_offline_release.apk \
   com.yuyan.pinyin.offline.release-<versionName>.<versionCode>.apk
# e.g. com.yuyan.pinyin.offline.release-20260701.05.2026070105.apk
```

## Distribution Part A — Admin Portal

Per WOLF doc "How To Update 3rd Party Apps"
(https://ifitdev.atlassian.net/wiki/spaces/WOLF/pages/3664511006):

1. Log into the Admin Portal → `Wolf Updates` → `App Updates`.
2. Click `Create` (top right).
3. On the `Upload File` tab, upload the renamed APK (filename must be
   `com.yuyan.pinyin.offline.release-<versionName>.<versionCode>.apk`).
4. On the `General` tab, fill in the app info — the **FQN** and **versionCode** must match
   the uploaded APK — then `Save`.

## Distribution Part B — China S3 / CDN

The APK must also be uploaded to the China CDN, served at
`https://ifit-wolf.svc.ifit.cn/android/builds/public/`. Steps 1-7 follow VC1 doc
"Pulling Customer Logs from China S3"
(https://ifitdev.atlassian.net/wiki/spaces/VC1/pages/3824386272):

1. Get the necessary AWS permissions from Platform.
2. **Do not be on the VPN.**
3. Visit https://ifitsso.awsapps.com/start/#/?tab=applications
4. Click **AWS China SSO Portal**.
5. Click **gateway-cn-production** to expand.
6. Click the `ifit-mobile-team-role-production` url.
7. Find the **S3** service and click into it.
8. Click into the `ifit-china-proxy-svc-production-ifit-wolf` bucket.
9. Click into `android` → `builds` → `public`.
10. Click **Upload** and upload the renamed APK.
````

- [ ] **Step 2: Verify the file reflects current repo facts**

Run:
```sh
cd /Users/chris.ripple/workspace/china-keyboard-task/YuyanIme
test -f CLAUDE.md && echo "CLAUDE.md exists"
grep -q "assembleOfflineRelease" CLAUDE.md && echo "build cmd ok"
grep -q "ifit-china-proxy-svc-production-ifit-wolf" CLAUDE.md && echo "cdn bucket ok"
grep -q "ifit_yuyanime" CLAUDE.md && echo "alias ok"
# confirm no secrets leaked into the doc
! grep -iE "password=.+[A-Za-z0-9]{6}" CLAUDE.md && echo "no secret values"
```
Expected: all five lines print their ok message.

- [ ] **Step 3: Commit**

```sh
git add CLAUDE.md
git commit -m "$(printf 'docs: add CLAUDE.md build & release runbook\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

### Task 2: Add iFIT section to `README.md`

**Files:**
- Modify: `README.md` (insert after the H1 title line `# 语燕输入法`, before the existing line 2)

**Interfaces:**
- Consumes: `CLAUDE.md` from Task 1 (link target).

- [ ] **Step 1: Insert the iFIT section**

Insert the following block immediately **after** the first line (`# 语燕输入法`) and before
the existing intro paragraph, so the upstream Chinese content stays intact below it:

```markdown

## iFIT Build & Release

> This repository is **iFIT's fork** of [`gurecn/YuyanIme`](https://github.com/gurecn/YuyanIme).
> iFIT ships the **offline** release APK (`com.yuyan.pinyin.offline.release`) to China consoles.
>
> **See [`CLAUDE.md`](./CLAUDE.md) for the authoritative build & release runbook** — JDK/signing
> prerequisites, the `./gradlew :app:assembleOfflineRelease` build, and the two distribution
> targets (Admin Portal + China S3/CDN at `ifit-wolf.svc.ifit.cn`).
>
> The Chinese content below is the upstream project's original README.

---
```

- [ ] **Step 2: Verify README still has upstream content and the new link**

Run:
```sh
cd /Users/chris.ripple/workspace/china-keyboard-task/YuyanIme
grep -q "iFIT Build & Release" README.md && echo "section ok"
grep -q "CLAUDE.md" README.md && echo "link ok"
grep -q "实现功能" README.md && echo "upstream content intact"
head -3 README.md
```
Expected: all three ok messages; first line is still `# 语燕输入法`.

- [ ] **Step 3: Commit**

```sh
git add README.md
git commit -m "$(printf 'docs: add iFIT build & release section to README\n\nCo-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>')"
```

---

### Task 3: Create the FRO Task

**Files:** none (external — Atlassian MCP).

**Interfaces:**
- Consumes: deliverables from Tasks 1-2 (referenced in the ticket description).

- [ ] **Step 1: Create the Jira issue**

Call `mcp__atlassian__createJiraIssue` with:
- `cloudId`: `9d239a8a-1dba-4e4c-94d6-3e096b3985ce`
- `projectKey`: `FRO`
- `issueTypeName`: `Task`
- `assignee_account_id`: `557058:1f54dd3a-3e1f-4842-8445-0b5ce70a5535`
- `summary`: `Document YuyanIme offline APK build & release process + produce 2026-06 offline build`
- `description` (markdown):
  ```
  Context: Junmei Fu / Don Jordan requested a new Yuyan (语燕) keyboard build after the
  TSCH-847 fix ("click settings in keyboard → blank screen") merged to `main` (commit 8f038a9).

  Deliverables:
  - Added `CLAUDE.md` runbook documenting the offline APK build + distribution.
  - Added an "iFIT Build & Release" section to `README.md` pointing to it.
  - Produced and verified the 2026-06 offline build:
    `com.yuyan.pinyin.offline.release-20260701.05.2026070105.apk`
    (package com.yuyan.pinyin.offline.release, versionCode 2026070105, signed alias
    ifit_yuyanime / O=iFIT, v1+v2 verified).

  Distribution targets: Admin Portal (Wolf Updates → App Updates) and the China S3/CDN
  bucket ifit-china-proxy-svc-production-ifit-wolf → android/builds/public
  (served at ifit-wolf.svc.ifit.cn).

  References:
  - Repo: https://github.com/ifit/YuyanIme
  - WOLF "How To Update 3rd Party Apps": https://ifitdev.atlassian.net/wiki/spaces/WOLF/pages/3664511006
  - VC1 "Pulling Customer Logs from China S3": https://ifitdev.atlassian.net/wiki/spaces/VC1/pages/3824386272
  ```

- [ ] **Step 2: Verify the issue was created and assigned**

Call `mcp__atlassian__getJiraIssue` with the returned key, fields `["summary","assignee","issuetype","status"]`.
Expected: summary matches, assignee = Chris Ripple, issuetype = Task. Record the FRO key.

---

## Self-Review

- **Spec coverage:** CLAUDE.md runbook (Task 1) ✔; README iFIT section (Task 2) ✔; FRO Task covering docs + build (Task 3) ✔. All three deliverables mapped.
- **Placeholder scan:** No TBD/TODO; all doc content is literal; `<versionCode>`/`<you>` are intentional user-substituted tokens in the runbook, not plan placeholders.
- **Type consistency:** Filename format, package name, versionCode `2026070105`, alias `ifit_yuyanime`, cloudId, and account ID are identical across Global Constraints and all tasks.
