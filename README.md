# buddhajump-ci

> GitHub Actions pipeline for the BuddhaJump Flutter VPN client: checks out the private app repo, builds Android/iOS/macOS/Windows, publishes the download assets as GitHub Releases **on this repo**, and optionally pushes to Play internal / App Store Connect / TestFlight / Microsoft Store.

本文件写于 2026-09-07，每一条事实都标注了在 `.github/workflows/*.yml` 里的出处（`文件:行号`）。
没有标注出处、来自运维记忆的内容，全部集中在文末「来自运维记忆（未在代码中复核）」一节，读的时候请区别对待。
逐文件的细节见 [CODEMAP.md](CODEMAP.md)。全部 42 个仓库的地图见 hydra 仓库的 [docs/REPOS.md](https://github.com/fjolskylduoryggisverndar/hydra/blob/HEAD/docs/REPOS.md)。

---

## 1. 这个仓库是什么、不是什么

| | |
|---|---|
| **是** | BuddhaJump 客户端的构建流水线。仓库里只有 6 个 workflow 文件、LICENSE 和一个 `.gitignore`，**没有任何应用源码**。 |
| **是** | 客户端下载产物的发布地：每次构建用 `softprops/action-gh-release` 在**本仓库**建 Release（`build-android.yml:130-138`、`build-windows.yml:148-156`、`build-apple.yml:332-341`）。官网的下载按钮最终指向这里的 Release 资产（见 §7）。 |
| **是** | 其余 10 个品牌 `<brand>-ci` 仓库的**母版**（见 §10）。 |
| **不是** | 源码仓库。源码在私有仓库 `BuddhaJumpApp/buddhajump`，由 workflow 通过 `PRIVATE_REPO` + `ORG_TOKEN` 检出（`build.yml:44-48`）。改代码、改版本号都要去那边。 |
| **不是** | 自动化的。虽然 `on:` 里写了 `repository_dispatch`，但全估算没有发送方（见 §2），实际每一次发版都是人手点 `workflow_dispatch`。 |

三件套：`BuddhaJumpApp/buddhajump`（源码，私有）→ **`BuddhaJumpApp/buddhajump-ci`（本仓库，公开）** → `BuddhaJumpApp/buddhajump-web`（官网快照）。
本仓库的 `.gitignore` 是 Rust 项目模板残留（`target/`、`Cargo.lock`、`*.rs.bk`），与本仓库内容无关，可忽略。

---

## 2. 触发方式：写的和真的不一样

`build.yml:10-34` 声明了两个触发器：

```yaml
on:
  repository_dispatch:
    types: [pubspec-updated]
  workflow_dispatch:
    inputs: publish_to_stores / publish_ios_testflight / publish_macos_testflight / build_windows_store_package  # 4 个布尔，默认 false
```

**`repository_dispatch: pubspec-updated` 是死路。**

- 发送方必须向 `api.github.com/repos/BuddhaJumpApp/buddhajump-ci/dispatches` POST `event_type: pubspec-updated`。`BuddhaJumpApp/buddhajump` 源码仓库里**没有** `.github/workflows/` 目录（本地克隆核对，`brand-facts.json` 顶层目录清单亦无 `.github`），所以没有人发。
- 全估算里唯一会发这个事件的是 `00000vpn`/`88888vpn` 两个源码仓库的 `.github/workflows/ci.yml`（`ci.yml:58-63`，POST 到 `secrets.PUBLIC_REPO`），但它们的触发条件写的是 `push: branches: [master]`（`ci.yml:34-36`），而这两个仓库的默认分支是 `main` —— push 永远打不中，事件永远发不出。而且它们的目标是各自的 ci 仓库，与本仓库无关。
- 即便事件真的来了，`build.yml:68-69, 88-90` 把所有布尔输入包在 `github.event_name == 'workflow_dispatch' && inputs.xxx || false` 里，dispatch 触发的构建等价于「四个开关全关」。

**真正的入口是 `workflow_dispatch`，四个布尔输入的实际效果：**

| 输入 | 传给谁 | 实际行为 |
|---|---|---|
| `publish_to_stores` | build-windows、build-apple | Apple：iOS `.ipa` + macOS 签名 `.pkg` 都 `altool --upload-app` 到 App Store Connect（`build-apple.yml:320-330`），且**不再**把 ipa/pkg 传到 GitHub Release，除非上传失败（`build-apple.yml:333`）。Windows：构建 MSIX 并 `msstore publish`（`build-windows.yml:172-202`）。 |
| `publish_ios_testflight` | build-apple | 只把 iOS `.ipa` 上传 ASC（`build-apple.yml:314, 321`）。因为 `publish_to_stores` 为 false，ipa **同时**还会传到 GitHub Release（`build-apple.yml:333` 的条件是 `!inputs.publish_to_stores`）。 |
| `publish_macos_testflight` | build-apple | 用 `productbuild` 签一个 `.pkg`（需要 Mac Installer Distribution 证书，`build-apple.yml:157-162`）并上传 ASC。缺 `APPLE_INSTALLER_PKCS12_*` 时 `test -n "$INSTALLER_CERT"` 直接失败（`build-apple.yml:160`）。 |
| `build_windows_store_package` | build-windows | 名字说「只构建不发布」，但 `Publish to Store` 步骤的条件是 `inputs.build_store_package \|\| inputs.publish_to_stores`（`build-windows.yml:199`）——**它会尝试发布**，失败了（`continue-on-error`）才把 MSIX 传到 Release（`build-windows.yml:204-213`）。 |

**四个开关全关时（默认）依然会发生的事：**

- Android AAB **无条件**上传 Google Play `internal` 轨道（`build-android.yml:115-128`，没有任何 `if:`，只有 `continue-on-error`）。
- APK、Windows zip、iOS ipa、macOS zip 全部发到本仓库 GitHub Release，且 APK 那一步 `make_latest: true`（`build-android.yml:137`）—— 这一步决定了 dl.* 下载链路指向哪个版本（§7）。
- 如果配置了 Developer ID secrets，macOS 还会产出公证过的 `.dmg`（§4.3）。

---

## 3. 版本号、build number、tag、资产名

三个数字来自两处不同的逻辑，**不要混淆**：

| 名称 | 来源 | 例（pubspec `version: 1.7.3+178`） |
|---|---|---|
| Release tag / `inputs.version` | `build.yml:55-58`：把 pubspec 的 `version:` 整行原样加 `v` 前缀，**不去掉 `+N`** | `v1.7.3+178` |
| Android `versionCode` / Apple `CFBundleVersion` | `build-android.yml:62-67`、`build-apple.yml:88-93`：去掉 `v` 和 `+N` 后 `a*100 + b*10 + c`，再 `sed` 回写 pubspec | `173` |
| Windows MSIX 版本 | `build-windows.yml:163-167`：`a.b.c.0` | `1.7.3.0` |

也就是说 **tag 里的 `+178` 与安装包里的 build number 173 可以不一致**，tag 只是 pubspec 那一刻的文字。ASC / Play 看到的永远是算出来的那个（`build-apple.yml:80-86` 的注释记录了 1.2.8 曾在 Android 是 128、iOS 是 30 的事故，这套推导就是为此加的）。

资产命名（`name` = pubspec `name:`，BuddhaJump 是 `buddhajump`；`version` = 上面的 tag）：

| 平台 | 文件名 | 出处 |
|---|---|---|
| Android APK（universal） | `buddhajump-android-universal-<tag>.apk` | `build-android.yml:110` |
| Android AAB（仅 Play 上传失败时才进 Release） | `buddhajump-android-<tag>.aab` | `build-android.yml:102, 140-148` |
| Windows 便携 zip | `buddhajump-windows-amd64-<tag>.zip` | `build-windows.yml:144` |
| Windows Store MSIX（仅失败回落） | `buddhajump-windows-amd64-<tag>-store.msix` | `build-windows.yml:168` |
| iOS | `buddhajump-<tag>.ipa` | `build-apple.yml:148` |
| macOS 未签 pkg 时 | `buddhajump-macos-<tag>.zip` | `build-apple.yml:164` |
| macOS TestFlight/Store | `buddhajump-<tag>.pkg` | `build-apple.yml:158` |
| macOS 直发 | `buddhajump-macos-<tag>.dmg` | `build-apple.yml:252` |
| Android hotfix | `buddhajump-android-universal-v<label>.apk`，tag `v<label>` | `build-android-hotfix.yml:107, 112` |

Windows 打包用的 `name` 同样来自 pubspec；Release 名固定为 `Release <tag>`（各文件的 `name: Release ${{ inputs.version }}`）。

---

## 4. 一次 `build.yml` 运行会做什么

`build.yml` 只有一个真正的 job `check-for-version`（ubuntu-latest），检出私有仓库、从 `pubspec.yaml` 读 `name`/`version`、从 `android/app/build.gradle.kts` 读 `applicationId`（`build.yml:50-59`），然后**并行**调用三个可复用 workflow（`build.yml:61-91`，`secrets: inherit`）。

### 4.1 build-android.yml → `build-android-app`（ubuntu-latest）

1. 检出私有仓库（`:30-34`），Temurin JDK 17（`:36-40`），Flutter `stable` 频道（`:42-46`，**未锁定版本**），pub 缓存（`:48-56`）。
2. 推导 build number 并回写 pubspec（`:58-68`）。
3. 写入签名：`PKCS12_BASE64` 解码成 `android/app/<PKCS12_NAME>-keystore.jks`，`android/key.properties` 里 storePassword = keyPassword = `PKCS12_PASSWORD`，keyAlias = `PKCS12_NAME`（`:78-86`）。
4. `rm -rf .dart_tool android/.gradle` → `flutter gen-l10n` → `flutter pub run flutter_launcher_icons` → `flutter build appbundle` + `flutter build apk`，都带 `--obfuscate --split-debug-info`（`:88-110`）。
5. AAB → Google Play **internal** 轨道，`status: completed`，`continue-on-error`（`:112-128`）。注释（`:123-126`）解释了为什么不是 production：这些 App 从未过 Play 审核，production+completed 会瞬间对所有人发布。
6. APK → 本仓库 Release，`make_latest: true`（`:130-138`）。
7. 仅当第 5 步失败：AAB 也传 Release（`:140-148`）。

### 4.2 build-windows.yml → `build-windows-app`（windows-latest）

1. 检出私有仓库（`:43-47`）。仅在 Store 开关打开时装 `msstore` CLI 并 `reconfigure`（`:49-54`，用 `PARTNER_TENANT/SELLER/CLIENT/SECRET`），再装 StoreBroker（`:56-59`，后面没有用到）。
2. Flutter stable，开启 `windows-desktop` 和 `native-assets`（`:65-85`）。
3. `flutter build windows --release --obfuscate ...`（`:96-107`）。**这里没有 `flutter gen-l10n`、没有 `flutter_launcher_icons`**（对比 android/apple 都有）——Windows 构建直接使用源码仓库里已提交的生成物。
4. **把 VC++ 运行库三个 DLL（`msvcp140.dll`、`vcruntime140.dll`、`vcruntime140_1.dll`）拷到 exe 旁边**（`:109-139`）。注释写明：开发机全局有这些 DLL 所以本地永远测不出，只在用户机器上报 `VCRUNTIME140_1.dll was not found`；路径用 `vswhere` 找，硬编码年份曾经断过一次。
5. `Compress-Archive` 成便携 zip → Release（`:141-156`）。
6. 仅在 Store 开关打开时：算 MSIX 版本 `a.b.c.0`（`:158-170`），`dart run msix:create --store true --sign-msix false`（`:172-189`，显示名取 `PARTNER_APP` 最后一个点后的段，publisher 为 `PARTNER_CN`，identity 为 `PARTNER_APP`，发布者显示名 `PARTNER_COMPANY`），`msstore publish -id PARTNER_STORE`（`:198-202`，`continue-on-error`），失败则 MSIX 传 Release（`:204-213`）。

job 声明了 `outputs.submission-success`（`:40-41`），但 `build.yml` 没有消费它。

### 4.3 build-apple.yml → `build-apple-app`（macos-latest，matrix `ios`/`macos` 两个 job，`fail-fast: false`）

1. 检出私有仓库、Flutter stable、`setup-xcode latest-stable`（`:49-64`，Xcode 也未锁定）。
2. 推导 build number 并用 BSD `sed -i ''` 回写 pubspec（`:76-94`）。
3. 建临时 keychain，导入 `APPLE_PKCS12_*`（App 签名证书）；如果设置了 `APPLE_INSTALLER_PKCS12_*` 再导入 Mac Installer Distribution 证书（`:103-125`，注释注明该证书 2027-07-29 到期）。
4. `APPLE_PROVISIONING_BASE64` 是 **base64 编码的 tar.xz**，解到 `~/Library/MobileDevice/Provisioning Profiles`（`:127-131`）。
5. `flutter gen-l10n` + `flutter_launcher_icons` 后：iOS `flutter build ipa --export-options-plist=ios/ExportOptions.plist`；macOS `flutter build macos`，然后按开关决定 `productbuild` 签 `.pkg` 还是 `ditto` 打 `.zip`（`:133-168`）。
6. **Build notarized DMG（仅 macOS job，本仓库独有）**（`:170-305`）：
   - 若 `APPLE_DEVELOPER_ID_PKCS12_BASE64` 为空则直接跳过（`:195-198`）。
   - 导入 Developer ID Application 证书；解出 `APPLE_DEVID_PROVISIONING_BASE64` 里的 profile，**文件名写死为 `BuddhaJump_DevID_macOS.provisionprofile` 和 `BuddhaJump_Service_DevID_macOS.provisionprofile`**（`:228, 230`）——移植到别的品牌必须改。
   - 在 `.app` 的副本上由内向外重签（dylib/framework → `*.appex` → app），Hardened Runtime，entitlements 用 `macos/ServiceExtension/ServiceExtension.entitlements` 和 `macos/Runner/Release.entitlements`（`:233-245`）。
   - `hdiutil` 做拖拽安装 DMG，`notarytool submit --wait --timeout 90m`（`:266-271`；注释：30m 曾在 2026-09-01 被 Apple 队列拖死），成功才 `stapler staple`，**未公证成功就不发布 DMG 但也不让 job 失败**（`:287-294`）。
   - 上传 Release 的条件是 `env.DMG_PATH != ''`（`:296-305`）。
7. 清 keychain 和 profile（`:307-311`，`if: always()`）。
8. 按开关：写 ASC API key（`:313-318`），`xcrun altool --upload-app --type ios|osx`（`:320-330`，`continue-on-error`），然后按 §2 表里的条件传 Release（`:332-341`），最后删 key（`:343-345`）。

### 4.4 谁跑在哪、产物去哪（速查）

| job | runner | 产物 | 去向 |
|---|---|---|---|
| check-for-version | ubuntu-latest | `name` / `pkg` / `version` 三个 outputs | 下游三个 job |
| build-android-app | ubuntu-latest | AAB、APK | AAB → Play internal（总是）；APK → 本仓库 Release（latest）；AAB → Release（仅 Play 失败） |
| build-windows-app | windows-latest | zip；可选 MSIX | zip → Release；MSIX → Microsoft Store（开关）/ Release（发布失败） |
| build-apple-app [ios] | macos-latest | ipa | ASC/TestFlight（开关）；Release（非 `publish_to_stores` 或上传失败） |
| build-apple-app [macos] | macos-latest | zip 或 pkg；可选 dmg | 同上；dmg → Release（仅公证成功） |
| upload-to-release（publish.yml） | ubuntu-24.04 | 从 Microsoft Store 拉回的 MSIX | Release |
| build-android-hotfix | ubuntu-latest | APK | 本仓库 **prerelease、非 latest** 的 Release |

---

## 5. 实际用到的 secrets（26 个，全部 `grep secrets\.` 得出）

旧版 fork README 写的 `ANDROID_KEYSTORE_BASE64 / ANDROID_KEYSTORE_PASSWORD / ANDROID_KEY_PASSWORD / ANDROID_KEY_ALIAS` **在任何 yml 里都不存在**，不要照着配。

| 组 | secret | 用途 | 出处 |
|---|---|---|---|
| 检出 | `PRIVATE_REPO` | 源码仓库 `owner/name`，本仓库应为 `BuddhaJumpApp/buddhajump` | 每个 workflow 的 `env:` + `actions/checkout with.repository` |
| 检出 | `ORG_TOKEN` | 能读该私有仓库的 token，给 `actions/checkout with.token` | `build.yml:48` 等 6 处 |
| 检出 | `GITHUB_TOKEN` | Actions 自带，`permissions: contents: write`，供 `action-gh-release` 在本仓库建 Release | 每个 workflow 的 `env:` |
| Android | `PKCS12_BASE64` | keystore（jks）base64 | `build-android.yml:80` |
| Android | `PKCS12_NAME` | key alias，也是 keystore 文件名前缀；**publish.yml 还拿它当 MSIX 资产名前缀** | `build-android.yml:80-85`、`publish.yml:36` |
| Android | `PKCS12_PASSWORD` | store 密码 = key 密码 | `build-android.yml:82-83` |
| Android | `GOOGLE_SERVICE_ACCOUNT_BASE64` | Play 服务账号 JSON | `build-android.yml:113` |
| Apple | `APPLE_PKCS12_BASE64` / `APPLE_PKCS12_PASSWORD` | App 签名证书（Apple Distribution） | `build-apple.yml:111-112` |
| Apple | `APPLE_PROVISIONING_BASE64` | App Store 描述文件，tar.xz 再 base64 | `build-apple.yml:131` |
| Apple | `APPLE_INSTALLER_PKCS12_BASE64` / `_PASSWORD` | Mac Installer Distribution 证书（可选；macOS TestFlight/Store 必需） | `build-apple.yml:119-123` |
| Apple | `APPLE_API_KEY_BASE64` / `APPLE_API_KEY_ID` / `APPLE_API_ISSUER_ID` | ASC API key：altool 上传 + notarytool 公证 | `build-apple.yml:260-270, 317-330` |
| Apple（本仓库独有） | `APPLE_DEVELOPER_ID_PKCS12_BASE64` / `_PASSWORD` | Developer ID Application 证书 | `build-apple.yml:195-205` |
| Apple（本仓库独有） | `APPLE_DEVID_PROVISIONING_BASE64` | Developer ID 描述文件 tar.xz | `build-apple.yml:215` |
| Microsoft | `PARTNER_TENANT` / `PARTNER_SELLER` / `PARTNER_CLIENT` / `PARTNER_SECRET` | `msstore reconfigure` | `build-windows.yml:54` |
| Microsoft | `PARTNER_APP` / `PARTNER_COMPANY` / `PARTNER_CN` | MSIX identity / 发布者显示名 / publisher CN | `build-windows.yml:177-186` |
| Microsoft | `PARTNER_STORE` | Store 应用 ID | `build-windows.yml:202`、`publish.yml:34` |

`PRIVATE_REPO + ORG_TOKEN` 机制：本仓库公开、源码仓库私有，所有 job 第一步都是用 `ORG_TOKEN` 去检出 `PRIVATE_REPO`。改了源码仓库的归属或名字，就要同步改 `PRIVATE_REPO`（GitHub 对转移后的旧名有重定向，但不要依赖它）。运维记忆说 `ORG_TOKEN` 是 classic PAT——代码只能看出它是个能读私有仓库的 token。

---

## 6. 怎么手动跑一次

```bash
# 普通发版：四平台构建，产物全部进本仓库 Release，AAB 自动进 Play internal
gh workflow run build.yml --repo BuddhaJumpApp/buddhajump-ci --ref master

# 只想把 iOS 推到 TestFlight（ipa 同时也会进 Release）
gh workflow run build.yml --repo BuddhaJumpApp/buddhajump-ci --ref master \
  -f publish_ios_testflight=true

# 正式提交商店（iOS + macOS pkg 上传 ASC；Windows 构建 MSIX 并 msstore publish）
gh workflow run build.yml --repo BuddhaJumpApp/buddhajump-ci --ref master \
  -f publish_to_stores=true

# Android 专用 hotfix（只出 APK，不进 Play，不动 latest）
gh workflow run build-android-hotfix.yml --repo BuddhaJumpApp/buddhajump-ci --ref master \
  -f label=1.6.9a

# 看进度
gh run list --repo BuddhaJumpApp/buddhajump-ci --workflow build.yml --limit 5
gh run watch --repo BuddhaJumpApp/buddhajump-ci <run-id>
```

`--repo` 必须是 **ci 仓库**而不是源码仓库；`--ref master` 是本仓库的默认分支（本地克隆 `git branch --show-current` = master）。
运行前先确认源码仓库 `pubspec.yaml` 的 `version:` 已经改成新版本，否则会往**同一个 tag** 再传一遍（`overwrite_files: true` 会覆盖 Windows/Apple 资产，Android 那步没有 `overwrite_files`）。

---

## 7. 下载链路对本仓库的依赖

官网 → `dl.buddhajump.xyz/android|windows` → Cloudflare Worker `fjolsky-downloads` → 302 → `edgeone.gh-proxy.org/https://github.com/BuddhaJumpApp/buddhajump-ci/releases/download/<tag>/<asset>`。

Worker 源码（`XTPU/cloudflare/workers/app-downloads.js`，本地副本核对）：

- 表里 BuddhaJump 一行：`repo: "BuddhaJumpApp/buddhajump-ci", prefix: "buddhajump", pin: "v1.2.8+30"`（`app-downloads.js:23-26`）。
- **请求时**对 `https://github.com/<repo>/releases/latest` 发 `redirect: "manual"` 的 fetch，从 `Location` 头解析 tag，边缘缓存 600 秒（`:81-111`）。
- 任何失败（仓库改名、转私有、超时）都**静默回落到 `pin`**——用户会拿到一个很旧但仍存在的版本，而不是报错（`:105-110, 137, 156`）。
- 资产名由 `prefix` 和 tag 拼出（`:76-79`），必须与 §3 的命名一致。

由此推出的规则：

1. 改本仓库名字/归属/可见性之前，先改 Worker 的 `TABLE` 并重新部署，否则下载会悄悄退回 `v1.2.8+30`。
2. `latest` 指针由 `build-android.yml:137` 的 `make_latest: true` 决定；hotfix workflow 刻意 `make_latest: false` + `prerelease: true`（`build-android-hotfix.yml:118-119`）就是为了不碰它。
3. Worker 缓存 600 秒，发版后 10 分钟内 `dl.*` 还可能给旧版。
4. `build-android-hotfix.yml:8-12` 的头注释就是对这条链路的描述。

---

## 8. publish.yml：每天 00:00 UTC 的 cron

`publish.yml:3-11`：`schedule: '0 0 * * *'` + `workflow_dispatch` + push 到 `master` 且只改了 `publish.yml` 自己时触发。

唯一 job `upload-to-release`（ubuntu-24.04）：

1. 检出私有仓库（`:24-28`）——后面的步骤并没有用到检出内容。
2. `JasonWei512/Upload-Microsoft-Store-MSIX-Package-to-GitHub-Release@v1`（`:30-36`）：按 `PARTNER_STORE` 从 Microsoft Store 拉回**已上架**的 MSIX，以 `<PKCS12_NAME>_{version}_{arch}` 命名传到本仓库 Release；`continue-on-error: true`。

为什么 `continue-on-error`：这个 action 只有在 Store 上真的有已发布包时才成功；目前 Windows 走的是便携 zip（§4.2），Store 那条路不常开，所以这一步几乎每天都失败，`continue-on-error` 让运行记录保持绿色而不是天天报红。代价是它每天消耗 1 分钟 ubuntu 时长（2026-09-07 用 `gh api` 统计 30 天：11 个 ci 仓库各 30 次 × 1 分钟）。它也是每个 fork 里唯一"自动"运行的东西。

---

## 9. build-android-hotfix.yml：Android 专用侧车

`build-android-hotfix.yml:1-18` 的头注释把设计意图写全了，要点：

- 只有 `workflow_dispatch`，一个必填字符串输入 `label`（例 `1.6.9a`），**不要求是合法 Dart 版本**（`:27-33`）。
- 版本和 build number **直接用 pubspec 的**，不重算（`:67-75`），所以 `1.6.9+170` 会得到 versionCode 170、能覆盖安装 `1.6.9+169`。
- 只构建 APK（`:93-107`），不构建 AAB、不传 Play、不跑其他平台。
- Release：tag `v<label>`、`prerelease: true`、`make_latest: false`（`:109-120`），`dl.*` 的 latest 不受影响；测试员要用资产直链。
- 签名、gen-l10n、launcher_icons 步骤与 `build-android.yml` 一致（`:83-99`）。

---

## 10. 与 10 个 fork 的漂移（`diff` 结果，2026-09-07）

对 `kamevpn-ci / 00000vpn-ci / 88888vpn-ci / goddessvpn-ci / openbridge-ci / libertygate-ci / maskaura-ci / aiglefree-ci / maschvpn-ci / ninjashield-ci` 逐文件 diff：

| 文件 | 与本仓库的差异 |
|---|---|
| `build.yml` | 0 行 |
| `build-android.yml` | 0 行；`kamevpn-ci`/`88888vpn-ci` 只多第 1 行注释 `# CI build workflows for <brand>, copied from ninjashield-ci.` |
| `build-windows.yml` | 同上 |
| `build-apple.yml` | 全部 10 个 fork 都**缺** `170-306` 行的「Build notarized DMG」+「Upload DMG to GitHub release」两段（138 行 diff），其余逐字节相同 |
| `publish.yml` | 0 行 |
| `build-android-hotfix.yml` | 10 个 fork **都没有** |

10 个 fork 彼此之间除那一行头注释外完全相同。三个 fork（kamevpn-ci、00000vpn-ci、88888vpn-ci）默认分支是 `main`，但 `publish.yml:8-9` 的 push 触发写的是 `master`，在那三个仓库里只剩 cron 和手动两种触发。

推论：workflow 层的修复只要在本仓库改完，再把同名文件整份拷到 10 个 fork 即可（DMG 段除外，它写死了 BuddhaJump 的 profile 文件名）。

---

## 11. 已知坑（代码里有证据的）

1. **Play 上传没有开关。** 每次 `build.yml` 都会把 AAB 推到 Play internal（`build-android.yml:115-128`）。想只出 APK 不碰 Play，用 `build-android-hotfix.yml`。
2. **`build_windows_store_package` 会真的发布**（`build-windows.yml:199`），见 §2。
3. **tag 里的 `+N` 不等于安装包的 build number**（§3）。
4. **Flutter 与 Xcode 都没锁版本**（`channel: stable`、`xcode-version: latest-stable`）：同一个 commit 隔几周再跑，工具链可能已经变了。
5. **Windows 构建不跑 `gen-l10n`**（§4.2 第 3 点）：源码仓库必须把 `lib/l10n/app_localizations*.dart` 生成物一起提交（源码仓库里它们确实是 git 跟踪的：`git ls-files lib/l10n` 列出 `.arb` 和 `app_localizations*.dart`，`l10n.yaml` 的 `output-dir: lib/l10n`）。只改 `.arb` 不提交生成物，Android/Apple 在 CI 现场重生成能过，Windows 就用旧字符串。
6. **VC++ 运行库必须随包发**（`build-windows.yml:109-139`），本地测不出来，别删这步。
7. **macOS 公证要等**：`--timeout 90m`（`build-apple.yml:271`），macOS job 可能跑一个半小时以上；没等到不会失败但 DMG 不会发布，重跑即可。
8. **DMG 段写死了 BuddhaJump 的 profile 文件名**（`build-apple.yml:228, 230`）。
9. **DMG 段签的是 `*.appex`**（`build-apple.yml:227-241`），源码仓库 `macos/` 下也只有 `ServiceExtension` 目录、没有 System Extension target——见 §12 第 8 条的运维记忆。
10. **macOS 主 App 不能带 `com.apple.security.network.server`**：源码仓库 `macos/Runner/Release.entitlements:9-32` 的注释记录了 2026-08-27 被 ASC 自动审核拒绝的原文，并说明该权限只留在 `ServiceExtension.entitlements:16`，主 App 已注释掉。
11. **Android JNA 两层锁**：源码仓库 `android/app/build.gradle.kts:99` 固定 `net.java.dev.jna:jna:5.17.0@aar`，`android/app/proguard-rules.pro:25-38` 有 `-keep class com.sun.jna.** { *; }` 等规则，注释（`:18-21`）记录了 R8 重命名 `Pointer#peer` 导致 `libjnidispatch.so` 找不到符号的崩溃。CI 用 `--obfuscate` 和 release 构建（`build.gradle.kts:63` 的 `proguardFiles`），两层缺一不可。
12. **Win32 `Create()` 会先跑一次 `OnDestroy()`**：源码仓库 `windows/runner/flutter_window.cpp:91-106` 的注释和 `if (flutter_controller_ || flutter_bridge_)` 守卫就是修复；1.5.5–1.5.9 每次启动第 6 秒被 `TerminateProcess` 杀掉。改 Windows runner 时别动那个守卫。
13. **Apple bundle id 前缀各品牌不一致，且与 Android 不同**：BuddhaJump 是 `io.fjolskylduoryggisverndar.buddhajump`（`ios/ExportOptions.plist:13-15`、`macos/Runner/Configs/AppInfo.xcconfig:11`），而 kamevpn 是 `com.fjolsky.kamevpn`、00000vpn 是 `com.fjolsky.vpn00000`（各自 `ios/Runner.xcodeproj/project.pbxproj` 的 `PRODUCT_BUNDLE_IDENTIFIER`）；Android `applicationId` 则是 `io.fjolskylduoryggisverndar.buddhajump` / `com.fjolskylduoryggisverndar.<brand>`（各自 `android/app/build.gradle.kts`）。旧文档里「统一 com.fjolsky.*」和「统一 com.fjolskylduoryggisverndar.*」两种说法都只对了一半。
14. **HEAD 不是 formatter-clean**：在源码仓库克隆上跑 `dart format --output=none --set-exit-if-changed lib`，105 个文件里 18 个会被改动。跑 `dart format` 会制造几百行与功能无关的 diff。
15. **只有 BuddhaJump 随包携带 sing-box 规则集**：源码仓库 `git ls-files assets/rules` 有 33 个 `.srs`，并有 `lib/utils/ruleset_helper.dart`；kamevpn、00000vpn 两个 fork 的同一命令返回 0。CI 不做任何规则集处理，靠源码仓库提交。
16. **端点发现只在 BuddhaJump 是活代码**：源码仓库 `lib/constants.dart:24-41` 把 `apiBaseUrl` 默认值刻意留空并解释了原因；kamevpn `lib/constants.dart:22-25` 默认值是 `https://api.fjolskylduoryggisverndar.com`，`lib/network.dart:171-174` 和 `lib/pages/landing.dart:119` 都以 `apiBaseUrl.isNotEmpty` 走「显式配置」分支，跳过 bit.ly → GitHub 的发现链。
17. **Android 日志回调的 JNI 线程附着**：源码仓库 `android/app/src/main/kotlin/com/fjolsky/buddhajump/bg/VPNService.kt:50, 100, 170` 引入并使用 `com.sun.jna.CallbackThreadInitializer`，注释说明了没有它时每条日志回调都 attach/detach 线程的问题。服务端那一半见 §12。
18. **Actions 分钟数**：两个账号都是 Free 计划，公开仓库不计费。2026-09-07 用 `gh api` 按 GitHub 的倍率（macOS ×10、Windows ×2、每 job 向上取整到分钟）统计 30 天：本仓库 78 次运行折合 6498 计费分钟；一个 fork 一次四平台 `build` 折合 ≈132–143 分钟（aiglefree-ci：6 次 790 分钟）。转私有前先算清楚。
19. **maskaura-ci 的公开 Release URL 被写进了杀软误报申诉材料**（本地文件 `project/fjolsky/maskaura-false-positive-submission.md`，引用 `https://github.com/BuddhaJumpApp/maskaura-ci/releases` 作为「CI 从源码直接构建」的证明）。ci 仓库转私有或改名会让这类举证失效。

---

## 12. 来自运维记忆（未在代码中复核）

以下内容来自维护者记忆或其他仓库，本仓库代码里找不到直接证据，写在这里只是提醒，不要当成已证实：

1. `ORG_TOKEN` 是 classic PAT（代码只能看出是 checkout 用的 token）。
2. 用 shell heredoc 写 Dart 文件会把 `$` 转义掉，曾发出连不上的包（1.5.4–1.5.6）。源码里没有留下相关注释。
3. Developer ID 直发的 macOS VPN 必须用 System Extension 而不是 appex，appex 在运行时被 `neagent` 拒绝——这意味着 §4.3 第 6 步产出的 DMG 可能装上了也连不上。代码只能证明它签的是 appex。
4. 版本号规则：补丁位 +1，到 9 后进位次版本号。仓库里没有任何脚本或注释体现。
5. Play 内部测试轨道的测试者只能在 Play Console 手点，API 加不了。`build-android.yml:123-127` 只说明了为什么选 internal 轨道。
6. Android 上引擎日志由 hydra 下发的配置关闭（服务端 v1.8.11）。本仓库和客户端代码里只有客户端那一半（§11 第 17 条）。
7. Apple bundle id 一经发布不能改——Apple 平台规则，与代码无关；§11 第 13 条只证明了当前值是什么。
8. `gh` 的活跃账号可能被其他并行会话切换；对 BuddhaJumpApp org 的 `gh workflow run` 依赖活跃账号，`export GH_TOKEN` 反而会 403/404。
9. 后端 `pkgs` 表里 BuddhaJump room 那行写死了完整的 GitHub Release URL，改 ci 归属前要 SQL 更新。
10. 7 个仍在 BuddhaJumpApp 下的非 BuddhaJump `<brand>-ci` 仓库要等下载链路迁移（R2）后再搬到 fjolskylduoryggisverndar。
