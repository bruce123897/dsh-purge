# 本仓库补丁说明（社区存档）

本仓库是 **dsh-purge 1.1.71 的社区存档分支**，基于 mmparko/dsh-purge 镜像（原作者 YuJunZhiXue，原仓库已闭源删除），在此之上叠加了一处本地修复。

## 修复内容：适配 dsh 0.2.1-rc.2

dsh 0.2.1-rc.2 的提示词模板引擎只注册 `provider` 和 `model` 两个变量，任何段落引用未注册变量都会在组装期抛错：

```
unknown prompt variable "{{cwd}}" in section "deployment:persona-suffix"; registered variables: provider, model
```

本补丁移除了三处 `{{cwd}}` 模板引用（工作目录上下文由 dsh 的 working-directory 服务另行提供，功能无损失）：

| 文件 | 改动 |
|---|---|
| `cordis.patch.yml` | 默认配置 `personaSuffix: Your working directory is {{cwd}}.` → 置空 |
| `lib/identity.js:8` | `CWD_ONLY_SUFFIX` 常量置空 |
| `lib/identity.js:156,164` | 身份剥离改写产物 `"Working directory: {{cwd}}. "` → 置空 |

补丁日期：2026-10-23。

## 署名（沿用镜像与原作者要求）

- 原作者：[YuJunZhiXue](https://github.com/YuJunZhiXue)，一切权利与署名归原作者
- 原仓库：https://github.com/YuJunZhiXue/dsh-purge （已闭源删除）
- 镜像来源：[mmparko/dsh-purge](https://github.com/mmparko/dsh-purge)，`MIRROR-NOTICE.md` 原样保留
- 许可证：MIT，非盈利条款沿用原 README 署名要求

## 恢复方法

```bash
# 还原到本机 ~/.dsh/plugins/dsh-purge 后，手动同步 profile 副本（两份非硬链接）
cp lib/identity.js ~/.dsh/profiles/desktop/node_modules/dsh-purge/lib/
cp cordis.patch.yml ~/.dsh/profiles/desktop/node_modules/dsh-purge/
# 完全退出并重启 DeepSeek Harness
```

## 修复 2：补丁 40 适配 0.2.1-rc.2（2026-10-23 03:24 写入、04:02 应用闭环）

rc.2 的 dsh-system-prompt 在 complete 分支前新增 requiredContexts 过滤行、contexts 改为保留必需项，补丁 40 的 6 个旧锚点全部失配，点「应用」报「失败：清洗没有写进全部宿主，已取消重启」。

修法：core.js 补丁 40 的 replacements 头部插入第 7 个变体（精确匹配 rc.2 形状），替换代码保留 requiredContexts 语义。实测：60/66 → 清洗成功全部落宿主，应用后自动重启生效。

## 修复 3：密封交换 helper 翻车导致的崩溃循环（2026-10-23 03:51 手动完成交换）

现象：点「应用」成功后，每次开机应用崩一次（crash dialog），不会自行恢复。（订正：最初记作「宿主子进程 7 轮重启后放弃」——经源码复核，解包树 app/lib/main.js 的 spawn 仅 1 处且带单飞守卫，无任何 respawn/MAX_RESTART 逻辑，外壳崩了就是崩了；crash log 里重复的禁用警告是一次 preflight 的确定性输出，非重生累积。）app.asar 始终原地。

根因：插件检测到桌面仍密封（app.asar 在）→ 判定补丁丢失 → 开机自动 reapply + 调度「挪走 asar + 整应用重启」；但其重启 helper 有已知故障（desktop.js:830 注释自认：进程杀干净、relaunch 没执行）→ 应用死、asar 没挪成 → 下次开机再来一轮。

修法（手动替它完成交换）：
1. 完全退出 DeepSeek Harness（确认 0 进程）
2. rename "D:\deepseek harness\resources\app.asar" → app.asar.bak（helper 脚本写死的目标名）
3. 重新启动应用 —— Electron 落到解包树 resources/app，插件自检通过（04:02 经「应用」按钮补全闭环，applied.json 04:03:01 落戳；desktop 面 settleMayRestart 恒 false，settle 自愈路径不可达，不会再自行调度重启）

注意：app.asar.bak 是官方原包，勿删（回滚与插件 unpacked 印章都依赖它）。dsh 桌面端自更新带回新密封包后可能复发同款「开机必崩」，处置同上。

## 修复 4：interpolate 放行集去掉 cwd（2026-10-23 独立复核后加固）

复核发现残留隐患：`lib/core.js:100/104` 的 `ALLOW_EXEC_PERSONA_BLOCK` / `ALLOW_EXEC_PERSONA_LINE` 仍含 `{{cwd}}`（当前为死代码，仅自身定义处引用）；而 `lib/identity.js` 的 `DSH_INTERPOLATE_ALLOWED` 原本放行 `cwd`，`escapeDshInterpolation` 会把 `{{cwd}}` 原样放过——将来任何一段真的带上它，rc.2 组装期仍会硬失败。

修法：放行集改为 `new Set(["model", "provider"])`，注释同步更正。此后任何文本里的 `{{cwd}}` 都被转义为字面 `{ {cwd}}`（非致命）；core.js 两处死代码字符串原样保留。三副本同步：`~/.dsh/plugins` 源、profile node_modules 副本、本仓库。
