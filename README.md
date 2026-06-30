# YuyanIme

iFIT's fork of [`gurecn/YuyanIme`](https://github.com/gurecn/YuyanIme), the Rime-based
Chinese input method (keyboard) shipped on iFIT China consoles. For the upstream project's
feature list and design notes, see the [original repository](https://github.com/gurecn/YuyanIme).

iFIT ships the **offline** release APK: `com.yuyan.pinyin.offline.release`.

> **[`CLAUDE.md`](./CLAUDE.md) is the canonical build & release runbook** (and the reference
> for AI agents). The steps below are the same process for humans; if the two ever differ,
> `CLAUDE.md` wins.

## Repo overview

- Modules: `app` (the IME app) + `yuyansdk` (the IME engine, a **git submodule** → `ifit/yuyansdk`).
- Package: `com.yuyan.pinyin.<flavor>.release`. Flavors: `online`, `offline`. **iFIT ships `offline`.**
- Default branch: `main`. No CI — this is a manual local Gradle build.
- Version is auto-derived from build time in **GMT+8**: `versionName = yyyyMMdd.HH`,
  `versionCode = yyyyMMddHH` (see `app/build.gradle`).

## Prerequisites

- **JDK 17.** Building with JDK 21 fails with `Inconsistent JVM-target compatibility` on
  `kspOnlineReleaseKotlin`. If your default JDK is not 17, point Gradle at a JDK 17 by adding
  this to `gradle.properties` (do **not** commit the line):
  `org.gradle.java.home=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home`
- **Android SDK** installed; create `local.properties` with `sdk.dir=/path/to/Android/sdk`.

## 1. Clone with submodules

```sh
git clone --recurse-submodules git@github.com:ifit/YuyanIme.git
# or, in an existing clone:
git submodule update --init --recursive
```

## 2. Signing setup (one-time per machine)

The signing keystore and its properties file live in **1Password → Valinor Vault**. There is
no `op` automation — **download both items manually**.

1. From the Valinor Vault, download the Yuyan keystore file and the `yuyanime.properties` file.
2. Place the keystore somewhere stable, e.g. `~/.gradle/yuyanime_keystore`.
3. Put the properties at **`~/.gradle/yuyanime.properties`** (the path `app/build.gradle` reads).
   Set `yuyanime_app_keystore_file` to the **absolute** path of the keystore:
   ```properties
   yuyanime_app_keystore_file=/Users/<you>/.gradle/yuyanime_keystore
   yuyanime_app_keystore_password=<from 1Password>
   yuyanime_app_keystore_key_alias=ifit_yuyanime
   yuyanime_app_keystore_key_password=<from 1Password>
   ```
   > Never commit the keystore or properties. `*.jks`, `yuyanime_keystore`, and
   > `local.properties` are gitignored — keep credentials out of the repo.
4. (Optional) confirm the keystore opens — expect a `PrivateKeyEntry` with an O=iFIT cert:
   ```sh
   keytool -list -keystore ~/.gradle/yuyanime_keystore -alias ifit_yuyanime
   ```

## 3. Build the offline release APK

```sh
./gradlew :app:assembleOfflineRelease
```

> **Gotcha:** `build.gradle` lists the Aliyun maven mirror first. It throws intermittent
> `502 Bad Gateway` errors that disable the repo for the rest of the build. If you hit a
> dependency-resolution failure mentioning `maven.aliyun.com`, just re-run — it's transient.

Output: `app/build/outputs/apk/offline/release/yuyanIme_<versionCode>_offline_release.apk`

## 4. Verify (sanity check)

```sh
# signature — expect v1 + v2 = true
$ANDROID_HOME/build-tools/<latest>/apksigner verify --verbose <apk>
# package + version
$ANDROID_HOME/build-tools/<latest>/aapt2 dump badging <apk> | grep -E "^package:"
```

## 5. Rename for distribution

Admin/CDN require the filename format `com.xx.xxx-versionname.versioncode.apk`:

```sh
cp app/build/outputs/apk/offline/release/yuyanIme_<versionCode>_offline_release.apk \
   com.yuyan.pinyin.offline.release-<versionName>.<versionCode>.apk
# e.g. com.yuyan.pinyin.offline.release-20260701.05.2026070105.apk
```

## 6. Distribute — Part A: Admin Portal

Per the WOLF doc
"[How To Update 3rd Party Apps](https://ifitdev.atlassian.net/wiki/spaces/WOLF/pages/3664511006)":

1. Log into the Admin Portal → `Wolf Updates` → `App Updates`.
2. Click `Create` (top right).
3. On the `Upload File` tab, upload the renamed APK (filename must be
   `com.yuyan.pinyin.offline.release-<versionName>.<versionCode>.apk`).
4. On the `General` tab, fill in the app info — the **FQN** and **versionCode** must match
   the uploaded APK — then `Save`.

## 7. Distribute — Part B: China S3 / CDN

The APK must also be uploaded to the China CDN, served at
`https://ifit-wolf.svc.ifit.cn/android/builds/public/`. Steps 1–7 follow the VC1 doc
"[Pulling Customer Logs from China S3](https://ifitdev.atlassian.net/wiki/spaces/VC1/pages/3824386272)":

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

## Credits

Built on [`gurecn/YuyanIme`](https://github.com/gurecn/YuyanIme) and the
[RIME engine](http://rime.im).
