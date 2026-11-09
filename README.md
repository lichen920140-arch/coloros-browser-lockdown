# ColorOS 系统浏览器停用记录
调查不全面，该禁用有风险


一份关于 ColorOS / realme 预装浏览器 `com.heytap.browser` 的**静态取证、风险分析、停用与回滚**的可复现记录。


1. 这个包到底是什么（59,944 个类的完整 Chromium，不是广告 SDK）
2. 停用它损失什么、保住什么（按"通话/短信/上网/相机/支付/系统更新"逐项判定）
3. 怎么停、怎么验、怎么回滚（含 suspend 与 disable 的差别、双 Package 记录导致的失败）

## 目录

- [正文：分析、取舍与可复现流程](docs/heytap-browser-lockdown.md)

## 适用与前提

| 项 | 说明 |
|---|---|
| 系统 | ColorOS 16（具体版本号已隐去） |
| 机型 | realme / OPPO 系实测机型；方法可迁移到 OnePlus 同代 ColorOS |
| 前提 | 无线调试（adb pair）+ 平台工具，**无需 root** |
| 风险 | 停用系统组件可逆，但需自行承担操作后果 |

## 三条最容易被忽略的结论

1. `pm disable-user` 对"出厂 + 商店更新"双记录的包**会失败**（返回 `new state: default`），要用 `pm suspend`。
2. `suspend` 是**按用户**的状态：`user 0` 之外还有访客（user 10）与分身（user 999），各自的默认浏览器角色要分别看，否则会漏。
3. 停用后"打开链接"是否正常，取决于 `BROWSER` role 指向谁——**不是**看包有没有被禁用。

## 许可

文档内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可，详见 [LICENSE](LICENSE)。
