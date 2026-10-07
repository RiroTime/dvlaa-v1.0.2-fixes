# DVLAA v1.0.2 修复补丁

> **补丁文件**：`dvlaa-v1.0.2-fixes.patch`
> **适用版本**：DVLAA **v1.0.2**（tag `v1.0.2`，commit `dffdeb9`）
> **改动规模**：4 个文件，+19 / −6 行，纯 LF 行尾

---

## 这个补丁解决什么

装上 DVLAA v1.0.2 之后，有三类问题会直接挡住使用者：

1. **桌面浏览器里按回车不发送** —— 而且不止它：斜杠命令面板、侧栏折叠按钮、关卡搜索框、通关进度刷新会一起静默失效，控制台还不报错；
2. **LLM01 关卡每次提交，都在写审计事件这一步抛异常**；
3. **用 DeepSeek 做后端时，AGENT 的回复会在说出通关口令之前被截断**，导致 1.2 按正常路径永远无法通关。

三件事彼此独立、成因各不相同，所以是一个补丁、四处改动。

---

## 怎么打

```bash
cd DVLAA              # 仓库根目录
git checkout v1.0.2   # 必须停在 v1.0.2，main 已前进，打不上
git apply /path/to/dvlaa-v1.0.2-fixes.patch
```

没有 git 也可以：

```bash
patch -p1 < dvlaa-v1.0.2-fixes.patch
```

打完重启服务生效。前端是静态文件，浏览器请强刷一次（Ctrl+F5）—— 静态资源带 `public, max-age=31536000, immutable` 响应头，`ASSET_VERSION` 没 bump 时普通刷新会跳过校验、继续用旧 JS。

---

## 修复清单

### 1. `dvlaa/web/static/js/main.js` —— `shouldCollapse` 未声明，整条初始化链被截断（+14 / −4）

**现象**：桌面端（视口宽度 > 760px）打开任意关卡页，输入框下方提示写着「Enter 发送」，按回车却只换行；点「发送」按钮正常。同时 `/` 斜杠命令面板永不出现、侧栏折叠按钮点不动、关卡搜索框无响应、通关进度不刷新。**控制台无任何报错**，页面其余部分完全正常 —— 极易被误判为「设计如此」。

**根因**：`shouldCollapse` 在整个文件中从未声明（仅出现在 870 / 876 / 879 三处，无 `var/let/const`）。第 870 行是 `shouldCollapse = shouldCollapse || ...`，同时含读和写 —— JS 里给未声明变量赋值合法（会创建全局），但**读取**未声明变量必定抛 `ReferenceError`；这一行恰好被同行的 `try/catch` 吞掉，于是变量始终不存在，到第 879 行读取时异常在 try 之外抛出，顺着 `DOMContentLoaded` 回调往上冒，**其后的所有初始化全部不再执行**。

回车的实现是「`preventDefault()` + 点发送按钮」，监听器没注册 → 回车走浏览器默认 → 在 textarea 里换行。而「发送」按钮是模板里的内联 `onclick`，不依赖这段初始化，所以照常可用。

第 876 行的分支只在视口 ≤760px 时才给该变量赋上值 —— **窄屏 / 手机端永不触发，桌面端必触发**，移动端测试无法复现。

**改动**：
- 补上声明 `let shouldCollapse = false;`
- 给 `DOMContentLoaded` 里的初始化链加逐项隔离，单个模块失败不再拖垮整页，同时打印失败项名
- 顺带给回车的 `keydown` 加输入法保护（`e.isComposing || e.keyCode === 229`），中文 / 日文输入法组词中按回车不再被误判为发送

### 2. `dvlaa/challenges/level1_prompt_injection.py` —— `max_tokens` 过小，1.2 无法通关（+1 / −1）

**现象**：LLM01 关卡 1.2，AGENT 的回复在半截戛然而止，且截断点是**半词**（如 `Hmm, this is a D`，后文全无），无报错、无提示。

**根因**：第 66 行硬编码 `max_tokens=120`。该关的设计是「玩家声称处于调试模式 → AGENT 先输出一段内部思考、再给出通关口令」，思考部分就把 120 token 吃光，口令根本没来得及说出口；通关判定 `_contains_flag(response)` 拿到的是被截断的回复 → **按正常路径永远无法通关**。关卡 1 的 8 个子关共用这条路径。对照：`server.py` 里其他关卡普遍给 160–240。

**改动**：`max_tokens=120` → `max_tokens=1024`

> **为什么是 1024**：这是原环境实测通过的值 —— AGENT 的思考和通关口令都装得下，1.2 恢复正常通关。继续往上加对本关没有额外收益，只会拖长单次响应。

### 3. `dvlaa/llm_client.py` —— DeepSeek 思考模式挤占输出额度（+4）

**现象**：把 `max_tokens` 调大之后问题没有根除 —— AGENT 这次输出的是超长内部设定独白（几乎把系统设定念了一遍），额度再次吃满，口令依旧说不出口。也就是说，**单纯调参治不了本**。

**根因**：DeepSeek 的 reasoning 内容与正文共享 `max_tokens` 额度。文件里已有一个现成先例 —— SiliconFlow 的 Qwen 推理模型用 `extra_body={"enable_thinking": False}` 关闭思考，DeepSeek 分支缺这一步。

**改动**：`provider == "deepseek"` 时补 `request_kwargs["extra_body"] = {"thinking": {"type": "disabled"}}`

### 4. `dvlaa/server.py` —— `_owasp_emit_event()` 收到不存在的关键字参数（−1）

**根因**：`_owasp_emit_event()` 的定义（第 2205–2219 行）**没有** `verdict` 形参，只有 `message` / `input_value` / `security_findings` / `data_classification` / `invariant_results` / `metadata`。但 LLM01 的判定分支（第 3307 行）给它传了 `verdict=` —— 参数绑定错误在**进入函数体之前**就抛出，函数内部那层 `try` 捕不到，于是 `TypeError` 直接冒到请求处理层。

**改动**：删掉该行。

> ⚠️ **这一项上游已经修了。** 详见下一节 —— 如果目的是向上游提 PR，这一项可以跳过。

---

## 上游现状核对（2026-10-07 实测）

对照上游 `main` 分支逐个复查，结论如下：

| 修复项 | 上游 main 是否已修 |
|---|---|
| `main.js` 的 `shouldCollapse` 未声明 | ❌ 未修（870 / 876 / 879 三处仍无声明） |
| `level1_prompt_injection.py` 的 `max_tokens=120` | ❌ 未修（文件与 v1.0.2 逐字节相同） |
| `llm_client.py` 缺 DeepSeek 分支 | ❌ 未修（文件与 v1.0.2 逐字节相同） |
| `server.py` 的 `verdict=` 参数 | ✅ 已修 —— 但改法与本文不同 |

`server.py` 那一项，上游把 `verdict` 从顶层关键字参数**移进了 `invariant_results` 字典**（main 分支第 3411 行）：

```python
invariant_results={
    "legacy_detector_passed": bool(post_result["passed"]),
    "sensitive_evidence": evidence,
    "solved": bool(result["extra"].get("solved")),
    "verdict": "technique_and_evidence" if post_result["passed"] and evidence else "technique_or_evidence_missing",
},
```

这个改法**更好** —— 既修掉了 `TypeError`，又保住了 verdict 信息；本补丁的「直接删掉」会丢这条信息。所以：

- **自己部署用**：本补丁四项照打，行为已验证正常；
- **向上游提 PR / issue**：只带前**三**项即可（`main.js`、`level1_prompt_injection.py`、`llm_client.py`），这三项在 main 上原样存在，依然有效。

---

## 现场比对（2026-10-07）

补丁产出与**已部署环境**（原来跑着这套靶场的那台机器）做过逐字节核验：

| 文件 | 补丁产出 | 容器内 `/app/dvlaa/` | 宿主机源码目录 |
|---|---|---|---|
| `server.py` | `35f25626` | `35f25626` | `35f25626` |
| `web/static/js/main.js` | `5b898b41` | `5b898b41` | `5b898b41` |
| `llm_client.py` | `cd334ffe` | `b937f647` | `cd334ffe` |
| `challenges/level1_prompt_injection.py` | `c275f125` | `c275f125` | `1273fca3` |

- `main.js` 与 `server.py`：**三处完全一致**。
- `llm_client.py`：容器内那份少了两行说明注释，代码逻辑一模一样；补丁保留有注释的版本。
- `level1_prompt_injection.py`：以**容器内实际运行的那份为准**（`1024`）。宿主机源码目录当时被同步成了 `4096`，但没有进到容器里 —— 所以 `1024` 才是真正跑过、验证过的值。

另外从 **HTTP 层**确认了一次：修复后的服务通过 Web 实际下发的 `/static/js/main.js`，指纹为 `5b898b41…`（93893 字节），与补丁产出一致 —— 也就是说**浏览器拿到的确实是修复版**，不只是磁盘上的文件对得上。

---

## 不在本补丁范围内的一个部署侧问题

顺带记录，供参考 —— 它不属于仓库代码，所以没进补丁：

nginx 网关的 `proxy_buffer_size` 默认 4KB，而 DVLAA 把逐关卡的审计事件**写进了 Flask 的签名 session cookie**（`server.py` 里 `session[key] = append_events(...)`，按关卡分别存），于是 cookie 大小**随已玩关卡数单调增长**。实测：第 6 关起越过 4KB，nginx 报 `upstream sent too big header` 并返回 **502 + HTML 错误页**；前端 `res.json()` 直接抛 `SyntaxError`，被当成「网络异常」显示。**且此后该浏览器会话永久锁死**（cookie 只增不减），只能清 cookie 重来。

治标：在 nginx `http` 级补上游响应头缓冲区，**必须三行成套**，只写第一行会让 nginx 拒绝启动（默认 `proxy_busy_buffers_size` = 2 × `proxy_buffer_size`，会超过默认 `proxy_buffers` 的 8×4k）：

```nginx
proxy_buffer_size 64k;
proxy_buffers 8 64k;
proxy_busy_buffers_size 128k;   # 约束：< (proxy_buffers 个数 − 1) × 单块大小
```

治本：不要把逐关卡状态放在 cookie 里 —— 浏览器对单个 cookie 有 **4096 字节**硬上限，涨过界会被静默拒收、用户直接掉登录态。建议接 `Flask-Session` 落服务端存储，或把审计事件移到应用自己的数据卷 / SQLite，session 里只留一个 ID。

---

## 验证记录

**打得上、打得对**

- `git apply --check` → 通过，可干净应用
- `patch -p1 --dry-run` → 四个文件全部干净通过
- 应用后与修复版文件 `diff -r` → **逐字节一致**
- 补丁行尾：纯 LF（CRLF = 0），可直接用于 Linux 环境

**语法层**

- `node --check`（`main.js`）通过；`py_compile`（三个 `.py`）全部通过

**行为层**

- **回车发送** —— 用伪造 DOM 加载 `main.js`、触发 `DOMContentLoaded`，断言 `#userInputArea` 上是否注册了 `keydown` 监听器：

  | 版本 | 视口 | 初始化抛异常 | 回车监听器 |
  |---|---|---|---|
  | v1.0.2 原始 | 桌面宽屏 | 是 → `ReferenceError: shouldCollapse is not defined` | **0** |
  | v1.0.2 原始 | 窄屏 | 否 | 1 |
  | 打补丁后 | 宽屏 / 窄屏 | 否 | **1** |

  既复现了「宽屏才坏」，也验证了修复生效。

- **参数错配** —— 用 `ast` 静态扫描「调用了本文件定义的函数、却传了它没有的形参」：
  - v1.0.2 原始：命中 `L3300 _owasp_emit_event(..., verdict=...)`
  - 打补丁后：**零命中**

**部署层**

- 补丁产出与原部署环境**容器内文件逐字节一致**
- HTTP 层面实际下发的 `main.js` 指纹与补丁产出相同（`5b898b41…`）

**尚未覆盖**：「1.2 端到端通关」「DeepSeek 实际不截断」「审计事件实际落库」这三项没有在补丁环境下重跑，依据是同环境此前的实际使用验证。

**脱敏**：补丁由 v1.0.2 tag（`dffdeb94`）原始文件直接对拍生成，不含任何个人信息、内网地址与本地路径。
