# 发版与签名指南

## 概述

本项目的发布流程为：推送一个 `v<major.minor.patch>` 格式的 Git 标签 → GitHub Actions 的 `.github/workflows/release.yml` 工作流自动运行 → 从标签解析 `versionName`、从 `app/build.gradle.kts` 中读取 `versionCode` → 构建签名后的 release APK → 创建 GitHub Release 并附带 APK、`mapping.txt` 和 `SHA256SUMS.txt`。签名完全在 CI 内部完成，签名材料通过 GitHub Secrets 注入，本地开发环境不需要也不应持有密钥。

## 前置条件

- JDK 17（直接使用 Android Studio 自带的 JBR 即可，无需单独安装）。
- 已克隆本仓库（`opencode_android_client`）。
- 拥有该 GitHub 仓库的 admin 权限，以便配置 Secrets。

## 生成签名密钥

签名密钥只需要生成一次，请妥善保管。使用任意一个 shell 执行以下命令：

**Windows PowerShell：**

```powershell
& "C:\Program Files\Android\Android Studio\jbr\bin\keytool.exe" -genkeypair -v -keystore "$PWD\opencode-client-release.jks" -alias opencode_client -keyalg RSA -keysize 2048 -validity 10950 -storepass "<STORE_PASS>" -keypass "<KEY_PASS>" -dname "CN=OpenCode Android Client, O=opencode_android_client, C=CN"
```

**macOS / Linux：**

```bash
keytool -genkeypair -v -keystore ./opencode-client-release.jks -alias opencode_client -keyalg RSA -keysize 2048 -validity 10950 -storepass "<STORE_PASS>" -keypass "<KEY_PASS>" -dname "CN=OpenCode Android Client, O=opencode_android_client, C=CN"
```

说明：

- **store 密码**：访问整个密钥库（keystore）的密码，对应 Secret `KEYSTORE_PASSWORD`。
- **密钥密码**：单独访问该条密钥（key）的密码，对应 Secret `KEY_PASSWORD`。
- **alias**：密钥在密钥库中的别名，固定为 `opencode_client`，对应 Secret `KEY_ALIAS`。
- **有效期**：`-validity 10950` 约等于 30 年，之后需要重新签名。
- **备份警告**：密钥丢失后无法为已上线的应用更新签名，请务必做双份离线备份（如 U 盘 + 加密压缩包），并把密码记录到密码管理器。
- **绝不提交**：`.gitignore` 已忽略 `*.jks`、`*.keystore`，密钥文件永远不要提交到 Git 仓库。

## 配置 GitHub Secrets

进入仓库 **Settings → Secrets and variables → Actions → New repository secret**，依次添加以下 4 个 Secret：

| Secret 名称 | 值 |
| --- | --- |
| `KEYSTORE_BASE64` | keystore 文件的 Base64 编码（单行） |
| `KEYSTORE_PASSWORD` | store 密码 |
| `KEY_ALIAS` | `opencode_client` |
| `KEY_PASSWORD` | 密钥密码 |

生成 `KEYSTORE_BASE64` 的方法：

**Windows PowerShell：**

```powershell
[IO.File]::WriteAllText("$PWD\keystore.b64", [Convert]::ToBase64String([IO.File]::ReadAllBytes("$PWD\opencode-client-release.jks")))
```

生成后把 `keystore.b64` 的内容（单行）填入 `KEYSTORE_BASE64`，并删除该临时文件。

**macOS / Linux：**

```bash
base64 -w0 opencode-client-release.jks | pbcopy
```

输出会直接复制到剪贴板，粘贴到 `KEYSTORE_BASE64` 即可。

## 本地验证签名构建（可选）

正常情况下不需要在本地签名——CI 会通过 Secrets 完成签名。如需本地验证，先在 shell 中设置 4 个环境变量：

**Windows PowerShell：**

```powershell
$env:KEYSTORE_BASE64 = "……"
$env:KEYSTORE_PASSWORD = "<STORE_PASS>"
$env:KEY_ALIAS = "opencode_client"
$env:KEY_PASSWORD = "<KEY_PASS>"
```

**macOS / Linux：**

```bash
export KEYSTORE_BASE64="$(base64 -w0 ./opencode-client-release.jks)"
export KEYSTORE_PASSWORD="<STORE_PASS>"
export KEY_ALIAS="opencode_client"
export KEY_PASSWORD="<KEY_PASS>"
```

然后构建并验证：

```bash
./gradlew assembleRelease
```

构建产物位于 `app/build/outputs/apk/release/`，用 `apksigner` 验证签名：

```bash
apksigner verify --print-certs app-release.apk
```

当 4 个环境变量全部设置时，`app/build.gradle.kts` 中 `hasSigning = true`，release 构建即为签名构建；否则为未签名构建。

## 发版仪式

1. **更新版本号**：在 `app/build.gradle.kts` 中把 `versionCode` 加 1（当前位于第 57 行），`versionName` 保持默认或按需修改。
2. **提交**：

   ```bash
   git add app/build.gradle.kts
   git commit -m "chore: bump versionCode to <N>"
   ```

3. **打标签**：标签使用语义化版本格式，不包含 versionCode：

   ```bash
   git tag v1.0.0
   ```

4. **推送**：

   ```bash
   git push
   git push --tags
   ```

5. **等待 CI**：GitHub Actions 的 Release 工作流会自动运行，构建签名 APK 并创建 GitHub Release。
6. **（可选）手动触发**：在 Actions 页面选择 Release 工作流 → Run workflow，通过 `workflow_dispatch` 输入 `versionName` / `versionCode` 手动触发。
7. **验证**：在 GitHub Releases 页面确认 Release 已创建，且包含 APK、`mapping.txt` 和 `SHA256SUMS.txt`。

## 安全注意事项

- **备份密钥库**：至少在两个不同位置离线备份 `opencode-client-release.jks`，密码存入密码管理器。密钥丢失意味着无法再为已发布的应用签名更新。
- **永不提交**：密钥库、`.env`、`*.jks`、`*.keystore`、`*.properties` 均已被 `.gitignore` 忽略，任何情况下都不要强行提交。
- **泄露处理**：一旦密钥库或密码泄露，立即在 GitHub 上轮换相关 Secrets，并考虑替换签名密钥（对于已上线的应用，更换签名密钥会导致无法覆盖更新，需评估影响）。
- **versionCode 单调递增**：每次发版必须比上一次大，且只增不减，否则安装包无法覆盖安装旧版本用户的升级路径会出问题。