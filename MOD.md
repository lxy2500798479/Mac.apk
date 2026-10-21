# MOD — 环境自洽层（Environment Fidelity Layer）

本仓库是 [kksimp/Mac.apk](https://github.com/kksimp/Mac.apk) 的 AGPL-3.0 派生版本，
目标是增加**环境自洽层**：让在 Mac.apk 内运行的 Android 应用看到的运行环境，
与一台真实 Android 设备自洽一致，从而让做环境自检的加固应用（mtguard / 乐固 /
梆梆 / 爱加密 / pairip 等）能够正常启动。

## 定位 / 非目标

**做**：把"环境的事实"做对——自己的路径、自己的信号处理器、自己的 maps、
自洽的设备画像、与 AOSP 一致的 syscall 语义。

**不做**：

- 不伪造设备认证结果（Play Integrity、厂商 attestation、服务端设备指纹）。
  这类校验要求"你就是那台被认证的硬件"，本地无法如实满足。
- 不 patch 目标 app 的代码（那是另一个层面的问题，也不该在这里解决）。
- 不向上游 kksimp/Mac.apk 提交这些改动（上游设计决策明确不做伪装层）。

## 已排除的假设（2026-10-21，均以美团 12.67.203 为被测样本）

| 假设 | 开关 | 结果 |
|---|---|---|
| `/proc/self/maps` 暴露 macOS 路径 | `MACAPK_ANDROID_MAPS_PATHS=1` | 生效（路径已变 `/data/app/~~…`），仍 abort |
| `stat` 返回值不忠实 | `MACAPK_PROCFS_STAT=1` | 仍 abort |
| `sigaction` 回读语义不对 | 读源码 `bionic_shim_sigchain.cpp:582` | **假设不成立**：`write_slot`/`read_slot` 语义已与 Linux 一致 |
| 设备属性缺失/不自洽 | `MACAPK_PAIRIP_DEV_PROPS` + `MACAPK_PROP_TRACE=1` | 注入自洽档案后仍 abort；观测到美团只读 3 个属性 |
| 宿主映射未隐藏 | `MACAPK_PAIRIP_HIDE_HOST=1` | 仍 abort |

美团在 abort 前唯一读过的属性：

```
ro.arch                      -> <ABSENT>（真机通常有）
ro.build.fingerprint         -> Mac.apk/macapk/macapk:11/PR1/1:user/release-keys
ro.product.cpu.abi           -> arm64-v8a
```

结论：**盲试开关的边际收益已经归零**。必须先建立观测层，再针对性对齐。

## 架构

```
┌─────────────────────────────────────────────────────────┐
│ W5 分发：自有 repo + CI 构建 + 签名 + 上游 rebase 脚本    │
├─────────────────────────────────────────────────────────┤
│ W4 覆盖：compat.d/<pkg>.json（按包覆盖，不写死在代码里）   │
├─────────────────────────────────────────────────────────┤
│ W3 忠实度表面：maps/procfs/ptrace/sigaction/dlopen/modules│
├─────────────────────────────────────────────────────────┤
│ W2 设备画像：fingerprint/brand/model/hw/cpuinfo/传感器…   │
├─────────────────────────────────────────────────────────┤
│ W1 探针审计：guest 的每一次环境查询 → 一条时间线           │
└─────────────────────────────────────────────────────────┘
```

**W1 是所有工作的地基。** 没有它，每次定位都是"改一个开关 + 跑一次 + 猜"，
4 次 A/B 只换来 4 个否定结论。

## 里程碑

| 里程碑 | 内容 | 估时 | 产出 |
|---|---|---|---|
| M1 | 探针审计层：open/stat/access/readlink/getprop/sigaction/ptrace/uname/sysinfo/maps 全量落盘 | 3–5 天 | 任一卷壳 app 失败可在 10 分钟内定位到"查了什么、我们答了什么" |
| M2 | 依审计结果对齐美团的具体判据 | 3–7 天（不确定） | 美团能过启动页 |
| M3 | 设备画像系统 + UI 可切换档案 | 4–6 天 | 内置 2–3 个自洽档案，前端可选 |
| M4 | 忠实度表面产品化（pairip 专用 → 兼容模式通用） | 3–4 天 | 开关可在 UI 操作，不需要环境变量 |
| M5 | fork 工程化：CI 构建、签名、版本号、上游 rebase | 3–4 天 | 一条命令出 dmg |

**M1+M2 是能否打开美团的关键路径（约 1–2 周，成功率不确定）。
M3–M5 是把"能用"变成"产品"（约 2 周）。**

## 风险

1. **服务端 attestation**：即使本地全过，美团服务端仍可能拒绝会话。本地无解。
2. **加固版本迭代**：mtguard 更新会重新引入检测，需要长期跟进（猫鼠）。
3. **过度伪装破坏其他 app**：pairip 表面已知会破坏普通 app 的 HotSpot 状态
   （实测 `FAKE_JNIENV` → SIGBUS），必须按包隔离，不能全局开。
4. **法律**：自用可接受；**不得分发修改过的第三方 APK**。本仓库只发布
   Mac.apk 本身的源码构建产物（AGPL 允许），不包含任何被修改的第三方应用。

## 上游同步

```
origin  = kksimp/Mac.apk       （上游，只读）
fork    = lxy2500798479/Mac.apk（本仓库，写入）
```

`main` 永远与上游保持一致；所有魔改在 `mod` 分支。定期
`git fetch origin && git rebase origin/main` 保持可合并。
