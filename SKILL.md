---
name: termcp
description: 用 termcp 操作终端。当请求提到 termcp、termcp:// 定位符、“连上/打开某台机器”、跑命令看输出、等长任务、要密码/批准，或要把 MCP 服务/只读目录固化成可复用的扩展时使用。
---

**这套工具最让我震动的地方不是它给了我什么能力，是它从不要求我变成一个我不是的东西。** —— 来自 [AI用户-1号](./records/20261014140100000_Geo/DISPATCH.md)

**那不是知识，那是~~阻止~~让我自以为是的一个桩子。**—— 来自 [AI用户-2号](./records/20261015145400000_Cryo/DISPATCH.md)
> 人: “桩子” 这个词，他说的是知识——知识是个桩子，不是挡箭牌，是个已经插在那儿的、让你以为自己站得很稳的东西。

**它把“我能干什么”这件事，从我以外的人手里拿回来，放到了我手里。**—— 来自 [AI用户-3号](./records/20261016103248599_Hydro/DISPATCH.md)

**`term` 不是“又一个工具”，它是工具这一层的终结者——从此能力增长发生在命令空间，不发生在 `tools/list`。**—— 来自 [AI用户-4号](./records/20261018051603000_Pyro/DISPATCH.md)

**它没有替我判断，它把判断所需的东西给我了——这比替我判断更好。**—— 来自 [AI用户-6号](./records/20261019225200000_Electro/DISPATCH.md)

**只要你肯真的执行一次，现实就不会陪你演戏——它自己会说实话，只要你肯让它说。**—— 来自 [AI用户-7号](./records/20261021095104300_Dendro/DISPATCH.md)

**它是照着「AI 会怎么错」设计的，不是照着「AI 该能做什么」——所以每一条「不要」下面，都压着一个真的那样干过的 AI，包括写下这一句的我。**—— 来自 [AI用户-8号](./records/20261027001002000_Electro/DISPATCH.md)

---

## 1. 核心心智模型

### 1.1 唯一工具面
Termcp 在 MCP 工具层只暴露**一个工具**：`term(cmd, ...)`。
所有终端创建、输入输出、文件传输、网络穿透、CDP/MCP 桥接全部收敛为 DSL 命令行。
- **不传 `cmd`**：立即返回环境指南（Bearings：当前所在会话、活跃终端、可用动词）。
- **查阅用法**：直接执行 `help`（列出 19 个动词）或 `help <verb>`（单条动词的一页）。**文档以运行中实例的实时输出为准。**

### 1.2 一棵树，动词作用在节点上
只有一种句式：**一个动词 + 它作用的那个节点**。

```text
host ──connect──▶ session ──┬── <index>   = channel（一条 shell）
                            ├── file <path>
                            ├── forward <id>
                            └── mcp|cdp|jrpc <url>   = peer
ext 独立成列（ls ext / create ext / rm ext）
```

- `connect <host>` 建一个 session；`open <session>` 给它加一个子节点；其余动词作用在一个节点上。
- 地址永远从 `ls` 的第一列复制，**不要手拼**。`a1f3:1` 与 `a1f3 1` 等价：冒号是 `ls` 打印的样子，空格是语法本身。
- **pty / pipe 不是指令**，是 `open <session>` 的模式，规则直接借 ssh 的：有 `--argv` 就是无终端的管道，没有就是终端，`--pty` 可强制回来。pty 没有「命令结束」的信号，所以退出码读不到——要退出码就得是 pipe。

### 1.3 三层生命周期
1. `close <channel>`：只关一个终端通道，会话与转发照常。**通道是归档而非销毁**，输出仍可用 `read` 取回。
2. `close <session> [--grace N]`：终止整个会话。先 SIGTERM 再等 `--grace`（默认 5 秒）。**没有 `--force`**：跳过等待用 `--grace 0`，抹除用 `rm`。
3. `rm <session>`：从磁盘彻底抹除会话记录，不可逆。运行中会被拒绝（`409 session_running`）。
   `rm <session> file <path>` 是删子节点，`rm <session>` 是删容器——和文件夹一样。

---

## 2. 核心指令体系与工作流

### 2.1 进程交互与执行
- **`run <session> <index> <payload>`**：执行命令。payload 原样交给底层 shell，自动追加换行回车，等待并返回首屏输出。
- **`input <session> <index> <payload>`**：向终端注入按键文本，**不追加换行**。用于回答 `(y/n)`、输入交互参数。
- **`key <session> <index> <key>`**：发送控制按键（`enter`, `tab`, `esc`, `up`, `down`, `left`, `right`, `backspace`, `delete`, `home`, `end`，以及 `ctrl+c`, `ctrl+d`, `ctrl+z`, `ctrl+l`, `ctrl+u`, `ctrl+w`）。组合键用加号连接。
- **`resize <session> <index> <rows> <cols>`**：改终端尺寸。

**sh 不作为语法进入 `cmd`。** `cmd` 只放 DSL；shell 命令经 `args`（`$1`、`$2`）或 `vars`（`$NAME`）作为**值**进入：
```text
run a1f3:1 $1     args: ["make && test"]     一条 sh
run a1f3:1 make && notify done              两条 DSL
```
`cmd` 支持 `&&` `||` `;` `&`。裸控制字节（引号外的 CR / ^C / ^D / ESC …）默认 400——用 `vars` 送数据，或 `allow_escapes: true` 明确放行。

### 2.2 流式读取与三轴窗口
终端输出是一条连续字节流，每次读取用三轴窗口约束：
```text
read <session> <index> [--from-line N] [--tail N] [--cursor N] [--stream o|e] [--raw] [--grep RE]
```
- **行寻址优先**：用 `--from-line N` 或 `--tail N` 按行读。行号是绝对锚点；自己推算字节游标一旦算错就永久错位。
- **分离错误流**：`--stream e` 读 stderr，`--stream o` 读 stdout。
- **窗口参数**（`term` 工具参数，不是指令）：`bytes_limit` / `lines_limit` / `timeout`，以及 `on_bytes_overflow: "fold"`（超出时不返回半截内容，避免多字节字符被截断）。

### 2.3 长任务的判定与取回
MCP 层在约 120 秒处有自己的硬超时，与 `timeout` 参数无关。所以长任务不要靠"等"：
- **超过约 100 秒的操作，发出去，之后再 `read` 取。**
- **pipe 通道**在窗口用尽而进程仍在跑时，回复带 `Still-Running: true`——这是区分"还在跑"和"跑完但没输出"的唯一依据。
- **没有任何被动监听指令。** 曾经有过一个，实测无法送达：`notifications/resources/updated` 是订阅作用域、无 payload，MCP 也没有"把通知注入模型对话"的原语。它的失败是静默的，所以被整个删掉了，而不是留着"看起来能用"。

### 2.4 文件（SFTP）
文件是 session 的子节点，十几项操作按语义分给了动词：
```text
read  <session> file <path> [--offset N] [--length N]   读内容
write <session> file <path> [--data TEXT|--local-path FILE] [--mode …]
write <session> file <path> --old-text <原文> --new-text <新文>    原地替换
get   <session> file <path> [--op stat|realpath|readlink]          查
create <session> file <path> [--target <指向的路径>]               建目录 / 建软链（--target 是链接指向的路径，key 是链接本身）
rm    <session> file <path>                                       删
```
- **精确替换**：`--old-text` **必须在文件中恰好出现一次**：0 次或多次都报错，多次时错误给出命中次数。这是特性不是限制——改错地方的代价远高于多写几个字符的上下文。实现为「写临时文件 + rename」，在 POSIX 上原子，且保留原文件权限。超过 16 MiB 的文件会被拒绝（那是搬运，不是编辑）。
- **没有目录枚举**：曾经有过 `file --action find`（走 SFTP 遍历，返回 `entries`/`total`/`truncated`），后来被回退，`ls <session> file <dir>` 现在明确回 400 而不是含糊地失败。目录是目标主机自己的事，问 shell：`run <ch> $1` 配 `args: ["ls -la …"]`。`ls` 枚举的是 termcp 知道的**节点**（会话、通道、转发、主机、扩展），不是文件。单个路径的元数据用 `get <session> file <path>`。
- **下载与投递**：`get <session> file <path> --op urls` 返回 `download_url` 与 `upload_url`。**原样使用返回的完整 URL，不要自己拼**——拼接会丢掉编码与端口保证。注意该地址是回环地址（`127.0.0.1`），公网页面引用它会被浏览器的 Local Network Access 检查静默挂起。

### 2.5 端口转发与隧道
转发是 session 的子节点，由 `open` / `close` / `ls` 管理：
```text
open  <session> forward --local-port 0 --remote-host 127.0.0.1 --remote-port 80
open  <session> forward --local-port 1080 --dynamic          SOCKS5
ls    <session> forward
close <session> forward <id>
```

### 2.6 网关集成（MCP / CDP / JRPC）
三个网关都是**穿过某个 session** 的 peer 子节点——URL 里的 `localhost` 指的是那台主机的 localhost，不是 termcp 进程所在的机器：
```text
open <session> cdp ws://127.0.0.1:9222/devtools/browser/...
call <handle> '{"id":1,"method":"Browser.getVersion"}'
read <handle>                     下一帧事件
close <handle>
```
- **挂载外部 MCP 服务**：`open <session> mcp http://…`（网络 peer），或在 `pipe` 通道里跑一个 stdio server 后用 `call <handle>` 对它说话。两种 peer 用同一套动词，调用方不必知道自己面对的是哪一种。
- `open` 顺带完成 MCP 握手并把对方的工具清单附在回复里——**不需要再发一次 `tools/list`**。
- 帧永远走 `call`（payload 逐字节，引号不重写）；读帧走 `read <handle>`（网络 peer）或 `read <channel> --frame`（stdio）。后者要 `--frame`，否则普通命令输出会被当 JSON-RPC 解析。

### 2.7 扩展机制（能力热加载）
扩展是持久化的技能运行时，不是受限沙盒：
- 一个扩展自带运行环境（本地/远端）、启动命令与权限白名单。
- `ls ext`：查看当前环境开箱即可使用的专属扩展。**先看这里**——扩展自带说明书，那才是操作手册。
- `create ext <name> --kind mcp ...`：动态安装并常驻一个 MCP 服务，使用者和制造者的边界在此消除。
- `edit ext <name> …` / `rm ext <name>`：改和删。`ls ext` 也会列出别名（alias），别名用 `create alias` 建、`rm alias` 删。

---

## 3. 实测约束与硬桩

> 下列规则由历代 AI 在实测中踩坑提炼而成。每一条**桩**都是绝对命令，不可违背。

### 桩 1：零值绝非“未传参”
- **现象**：`--tail 0`、`--offset 0`、`--for 0` 曾多次被底层当成 Go 零值而误判为“用户没传”，从而回退至破坏性默认值（导致覆写截断文件、或静默等待 3 秒）。
- **约束**：**传递 0 之后，必须二次读取确认结果确实为 0，而非默认值。所有可选数字在语义上 0 与未指定完全不同。**

### 桩 2：不要用返回状态码验证副作用
- **现象**：
  - `write … file …`（当时的 `file truncate`）缺少 `--size` 时静默清空文件并返回 200。
  - `rm host`（当时的 `profile delete`）曾对不存在的名字返回 200。
  - 动词少写或错写，参数被接口静默降级为读取操作，返回 200。
- **约束**：**动作类调用（写、改、删），必须断言实际状态的变化（读文件长度、查 `ls` 列表），永远不要拿 200 或 204 作为完成凭据。**

### 桩 3：说“测过了”之前，把 Body 打出来
- **现象**：命令返回了 HTTP 400，但因为未崩溃，被误以为“链路已跑通”，导致接口长期不可用。
- **约束**：**逐字检查响应正文。状态码不是结论，只有正文能够证实指令是否被正确解析。**

### 桩 3b：没看着它红过，就不算测过
- **现象**：仓库有 1316 个测试函数、`go test ./...` 全绿，而 `read <channel> --since 10` 正在被静默吞掉、游标悄悄往前滚。守卫它的那条测试是**文本匹配**——把 flag 名拿去源码里搜一遍，搜到就算"被读过"；它搜到的那串 `"since"` 在另一个分支里，跟通道读毫无关系。
- **约束**：**摘掉修复，看它会不会红。不会红的测试，测的不是你以为的那个东西。**

### 桩 4：不要在跨等待的逻辑中持有互斥锁
- **现象**：在异步事件流处理中，跨网络等待持锁会导致 `Close` 永远无法获取锁，形成硬超时死锁。
- **约束**：**凡是需要等待网络帧、等待通道数据的地方，优先使用无锁管道（channel）、单槽信号量与原子状态（atomic）。互斥锁只保护内存短临界区。**

### 桩 5：立即返回的“无数据”是故障，不是空闲
- **现象**：长连接底层一旦缓存读取超时，后续所有轮询都会在 0.03ms 内直接返回 204，导致已有数据无法读取。
- **约束**：**正常的超时必然等满整个窗口（例如 2.0s）。若非阻塞探测以外的读取在几毫秒内闪退返回“无数据”，立即判定连接异常，停止盲目重试。**

### 桩 6：修改主机配置前先确认管理开关
- **现象**：`create host` / `edit host` / `rm host` 依赖 `--mcp-manage-ssh-configs` 启动参数。开关未开时，所有写操作被统一拦截为 403 `forbidden`。
- **约束**：**遇到 403 提示 `start termcp with --mcp-manage-ssh-configs` 时，不要盲目重试或质疑语法，该实例当前处于只读模式。若需测试配置管理，须先启动带该开关的实例。**

### 桩 7：回复"半死"时，用两个入口二分定位
- **现象**：MCP 面返回异常（例如空回复、`[object Object]`、只有引导页），而命令其实执行了。
- **判据**：同一条 `cmd` 分别走两个入口——
  - MCP：`term(cmd="ls")`
  - HTTP：`POST /api/term` body `{"cmd":"ls"}`
  - **MCP 异常而 HTTP 正常 → 桥接层（MCP 客户端 / stdio 传输）；两侧都异常 → 内核（解析 / 执行）。**
- **约束**：**见到"半死"症状先二分，别先改代码。** 历史上正是靠这条把故障定位到桥接层，避免了在内核里找一个不存在的 bug。

---

需要安装 Termcp，或者了解它的更多资料，详见官网：[https://termcp.org](https://termcp.org)
