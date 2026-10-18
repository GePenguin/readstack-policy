# 策略清单发布流程

## 当前状态

托管仓库为 `GePenguin/readstack-policy`，目标分支为 `main`。固定 Contents API 地址：

`https://api.github.com/repos/GePenguin/readstack-policy/contents/policy/restrictions.signed.json?ref=main`

初始化文件只提供未签名的 `policy/payload-v2.draft.json`。**尚无生产公钥，也没有正式 `policy/restrictions.signed.json`。** 没有有效签名和 App 内独立信任配置时，远程同步不能上线。不得上传测试签名充当正式策略。

以下步骤供维护者在自己的受控环境执行，不代表这些命令已执行，不会自动提交或推送。首次推送必须先取得用户明确确认。

## 1. 审核清单内容

- 初始 `revision` 为 **2**，必须大于 App 内置清单版本 1。后续每次内容变更都增加版本，范围为 `1..9007199254740991`，禁止回退或同版本修改 payload。
- 每次发布都保留 **全部当前有效规则** 与 **累计撤销 ID**，方便新安装取得完整状态。旧安装对省略的规则仍保留禁令，不能把删除规则行当作撤销。
- 解除禁令必须在更高版本的 `revokedRuleIds` 中明确加入原规则 ID。撤销记录只增不减；撤销 ID 不能再次用于活动规则。
- 同一个规则 ID 的属性不可改写。调整范围、原因或动作时，使用新 ID，并通过更高版本明确撤销旧 ID；注意其他仍匹配的禁令继续有效。
- `repository` 为 `owner/repo`；`scope` 为 `repository`、`directory` 或 `file`。仓库规则不带 `path`，其他规则必须提供相对路径。目录匹配有 `/` 边界，路径区分大小写。省略 `revision` 表示所有版本，提供时必须为完整 40 位 commit SHA。
- 动作按严重程度叠加：`disableNative` < `disableInApp` < `removeEntry`。较窄规则不能取消较广禁令。
- `reason` 只包含必要且适合公开的说明，不包含投诉人隐私、邮件正文或凭证。处理记录的私人材料应在仓库外受控保管。
- 活动规则和撤销 ID 各最多 2000 条。manifest UTF-8 最多 1 MiB，但外层签名 payload 更严格，最多 **512 KiB**；签名信封最多 **768 KiB**。不得以拆包或忽略规则绕过上限。

`schema/` 提供结构检查辅助，包含严格字段与基本边界；它不能验证签名、JSON 重复键、UTF-16/UTF-8 长度差异、规则 ID 冲突、跨版本单调性及所有累计语义。**最终以 ReadStack Dart 验证器和签名工具检查为准。**

## 2. 准备可信公钥与私钥边界

由维护者在受控环境自行准备 Ed25519 密钥。本流程不生成生产密钥。私钥必须放在任何 Git 仓库之外，不复制到本仓库、ReadStack 源码或构建资源中。

公开文件格式见 `../keys/README.md`：32 字节原始 Ed25519 公钥的标准 base64。维护者独立核对公钥、来源与指纹，确定稳定的 `keyId`；签名者和审核者都确认后，才可将公钥加入 App 内信任配置。不要信任下载响应携带的公钥。

## 3. 对单个 payload 签名并封装

发布工具位于 **ReadStack 应用仓库**的 `tool/policy_sign.dart`，不是本策略仓库的工具。先把下列示意变量替换成自己核对过的实际绝对路径；私钥路径只在本机设置，不写入提交。

```powershell
$ReadStackRoot = '<ReadStack 应用仓库绝对路径>'
$PolicyRoot = '<readstack-policy 仓库绝对路径>'
$PublicKeyFile = '<已核对的公开 base64 文件绝对路径>'
$PrivateKeyFile = '<任何 Git 仓库外的 Ed25519 私钥 PEM 绝对路径>'
$PolicyKeyId = '<已核对的 keyId>'
```

可使用包装命令，由本机 OpenSSL 对 **原始 payload 文件字节**签名，再验证并生成信封：

```powershell
dart run "$ReadStackRoot/tool/policy_sign.dart" sign --payload "$PolicyRoot/policy/payload-v2.draft.json" --private-key "$PrivateKeyFile" --public-key "$PublicKeyFile" --key-id "$PolicyKeyId" --out "$PolicyRoot/policy/restrictions.signed.json"
```

工具支持 `--openssl <本机 openssl 可执行文件路径>`。请在 ReadStack 仓库工作目录运行，以使用其已安装的 Dart 依赖。该工具不生成密钥，不应输出私钥；签名完成后仍需独立执行下面的验证。

也可以在受控签名环境自行生成 **64 字节二进制** Ed25519 签名，再仅把 payload、公开公钥和签名交给封装工具。签名文件不是 PEM 文本或 base64 文本。示意为：

```powershell
$PolicySignatureFile = '<仓库外的 64 字节二进制签名文件绝对路径>'
openssl pkeyutl -sign -rawin -inkey "$PrivateKeyFile" -in "$PolicyRoot/policy/payload-v2.draft.json" -out "$PolicySignatureFile"
dart run "$ReadStackRoot/tool/policy_sign.dart" assemble --payload "$PolicyRoot/policy/payload-v2.draft.json" --signature "$PolicySignatureFile" --public-key "$PublicKeyFile" --key-id "$PolicyKeyId" --out "$PolicyRoot/policy/restrictions.signed.json"
```

两种流程都对原始 UTF-8 字节签名，不先做 JSON 重排。签名后不要修改 payload 的空格、换行、编码或任何字段。信封里的 `payload` 是这些原始字节的标准 base64，`signature` 是 64 字节签名的标准 base64。payload 内的 `source` 必须与上述固定 API 地址完全相同。

## 4. 独立验证与 App 配置

```powershell
dart run "$ReadStackRoot/tool/policy_sign.dart" verify --envelope "$PolicyRoot/policy/restrictions.signed.json" --public-key "$PublicKeyFile" --key-id "$PolicyKeyId"
```

工具验证结果不代替变更内容审查。审核 payload、版本号、规则及累计撤销记录；确认信封能通过当前 App 同一验证实现，且没有私钥或私人反馈材料进入暂存区。

ReadStack 已在 `lib/features/documents/data/restriction_remote_source.dart` 的 `productionRestrictionSourceUri` 中固定上述 API 地址。只将独立确认的 `keyId` → 32 字节公钥标准 Base64 加入同文件的 `productionRestrictionPublicKeysBase64`，由 `productionRestrictionSyncConfig` 构造可信配置。不保存私钥，不从网络响应自动配置信任根。公钥改动需定向测试、版本发布及用户验收；仓库建立本身不代表 App 同步已启用。

生产信任配置变化会影响旧的持久化签名历史。不要移除仍用于验证历史的可信公钥，也不要直接切换来源后假定旧禁令可以丢弃。需要先审查迁移方案。

## 5. 审核后提交、经确认再推送

提交前查看 Git 差异与暂存文件，确认只有公开清单、Schema 和说明，不含任何私钥、临时签名或投诉人信息。仅在用户明确同意后才执行首次推送；此初始化流程不执行 push。

发布时 payload 与信封应一起提交以便审查。后续草稿文件名应对应新版本，不能覆盖旧版 payload 后继续沿用旧版号。生产文件始终使用 `policy/restrictions.signed.json`，其内部版本随发布递增。

客户端在首次使用以及距上次成功检查超过 6 小时的使用事件中尝试同步，无后台唤醒、无持续轮询。失败遵守退避与 GitHub 限流，保留最后可信清单；**不存在 24 小时过期停读**。已知禁令持续有效，直到更高版本可信签名清单明确撤销。首次无可信清单仍沿用内置与授权检查，不开放真实无协议文档的原生读取。

版权与内容反馈：[chunqiuchan995@gmail.com](mailto:chunqiuchan995@gmail.com)。公开发布规则时始终保护投诉人的私人信息。
