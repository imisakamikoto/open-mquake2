# iPadOS / IPA 云端构建

GitHub Actions 工作流：`.github/workflows/ipados-ipa.yml`。

每次推送到 `main` 且 `app/` 有变化时自动构建，也可以在 Actions 页面手动运行
`Build iPadOS IPA`。产物在运行记录的 **Artifacts → Mquake-iPadOS-IPA** 中。

## 未签名 IPA

未配置 Apple 签名 Secrets 时，工作流执行：

```bash
flutter build ios --release --no-codesign
```

然后把 `Runner.app` 装进标准 `Payload/` 目录生成未签名 IPA。未签名 IPA 不能直接
从文件安装，需要用 Sideloadly、AltStore 等工具使用你自己的 Apple ID / 开发证书重签。

## 签名 IPA（可选）

在仓库 `Settings → Secrets and variables → Actions` 添加：

| Secret | 必需 | 说明 |
| --- | --- | --- |
| `IOS_CERTIFICATE_BASE64` | 是 | `.p12` 开发证书的 Base64 |
| `IOS_CERTIFICATE_PASSWORD` | 是 | `.p12` 导出密码 |
| `IOS_PROVISIONING_PROFILE_BASE64` | 是 | `.mobileprovision` 的 Base64 |
| `IOS_TEAM_ID` | 是 | Apple Developer Team ID |
| `IOS_BUNDLE_ID` | 否 | 默认 `jp.misaka.mquake`，需与描述文件一致 |
| `IOS_EXPORT_METHOD` | 否 | 默认 `development`；也可设为 `ad-hoc` |

配置完整后，同一工作流会自动改用 `flutter build ipa`，上传签名 IPA。
