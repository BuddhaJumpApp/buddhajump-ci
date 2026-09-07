# CODEMAP — buddhajump-ci

> File-by-file map of the BuddhaJump CI repo: six GitHub Actions workflows (one orchestrator, three reusable per-platform builders, one Android-only hotfix, one daily MSIX mirror), plus LICENSE and a leftover .gitignore. Every claim cites `file:line`.

写于 2026-09-07。仓库全部文件如下；每个条目给出角色、触发、输入、关键步骤（行号）、输出、用到的 secrets、与外部的联系。总览见 [README.md](README.md)，全估算仓库地图见 hydra 的 [docs/REPOS.md](https://github.com/fjolskylduoryggisverndar/hydra/blob/HEAD/docs/REPOS.md)。

```
.
├── .github/workflows/
│   ├── build.yml                  91 行  编排：读版本 → 并行调三个平台 workflow
│   ├── build-android.yml         148 行  AAB+APK；AAB→Play internal；APK→Release(latest)
│   ├── build-apple.yml           345 行  iOS ipa + macOS zip/pkg(+公证 dmg)；可选上传 ASC
│   ├── build-windows.yml         213 行  便携 zip(+VC++ DLL)；可选 MSIX→Microsoft Store
│   ├── build-android-hotfix.yml  120 行  只出 APK 的非 latest prerelease
│   └── publish.yml                36 行  每日 cron：把 Store 上的 MSIX 镜像回 Release
├── .gitignore                      Rust 模板残留，与本仓库无关
└── LICENSE                         GNU GPL v3 全文（35 KB）
```

没有 README 以外的文档、没有脚本、没有源码。所有构建输入都来自 `PRIVATE_REPO` 指向的私有源码仓库。

---

## `.github/workflows/build.yml`

- **角色**：唯一的人工入口；解析版本并扇出到三个可复用 workflow。
- **触发**（`:10-34`）：`repository_dispatch: [pubspec-updated]`（无发送方，实际死路，见 README §2）；`workflow_dispatch`，4 个布尔输入 `publish_to_stores` / `publish_ios_testflight` / `publish_macos_testflight` / `build_windows_store_package`，默认全 false。
- **全局**：`env.GITHUB_TOKEN`、`env.PRIVATE_REPO`（`:3-5`）；`permissions: contents: write`（`:7-8`）。
- **job `check-for-version`**（ubuntu-latest，`:37-59`）
  - `actions/checkout@v7` 检出 `${{ env.PRIVATE_REPO }}`，token `secrets.ORG_TOKEN`（`:44-48`）。
  - step `get-info`（`:50-59`）：`NAME` ← pubspec `^name:`；`PKG` ← `android/app/build.gradle.kts` 的 `applicationId = "..."`；`VERSION` ← pubspec `^version:` 整行加 `v` 前缀（**保留 `+N`**）。
  - outputs：`name` / `version` / `pkg`（`:39-42`）。
- **job `build-windows` / `build-android` / `build-apple`**（`:61-91`）：`uses: ./.github/workflows/build-*.yml`，`secrets: inherit`；布尔输入一律包成 `github.event_name == 'workflow_dispatch' && inputs.X || false`（`:68-69, 88-90`），非手动触发时等价于全关。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`。
- **外部联系**：私有仓库 `BuddhaJumpApp/buddhajump`（读 `pubspec.yaml`、`android/app/build.gradle.kts`）。

---

## `.github/workflows/build-android.yml`

- **角色**：Android 构建 + Play 上传 + APK 发布，并**决定 GitHub Release 的 latest 指针**。
- **触发**（`:10-24`）：`workflow_call`，输入 `name` / `pkg` / `version`（string，必填）。
- **job `build-android-app`**（ubuntu-latest，`:27-148`）
  1. checkout 私有仓库（`:30-34`）。
  2. `actions/setup-java@v5` temurin 17（`:36-40`）；`subosito/flutter-action@v2` `channel: stable`, `cache: true`（`:42-46`）；`actions/cache@v5` pub 缓存，key 含 `pubspec.lock` hash（`:48-56`）。
  3. step `build-env`（`:58-68`）：去 `v`、去 `+N`，`BUILD_NUMBER=a*100+b*10+c`，`sed -i` 回写 pubspec `version: X.Y.Z+BUILD_NUMBER`；output `build_number`（后续无人消费）。
  4. `flutter pub get`、`flutter --version`（`:70-76`）。
  5. 签名（`:78-86`）：`PKCS12_BASE64` → `android/app/<PKCS12_NAME>-keystore.jks`；heredoc 写 `android/key.properties`（storePassword/keyPassword = `PKCS12_PASSWORD`，keyAlias = `PKCS12_NAME`，storeFile = `<PKCS12_NAME>-keystore.jks`）。
  6. 构建（`:88-110`）：`rm -rf .dart_tool android/.gradle`；`flutter gen-l10n`；`flutter pub run flutter_launcher_icons`；`flutter build appbundle --release --obfuscate --split-debug-info=debug-info/android --suppress-analytics --no-tree-shake-icons` → `<name>-android-<version>.aab`；`flutter build apk`（同参数）→ `<name>-android-universal-<version>.apk`。
  7. 解码 `GOOGLE_SERVICE_ACCOUNT_BASE64` → `service-account.json`（`:112-113`）。
  8. step `publish`：`r0adkll/upload-google-play@v1`，`packageName: inputs.pkg`，`track: internal`，`status: completed`，`continue-on-error: true`（`:115-128`）。**无 `if:`，每次都跑**。注释 `:123-126` 解释为何不是 production。
  9. `softprops/action-gh-release@v3` 传 APK：`tag_name: inputs.version`，`name: Release <version>`，`make_latest: true`（`:130-138`）。
  10. `if: steps.publish.outcome == 'failure'` 时再传 AAB（`:140-148`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`PKCS12_BASE64`、`PKCS12_NAME`、`PKCS12_PASSWORD`、`GOOGLE_SERVICE_ACCOUNT_BASE64`。
- **外部联系**：Google Play（internal 轨道，包名 = `inputs.pkg`）；本仓库 Release；dl.* Worker 通过 `/releases/latest` 间接依赖第 9 步。

---

## `.github/workflows/build-apple.yml`

- **角色**：iOS + macOS 构建、可选 ASC/TestFlight 上传、macOS Developer ID 公证 DMG（**本仓库独有**，10 个 fork 无）。
- **触发**（`:10-39`）：`workflow_call`，输入 `name` / `pkg` / `version` + 布尔 `publish_to_stores` / `publish_ios_testflight` / `publish_macos_testflight`。`:39` 的描述明确 macOS TestFlight 需要 `APPLE_INSTALLER_PKCS12_BASE64/_PASSWORD`。
- **job `build-apple-app`**（macos-latest，`strategy.matrix.platform: [ios, macos]`，`fail-fast: false`，`:42-345`）
  1. checkout 私有仓库（`:49-53`）；Flutter stable（`:55-59`）；`maxim-lobanov/setup-xcode@v1` `latest-stable`（`:61-64`）；pub 缓存（`:66-74`）。
  2. build env（`:76-94`）：`BUILD_VERSION` 进 env；同 Android 的 `a*100+b*10+c` 推导，BSD `sed -i ''` 回写 pubspec。注释 `:80-86` 记录 1.2.8 曾 Android 128 / iOS 30 的不一致。
  3. `flutter pub get`、`flutter --version`（`:95-101`）。
  4. keychain（`:103-125`）：临时 keychain，随机密码，导入 `APPLE_PKCS12_BASE64`（密码 `APPLE_PKCS12_PASSWORD`）；若 `APPLE_INSTALLER_PKCS12_BASE64` 非空再导入 installer 证书（`:119-123`，注释注明到期 2027-07-29）；`set-key-partition-list`；`list-keychain`。
  5. 描述文件（`:127-131`）：`APPLE_PROVISIONING_BASE64` → `base64 -d | tar -xJf -` 到 `~/Library/MobileDevice/Provisioning Profiles`。
  6. 构建（`:133-168`）：`rm -rf .dart_tool build/<platform> <platform>/.Pods`；`flutter gen-l10n`；`flutter_launcher_icons`；
     - ios：`flutter build ipa --release --obfuscate --split-debug-info=debug-info/ios --export-options-plist=ios/ExportOptions.plist` → `build/ios/ipa/<name>-<version>.ipa`，写 `UPLOAD_FILE`（`:141-149`）。
     - macos：`flutter build macos --release ...`；若 `publish_to_stores || publish_macos_testflight`：`security find-identity` 找 `3rd Party Mac Developer Installer`，`test -n` 失败即中止，`productbuild --sign` → `build/macos/<name>-<version>.pkg`；否则 `ditto -c -k --keepParent` → `build/macos/<name>-macos-<version>.zip`（`:150-167`）。
  7. **Build notarized DMG**（`if: matrix.platform == 'macos'`，`:188-294`；设计说明在 `:170-187`）：
     - `APPLE_DEVELOPER_ID_PKCS12_BASE64` 为空 → `exit 0`（`:195-198`；注释 `:192-194` 解释为何不能放在 `if:` 里）。
     - 导入 Developer ID 证书到同一 keychain，`find-identity -p codesigning | grep "Developer ID Application"`（`:200-208`）。
     - `APPLE_DEVID_PROVISIONING_BASE64` 解到 `$RUNNER_TEMP/devid-profiles`（`:213-215`）。
     - 复制 `.app` 到 `$RUNNER_TEMP/dmgwork`（`:217-225`）；找 `*.appex`；**写死** `BuddhaJump_DevID_macOS.provisionprofile` / `BuddhaJump_Service_DevID_macOS.provisionprofile`（`:227-231`）。
     - 由内向外 `codesign --force --timestamp --options runtime`：dylib/framework → appex（entitlements `macos/ServiceExtension/ServiceExtension.entitlements`）→ app（`macos/Runner/Release.entitlements`）；`codesign --verify --deep --strict`（`:233-245`）。
     - `hdiutil create -format UDZO` → `build/macos/<name>-macos-<version>.dmg`，签 DMG（`:247-255`）。
     - 写 ASC key 到 `~/private_keys/AuthKey_<APPLE_API_KEY_ID>.p8`；`xcrun notarytool submit --wait --timeout 90m`（`:259-271`，注释 `:262-265` 记录 30m 被 2026-09-01 的队列拖死）；成功则 `stapler staple`；`rm -rf ~/private_keys`（`:272-280`）。
     - 两个 rc 都为 0 才 `spctl -a -t install` 验证并写 `DMG_PATH`，否则 `::warning::` 且**不失败**（`:287-294`）。
  8. Upload DMG（`if: matrix.platform == 'macos' && env.DMG_PATH != ''`，`overwrite_files: true`，`:296-305`）。
  9. Clean up keychain（`if: always()`，删 keychain 和 profile 目录，`:307-311`）。
  10. Setup ASC API key（条件 `publish_to_stores || (ios_tf && ios) || (macos_tf && macos)`，`:313-318`）。
  11. step `publish`：`xcrun altool --upload-app --type ios|osx --file $UPLOAD_FILE --apiKey --apiIssuer`，`continue-on-error`（`:320-330`）。
  12. Upload to GitHub release（`if: !inputs.publish_to_stores || steps.publish.outcome == 'failure'`，`overwrite_files: true`，`files: env.UPLOAD_FILE`，`:332-341`）。
  13. Clean up API key（`if: always() && <同 10 的条件>`，`:343-345`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`APPLE_PKCS12_BASE64`、`APPLE_PKCS12_PASSWORD`、`APPLE_INSTALLER_PKCS12_BASE64`、`APPLE_INSTALLER_PKCS12_PASSWORD`、`APPLE_PROVISIONING_BASE64`、`APPLE_DEVELOPER_ID_PKCS12_BASE64`、`APPLE_DEVELOPER_ID_PKCS12_PASSWORD`、`APPLE_DEVID_PROVISIONING_BASE64`、`APPLE_API_KEY_BASE64`、`APPLE_API_KEY_ID`、`APPLE_API_ISSUER_ID`。
- **外部联系**：App Store Connect / TestFlight（altool）；Apple notary service（notarytool）；本仓库 Release；私有仓库里的 `ios/ExportOptions.plist`（`method: app-store`, `signingStyle: manual`, teamID + 两个 profile 名）、`macos/Runner/Release.entitlements`、`macos/ServiceExtension/ServiceExtension.entitlements`。

---

## `.github/workflows/build-windows.yml`

- **角色**：Windows 便携 zip（含 VC++ 运行库）+ 可选 Microsoft Store MSIX。
- **触发**（`:11-35`）：`workflow_call`，输入 `name` / `pkg` / `version` + 布尔 `build_store_package` / `publish_to_stores`。
- **全局 env**：`PARTNER_CENTER_API: https://manage.devcenter.microsoft.com/v1.0/my/applications`（`:5`，后续步骤未引用）。
- **job `build-windows-app`**（windows-latest，`outputs.submission-success: steps.publish.outcome == 'success'`（`:40-41`，无人消费），`:38-213`）
  1. checkout 私有仓库（`:43-47`）。
  2. Store 前置（条件 `build_store_package || publish_to_stores`）：`microsoft/microsoft-store-apppublisher@v1.3`（`:49-51`）；`msstore reconfigure --tenantId PARTNER_TENANT --sellerId PARTNER_SELLER --clientId PARTNER_CLIENT --clientSecret PARTNER_SECRET`（`:52-54`）；`Install-Module StoreBroker`（`:56-59`，后续未用）。
  3. `rm -rf build windows/.Dart_tool .dart_tool`（`:61-63`）。
  4. Flutter stable（`:65-69`）；pub 缓存（Windows 路径，`:71-79`）；`flutter config --enable-windows-desktop --enable-native-assets`（`:81-85`）；`flutter pub get`（`:88-92`）。
  5. `flutter build windows --release --obfuscate --split-debug-info=debug-info/windows-amd64 ...`，env `CL=-D_SILENCE_EXPERIMENTAL_COROUTINE_DEPRECATION_WARNINGS`（`:96-107`）。**没有 gen-l10n、没有 launcher_icons。**
  6. Bundle VC++ runtime（pwsh，`:109-139`）：`msvcp140.dll`、`vcruntime140.dll`、`vcruntime140_1.dll` 从 `vswhere` 找到的 `VC\Redist\MSVC\*\x64\Microsoft.VC14*.CRT`（回退 System32）拷到 `build/windows/x64/runner/Release`，缺一个就 `throw`。
  7. `Compress-Archive` → `<name>-windows-amd64-<version>.zip`，写 `WINDOWS_ZIP`（`:141-146`）；上传 Release，`overwrite_files: true`（`:148-156`）。
  8. Store 路径（条件同 2）：`MSIX_VERSION=a.b.c.0`、`MSIX_FILENAME=<name>-windows-amd64-<version>-store.msix`（`:158-170`）；`dart run msix:create -e "internetClient, privateNetworkClientServer" -d <PARTNER_APP 最后一段> -l assets/logo.png -u PARTNER_COMPANY -i PARTNER_APP --version MSIX_VERSION --publisher PARTNER_CN --store true --trim-logo false --sign-msix false`（`:172-189`）；移动到 `MSIX_FILENAME`（`:191-196`）；step `publish`：`msstore publish '<MSIX_FILENAME>' -id PARTNER_STORE`，`continue-on-error`（`:198-202`）；发布失败则 MSIX 上传 Release（`:204-213`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`PARTNER_TENANT`、`PARTNER_SELLER`、`PARTNER_CLIENT`、`PARTNER_SECRET`、`PARTNER_APP`、`PARTNER_COMPANY`、`PARTNER_CN`、`PARTNER_STORE`。
- **外部联系**：Microsoft Partner Center / Store；本仓库 Release；私有仓库的 `assets/logo.png` 和 `msix` 依赖（`pubspec.yaml` `msix: ^3.16.13`）。

---

## `.github/workflows/build-android-hotfix.yml`

- **角色**：Android 专用侧车，只出 APK，不进 Play、不碰 latest。头注释 `:1-18` 是完整设计说明。
- **触发**（`:27-33`）：仅 `workflow_dispatch`，必填 string `label`（例 `1.6.9a`）。
- **job `build-android-hotfix`**（ubuntu-latest，`:36-120`）
  1. checkout / Java 17 / Flutter stable / pub 缓存（`:39-65`，与 build-android.yml 相同）。
  2. step `info`（`:67-75`）：`name`、`version` 直接取 pubspec，**不重算 build number**。
  3. `flutter pub get`、`flutter --version`（`:77-81`）。
  4. 签名（`:83-91`，与 build-android.yml `:78-86` 相同）。
  5. `flutter gen-l10n`、`flutter_launcher_icons`、`flutter build apk --release --obfuscate ...` → `<name>-android-universal-v<label>.apk`（`:93-107`）。
  6. `softprops/action-gh-release@v3`：`tag_name: v<label>`，`name: Release v<label> (Android hotfix)`，body 说明，`prerelease: true`，`make_latest: false`（`:109-120`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`PKCS12_BASE64`、`PKCS12_NAME`、`PKCS12_PASSWORD`。
- **外部联系**：本仓库 Release（prerelease）。注释 `:8-12` 点明它刻意避开 `dl.<domain>/android|windows` 的 latest 解析。

---

## `.github/workflows/publish.yml`

- **角色**：把 Microsoft Store 上已上架的 MSIX 镜像回本仓库 Release。
- **触发**（`:3-11`）：`schedule: '0 0 * * *'`；`workflow_dispatch`；`push` 到 `master` 且路径为 `.github/workflows/publish.yml`。
- **job `upload-to-release`**（ubuntu-24.04，`:21-36`）
  1. checkout 私有仓库（`:24-28`）——后续步骤未使用检出内容。
  2. `JasonWei512/Upload-Microsoft-Store-MSIX-Package-to-GitHub-Release@v1`，`store-id: PARTNER_STORE`，`token: GITHUB_TOKEN`，`asset-name-pattern: <PKCS12_NAME>_{version}_{arch}`，`continue-on-error: true`（`:30-36`）。
- **secrets**：`GITHUB_TOKEN`、`PRIVATE_REPO`、`ORG_TOKEN`、`PARTNER_STORE`、`PKCS12_NAME`。
- **外部联系**：Microsoft Store 公开包接口；本仓库 Release。每天消耗 1 分钟 ubuntu 时长。

---

## `.gitignore`

内容：`.DS_Store`、`.claude`、`CLAUDE.md`、`debug/`、`target/`、`Cargo.lock`、`**/*.rs.bk`、`*.pdb`。除前三项外都是 Rust 项目模板残留；本仓库没有任何 Rust 内容。注意它忽略了 `CLAUDE.md`——本仓库不放 CLAUDE.md 是这个原因。

## `LICENSE`

GNU GPL v3 全文（`LICENSE:1-2`：`GNU GENERAL PUBLIC LICENSE / Version 3, 29 June 2007`）。只覆盖本仓库的 workflow 文件；应用源码在私有仓库，不受此文件约束。

---

## 跨文件的共同模式

| 模式 | 出处 |
|---|---|
| 每个 workflow 顶部 `env.GITHUB_TOKEN` + `env.PRIVATE_REPO`，`permissions: contents: write` | 6 个文件开头 |
| 第一步永远是 `actions/checkout@v7` + `repository: env.PRIVATE_REPO` + `token: secrets.ORG_TOKEN` | 每个 job 第一步 |
| Release 全部通过 `softprops/action-gh-release@v3`，`tag_name` = `inputs.version`（hotfix 为 `v<label>`），`name: Release <tag>` | 7 处 |
| 商店上传步骤全部 `continue-on-error: true` + `id: publish`，失败时把产物改传 Release | build-android `:115-148`、build-apple `:320-341`、build-windows `:198-213` |
| build number `a*100+b*10+c` 回写 pubspec | build-android `:58-68`、build-apple `:76-94`（hotfix 例外） |
| `flutter gen-l10n` + `flutter pub run flutter_launcher_icons` 在构建前现跑 | build-android `:93-94`、build-apple `:138-139`、hotfix `:98-99`；**build-windows 没有** |
| 构建参数 `--release --obfuscate --split-debug-info=debug-info/<platform> --suppress-analytics --no-tree-shake-icons` | 四个构建命令 |
| Flutter `channel: stable` 未锁版本；Xcode `latest-stable` | 所有构建 job |

## 第三方 action 一览

| action | 版本 | 用处 |
|---|---|---|
| `actions/checkout` | v7 | 检出私有仓库 |
| `actions/setup-java` | v5 | Temurin 17（Android） |
| `subosito/flutter-action` | v2 | Flutter stable |
| `actions/cache` | v5 | pub 缓存 |
| `maxim-lobanov/setup-xcode` | v1 | Xcode latest-stable |
| `r0adkll/upload-google-play` | v1 | AAB → Play internal |
| `softprops/action-gh-release` | v3 | 所有 Release 上传 |
| `microsoft/microsoft-store-apppublisher` | v1.3 | `msstore` CLI |
| `JasonWei512/Upload-Microsoft-Store-MSIX-Package-to-GitHub-Release` | v1 | publish.yml |
