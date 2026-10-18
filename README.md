"# readstack-policy" 



## ReadStack 版权限制清单

本仓库为 ReadStack 提供经过人工维护、Ed25519 签名的版权限制清单。它只用于限制文档展示，不是版权授权清单，不会把未声明授权的文档变为已授权内容。

**初始化状态：尚未发布正式签名清单，尚未配置生产可信公钥。** `policy/payload-v2.draft.json` 是未签名草稿；它不能直接作为 App 的线上清单使用。完成受控签名、公钥独立核对及 App 信任配置后，才能启用同步。

- 正式发布文件：`policy/restrictions.signed.json`，当前不创建占位文件。
- 固定获取地址：`https://api.github.com/repos/GePenguin/readstack-policy/contents/policy/restrictions.signed.json?ref=main`。
- 文件结构：`schema/`；公钥管理：`keys/README.md`；发布流程：`docs/publishing.md`。
- 首个清单版本为 **2**，高于 App 当前内置版本 1。每次发布均包含完整活动规则和累计撤销 ID；版本只增不减，同版本不得修改 payload。

App 首次使用尝试检查，此后仅在使用事件发生、距上次成功检查超过 6 小时时再次尝试；不后台唤醒、不持续轮询。网络失败保留最后可信清单，没有 24 小时停读机制。已知禁令不会因省略、过期或离线解除，只能通过更高版本签名清单明确撤销。

### 版权与内容反馈

版权反馈邮箱：[chunqiuchan995@gmail.com](mailto:chunqiuchan995@gmail.com)。权利人可提出停止原生展示、限制收录或移除入口的请求，请说明仓库、路径、版本、原始 URL、问题和必要证据。维护者应记录处理状态，并在确认后按适当范围更新限制规则。

请勿把投诉人的邮箱、姓名、身份证明、邮件正文或其他私人信息写入公开规则、提交说明或 Issue。规则中的 `reason` 仅记录适合公开的最少处理依据。

ReadStack 是独立阅读工具，与第三方文档作者不存在默认合作或背书关系。本仓库的清单不替代作者声明、适用许可审查或法律评估。禁止通过其他展示模式绕过有效限制。
