# 法语助手去弹窗

拦截**法语助手**（`com.eusoft.frhelper`）在查词、启动过程中不时弹出的会员推广弹窗。

- 只干预「弹窗展示」这一层 UI 行为
- 不修改账号、授权、订阅状态，不伪造会员身份
- 不拦截用户主动点击「开通会员」后的正常购买流程

> 源码、构建脚本与签名证书：https://github.com/Freesol916/FrhelperNagFree

---

## 为什么会有这个弹窗

它不是随机出现的 UI 缺陷，而是**有意设计的免费版转化机制**，频次由服务端下发的广告/活动配置控制，所以表现为「时不时跳出来」。

应用资源里直接写着这句话（`string/tool_reg_expire1`）：

> 开通VIP不会有弹窗打扰，还有额外增强功能。

同一组资源构成完整的「试用到期」弹窗：`您已试用一个月` / `开通VIP不会有弹窗打扰，还有额外增强功能。` / `查看VIP权益` / `延长2周`。

触发链路：

```
启动 / 查词累计  →  拉取广告活动配置(AdResponseModel)  →  按 display_count / hide_time 节流
      →  统一弹窗构造方法  →  自定义 AlertDialog.show()
```

因此拦截点选在 `Dialog.show()`：无论上游是启动计时、查词计数还是服务端投放，最终都要经过这里。

---

## 实现方式

- **作用域**：仅 `com.eusoft.frhelper`，按包名过滤，不注入其他进程
- **拦截点 1**：`android.app.Dialog#show()`
  - 命中规则时 `setResult(null)` —— 窗口不会被创建，**无闪烁**
  - `show()` 之后延迟 80ms 复查，兜底处理「文案后填充」的自定义弹窗
- **拦截点 2**：`Activity#startActivity` —— 仅记录推广页跳转，默认不拦截
- **判定规则**：类名特征（`promo|advert|interstitial|activityads|splashad`）+ 文案强特征（`开通VIP`、`查看VIP权益`、`您已试用`、`不会有弹窗打扰`、`延长2周` 等）+ 弱特征配行动号召词；其余放行并记录日志

判定以文案为主，不按类名拦 `VIPBottomDialog`，避免误伤正常购买页。

---

## 使用

1. 设备已 Root 并安装 **LSPosed**
2. 安装下方 Release 中的 APK
3. LSPosed → 模块 → 勾选「法语助手去弹窗」
4. 点进模块 → 作用域 → 勾选「法语助手」
5. 强行停止法语助手后重新打开

验证：

```bash
adb logcat -c && adb logcat -s NagFree
```

出现 `v1.0 attached` 表示注入成功，`BLOCK(show)` / `BLOCK(late)` 表示已拦截。

---

## 兼容性

- 传统 Xposed API，`xposedminversion=93`
- minSdk 23 / targetSdk 34
- 目标应用 v26.9.0（versionCode 34292），Android 16 / arm64-v8a 实测注入正常
- 其他版本未逐一验证；若新版改动弹窗构造方式，可能需要在 `NagFilter` 中补充特征

---

## 边界与免责

本模块属于用户对自身设备界面的控制，仅隐藏推广弹窗。它**不**解除任何付费功能限制、**不**伪造会员身份、**不**影响服务端授权校验 —— 需要 VIP 才能使用的功能仍然需要 VIP。

请自行确认所在地区的法律法规与软件许可协议，使用风险自负。本项目与欧路软件（Eusoft）无任何关联。

## 许可

MIT
