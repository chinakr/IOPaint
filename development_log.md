# IOPaint 二次开发记录（笔刷尺寸 Bug 修复）

项目目录：`/Users/chinakr/workspace/trae/IOPaint`
上游：`Sanster/IOPaint`（已归档，Apache-2.0）｜ 本机 fork：`chinakr/IOPaint`
工作分支：`fix/brush-size-on-image-change`｜ 修复提交：`05f7b8d`
记录日期：2026-10-27（Asia/Shanghai）

---

## 一、需求与决策时间线

1. **需求提出**：clone IOPaint 到 `workspace/trae/` → 二次开发 → fork 原项目 → 推送结果。明确诉求是排查并修复「首图处理后再拖入一张图，消除笔尺寸异常变化；刷新后重拖则正常」。要求先梳理需求与方案、不直接动手。
2. **需求梳理与根因假设**（首轮，源码级）：可见笔刷 = `baseBrushSize × brushSizeScale`；`setImageSize` 换图时只重算 scale、不重置 base → 尺寸跳变；刷新使 base 回到默认 40 → 看起来正常。同时指出原方案四项需修正：main 已是 1.6.0 无需「更新到 1.6.0」；上游已归档（可 fork、不能回提 issue/PR）；本机是 wheel 安装不含 git 历史，源码只能取自 GitHub；SSH 当时实测不通。
3. **只读核验**（本轮，全部命中，仅一处笔误）：见下节证据表。唯一偏差：版本号定义在 `setup.py:31`，不是 `pyproject.toml`（仓库根目录无该文件）。
4. **四项决策确认**：修复取向 **B（换图保持画布可见笔刷粗细不变）**；推送方式 **新生成专用 SSH key**；仓库形态 **fork 原仓库**；范围 **只修这个笔刷 bug**（滑杆 `defaultValue={[50]}` 与受控值混用的认知不一致按约定未动）。

## 二、核验证据

| 断言 | 结果 | 证据 |
| --- | --- | --- |
| 上游已归档 | 是 | GitHub API `archived=true`，最后 push 2025-04-29，23.3k stars，Apache-2.0 |
| main = 1.6.0、无 1.6.0 tag | 是（文件名为 setup.py） | `setup.py:31 version="1.6.0"`；`python_requires=">=3.7"`；tags 仅到 `iopaint-1.5.3` |
| 根因 = base × scale | 成立 | `states.ts:783-785` `getBrushSize = baseBrushSize * brushSizeScale`；`:981-988` `setImageSize` 重算 scale、不碰 base；`:255` 初始 base = `DEFAULT_BRUSH_SIZE`；const.ts：DEFAULT 40 / MIN 1 / MAX 200 |
| setImageSize 仅尺寸变化时调用 | 成立 | `Editor.tsx:246-248` 有 `width !== imageWidth \|\| height !== imageHeight` 守卫；另有 `states.ts:543/590` 处理结果回写调用 |
| 滑杆语义混用 | 成立 | `Editor.tsx:932 defaultValue={[50]}`、`:937 value={[baseBrushSize]}`、`:731` 滑块改 base、`:898-907` 滚轮、`:603-613` 快捷键均只改 base |
| SSH 不通 | 成立（修复前） | 原三对 key（id_ed25519 / id_rsa / myclaw）均 `Permission denied (publickey)`；config 无 github 条目；`gh` CLI 未安装 |
| trae/ 无 IOPaint | 成立 | 目录内 24 个项目，无 IOPaint |

## 三、执行过程

1. **clone**：完整历史克隆至 `trae/IOPaint`，main @ `61a759f`，34MB。
2. **专用 SSH key**：`~/.ssh/iopaint_github_ed25519`；`~/.ssh/config` 最前插入 github.com 条目（`User git` / `IdentityFile` / `IdentitiesOnly yes`，避免白试现有 key），原有内容完整保留（备份 `ssh_config.bak`）。沙箱内写 `~/.ssh` 被系统拒绝（EPERM），改主机侧授权执行；过程中产生的 `~/.ssh/config.new` 残留已走系统废纸篓安全删除。
3. **修复实现**（`web_app/src/lib/states.ts`，+24/-2）：`setImageSize` 先取换图前的 `imageWidth/imageHeight/brushSizeScale`，计算 `newScale`；仅当**非首次加载**（前值 > 0）且 scale 变化时，按 `visible = base × oldScale` 反算 `newBase = clamp(round(visible/newScale), MIN_BRUSH_SIZE, MAX_BRUSH_SIZE)`；首次加载保留原设计（base 40 × scale）。同时补 `MIN_BRUSH_SIZE` import。
4. **构建环境**：`npm ci`（package-lock 驱动）→ 430 包 / 475MB / 37 秒；`npm run build` = `tsc && vite build` → tsc 类型检查通过，产物 `web_app/dist` 3.1MB（`index-MxoDzEAF.js` + `index-chrgyh9P.css`），并同步一份到 `iopaint/web_app`（该路径仓库不跟踪，供源码运行）。
5. **免装重依赖的验证路径**：发现现装实例为 **uv tool 安装的 IOPaint 1.6.0**（与 main 同版本），位于 `~/.local/share/uv/tools/iopaint/lib/python3.12/site-packages/iopaint/`，遂将新构建产物**整目录热替换**进去（原目录改名 `web_app.orig-1.6.0`，另在工作区留副本），避开 torch 等数 GB 依赖重装。
6. **提交与推送**：分支 `fix/brush-size-on-image-change`，commit `05f7b8d`；`git remote rename origin upstream` + 新增 `origin = git@github.com:chinakr/IOPaint.git`；`git push -u origin` 成功，`git ls-remote --heads origin` 核对远端 `refs/heads/fix/brush-size-on-image-change = 05f7b8d...`、`main = 61a759f`（仍与上游一致）。
7. **用户回报**：公钥已加入 GitHub 账号；fork 已存在（API 确认 `fork=true`、`parent.full_name=Sanster/IOPaint`）；SSH 实测 `Hi chinakr!`；刷新加载后「之前的 Bug 好像没有了」。

## 四、补丁生效的物证

- **改前**（`web_app.orig-1.6.0` 两个 bundle 一致）：`o.imageWidth=n,o.imageHeight=r,o.editorState.brushSizeScale=Math.max(Math.min(n,r),512)/512` —— 只改 scale。
- **改后**（现用 bundle）：`...d(f/u),rk),r1)}o.imageWidth=n,o.imageHeight=r,o.editorState.brushSizeScale=u})` —— 赋值前先 `Math.round(f/u)` 反算 base 并被 `rk`/`r1`（MAX 200 / MIN 1 内联常量）钳制。
- **一致性**：已装 `index-MxoDzEAF.js` 与 `dist/` 同名文件 md5 相同（`6a94df25…`）；`index.html` 亦相同（`22560490…`）；CSS 哈希与原版相同 → 本次仅改 JS 逻辑。
- **压缩产物验证方法论**：不能用 `grep 标识符` 判定（常量被内联、变量被改名，0 命中不代表未生效）；应比对锚点（`brushSizeScale`）附近逻辑 + md5 校验已装文件与产物一致 + 核对 `index.html` 引用。

## 五、已知副作用与风险

1. **滑杆读数换图时会跳变**：滑块显示的是 `baseBrushSize`，方案 B 会反算该值，故画布笔刷圈不变、滑杆数字变。已事先与你确认接受。
2. **wheel 原版 assets 含两份无引用旧产物**（`index-01mZZypB.js`、`index-0foJStMW.css`），整目录替换后不再存在；现用 `index.html` 只引用新 js/css，自洽。若运行时出现相关 404，回滚即可。
3. **上游归档**：改动无法回流 `Sanster/IOPaint`（不能提 issue/PR），fork 仅保留血缘。

## 六、回滚方法

- **撤销已装实例的热替换**（单条）：
  `mv ~/.local/share/uv/tools/iopaint/lib/python3.12/site-packages/iopaint/web_app{,.patched} && mv ~/.local/share/uv/tools/iopaint/lib/python3.12/site-packages/iopaint/web_app.{orig-1.6.0,}`
  另有工作区副本 `iopaint-web_app-backup`（原始 1.6.0 前端）。
- **撤销 ssh config 改动**：删除 `~/.ssh/config` 开头 github.com 那 5 行，或用 `ssh_config.bak` 覆盖（原值：仅 `ServerAliveInterval 60` + `Host tencent-server` 段）。新增 key 文件 `~/.ssh/iopaint_github_ed25519{,.pub}` 可保留或删除。
- **撤销代码修复**：`git revert 05f7b8d` 或直接使用上游原版 `upstream/main`。

## 七、可选后续（未开工）

- 统一 `Editor.tsx` 滑杆语义（`defaultValue={[50]}` 与受控值混用、显示值 ≠ 画布可见值），即当初的方案 C。
- 键盘 `decreaseBaseBrushSize` 在 base ≤ 3 时可减到 0/负值（上游既有小瑕疵，与本 bug 无关）。
- 是否把 `main` 也推到 fork 并加 README 血缘说明（当前 main 与上游一致，未额外改动）。
