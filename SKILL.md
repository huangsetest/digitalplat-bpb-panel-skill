---
name: digitalplat-bpb-panel
description: 全流程部署自建代理面板：DigitalPlat 免费域名注册 → 托管到 Cloudflare → 部署 BPB Panel (Worker) → 域名绑定 → 面板登录取 Clash 订阅 → 导入 Clash Party (mihomo-party)。当用户提到"注册免费域名"、"bpb/bpb面板"、"BPB Wizard"、"域名托管到Cloudflare"、"部署worker面板"、"导入clash订阅"、或要求从零搭建自建代理时使用。全程通过 bsk 接管 Chrome 完成。
---

# DigitalPlat 免费域名 + Cloudflare + BPB Panel 全流程部署

五阶段流水线，每阶段有验收条件，达成即进入下一阶段。全程用 bsk 接管浏览器实例（见 browser-skill），本机代理 `socks5/http 127.0.0.1:10808`（curl 访问境外站点必加 `--socks5-hostname 127.0.0.1:10808`）。

## 环境速查（本机已验证值）

| 项 | 值 |
|---|---|
| Chrome 实例 | bsk `81caf267`（ID 每次重连会变，用 `bsk browsers --json` 确认） |
| 浏览器 profile | Profile 5 = aistudent2078@gmail.com（Gmail u/0） |
| CF 账户 ID | `b05849133a9c9971743a3e1549e2c78b`（Aistudent2078 账户） |
| 已注册域名 | `aistudentfeature.dpdns.org`（永久免费，已托管 CF） |
| 面板 Worker | `drt4u8no085opqpg9ui6jo4y4dn`，面板 `https://<域名>/<密钥路径>/panel` |
| DigitalPlat 账号 | aistudent2078@gmail.com / Dp2026aistudent! |
| BPB 面板账号 | aistudent2078@gmail.com / Huangse007 |
| Clash Party | `C:\Program Files\Clash Party\Clash Party.exe`，数据目录 `%APPDATA%\mihomo-party` |

## 阶段 1：DigitalPlat 注册免费域名

1. `bsk session start --browser <id>`，导航 `https://dashboard.digitalplat.org/auth/login` → "创建账户"。
2. 填表：Username、邮箱、姓名、电话（`+86-号码` 格式）、密码 + WHOIS 地址（Address/City/State/Postal/国家 select 用 **两位国家码** 如 `CN`）。
3. Turnstile 勾选框：常需点第二次才过，screenshot 核对"成功！"。
4. 提交后弹邮箱验证码 → 到 Gmail 收件箱搜发件人取 6 位码填入。
5. 登录后进控制台 → 注册域名 → "DigitalPlat 域名" → 填名称 + 选后缀 → 勾条款 → 检查可用性（**结果只在 toast 几秒**，用 `[data-sonner-toast]` 读）。
6. 可用后缀现状：`.dpdns.org`（免费可选）、`.us.kg`/`.xx.kg`（暂停）、`.qzz.io`（吃付费名额）。
7. 提交注册 → 若要求 KYC：跳 `/user/kyc`，选 `github` 值 → Verify with GitHub → 需要用户手动登录 GitHub 时用 `bsk request-help`。
8. 注册弹窗中 NS1/NS2 可先填 `ns1/ns2.digitalplat.net`（阶段 2 会换掉）。

**验收**：域名列表出现该域名，状态 ok。

## 阶段 2：托管到 Cloudflare

1. CF dashboard 首页 → **"Add a domain"** → Connect a domain（注意：`/customers/add/site` 路由 404 不存在）。
2. 输入完整域名 → Continue → 选 **Free** 计划 → Continue to activation。
3. DNS 记录 0 条时确认继续即可。
4. 记下分配的 2 个 NS（如 `addyson.ns.cloudflare.com` / `roan.ns.cloudflare.com`）。
5. 回 DigitalPlat 域名管理页 → 名称服务器 tab → 把 NS1/NS2 改为 CF 的 → 点"更新名称服务器"。
   - 该按钮常被浮动"询问 Digi AI"挡住导致误点，**用 DOM 定位按钮 click()** 可靠。
6. 验证：`curl --socks5-hostname 127.0.0.1:10808 "https://dns.google/resolve?name=<域名>&type=NS"` 看到 CF NS 即委派生效；CF 侧自动转 Active（"Your domain is now protected by Cloudflare"）。

**验收**：DoH 查到 CF NS + CF 站点 Active。

## 阶段 3：部署 BPB Panel

1. 打开 `https://wizard.bpb-panel.workers.dev/`（BPB Wizard）。它需要一个 CF API Token。
2. **创建 token**（dashboard → profile/api-tokens → Create Token → Use template "Edit Cloudflare Workers"）：
   - **必须** Account Resources 选具体账户（禁用 All accounts）、Zone Resources 选 Specific zone + 具体域名（用户明令：禁 All，且**不要填面板的 Custom Domain**，会冲突）。
   - 表单校验不过时 "Continue to summary" 会**静默无反应**——检查 "Choose an account/zone resource" 红字。
   - 下拉框是自定义组件：用 `[role=combobox]` 枚举 + getBoundingClientRect 中心点派发 pointer 事件序列；选项渲染在 portal 里。
   - Apply/Create 类提交按钮必须 **bsk 真实 click**（JS dispatch 不触发提交）。
   - 令牌值只在创建成功页出现一次：用正则 `/cfut_[A-Za-z0-9_-]+/` 从叶子 DOM 节点抓，然后 `curl -H "Authorization: Bearer <tok>" https://api.cloudflare.com/client/v4/user/tokens/verify` 验证。
3. 向导页填 token → alt_route 保持 `workers` → Install。日志逐行 ✓ 直到 "Panel URL: https://<worker子域>.workers.dev/<密钥>/panel"。
4. （可选）把面板绑到自有域名：先在旧 Worker 的 Domains tab 解绑域名（More options → Remove），再到新面板 Worker 的 Domains tab → Add Domain → 选 zone → 子域留空 → Add domain。
   - `/workers/domains` API 对 token 报 10405，只能走控制台 UI。
   - 新绑域名证书签发需 ~30s，期间 curl TLS 失败属正常。

**验收**：`https://<域名>/<密钥>/panel` 返回 200 HTML。

## 阶段 4：面板初始化 + 取 Clash 订阅

1. 打开面板 URL。首次会弹 **Set Password**（Cloudflare Email + New Password），填用户给的账号后自动跳登录页，再登录。
2. 展开 VLESS-Trojan 等默认配置 → 真实 click "Apply"（按钮文本含 check_circle 图标，innerText 是 "check_circle Apply"）。
3. **关键**：默认生成的节点 server/SNI 全是 `*.workers.dev`，本机 GFW 对该 SNI 直接 RST（连真实 CF IP 也不行）。修复：Common 区 **Custom CDN** 三项 `customCdnAddrs` / `customCdnHost` / `customCdnSni` 全部填自有域名 → 再 Apply。
   - 勿动 `customDomain` 字段（用户明确要求）。
4. 取订阅链接：订阅区每行（Normal/Fragment/Raw/Warp/Warp Pro）有 qr_code/content_copy/download 三个无名按钮。自动化下剪贴板不可用，用**劫持法**：
   ```js
   window.__cap=[]; navigator.clipboard.writeText=t=>{window.__cap.push(t);return Promise.resolve()};
   // 点击订阅区第 N 个 content_copy 按钮后读 window.__cap
   ```
   Clash 链接形如 `https://<域名>/<密钥>/sub/normal?app=clash`。
5. 验证：`curl --socks5-hostname 127.0.0.1:10808 "<订阅URL>"` 返回 YAML，且 `"servername"` 含自有域名。

**验收**：订阅 YAML 中节点 server/SNI/Host 指向自有域名或 CF IP+自有 SNI。

## 阶段 5：导入 Clash Party（mihomo-party）

数据目录 `%APPDATA%\mihomo-party`：
1. 备份 `profile.yaml` 到工作区。
2. 生成 11 位 id（如 `b7f3a91c25d`），把订阅 YAML 存为 `profiles/<id>.yaml`。
3. `profile.yaml` 的 items 追加：
   ```yaml
   - id: <id>
     name: BPB Clash
     type: remote
     url: "<订阅URL>"
     substore: false
     interval: 1440
     override: []
     useProxy: false
     allowFixedInterval: false
     autoUpdate: true
     updated: <毫秒时间戳>
   ```
   并把 `current:` 改为 `<id>`。
4. `taskkill /F /IM "Clash Party.exe" /T` → 后台启动 `"C:\Program Files\Clash Party\Clash Party.exe" &`。
5. 验证：`curl -x http://127.0.0.1:7890 https://www.gstatic.com/generate_204` 返回 **204**。
   - 若 000：看 `%APPDATA%\mihomo-party\logs\core-*.log` 的 dial error；多半是节点 SNI 还是 workers.dev（回阶段 4 修 Custom CDN），或选中了死节点（删 `work/cache.db` 后重启让其重选）。

**验收**：204 测试通过，出口 IP 变化。

## 忘记面板入口路径的找回方法

面板密钥路径（`/<16位>/panel`）不直接存 KV（KV 只有 secretKey/pwd，路径带盐派生），找回按优先级：

1. **CF 控制台日志**（本 worker 已启用 Observability）：Workers & Pages → 面板 worker（随机名如 `drt4u8no...`）→ Observability → Logs，最近访问显示完整 URL `GET https://<域名>/<路径>/panel`。新部署时记得在 Settings→Observability→Logs 打开开关（Include Invocation logs + Persist to dashboard，Free 计划 200k/天），否则查不到历史。
2. **Clash Party 配置**：`%APPDATA%\mihomo-party\profile.yaml` 中 BPB Clash 项 `url:` 里 `/sub/normal` 前那段就是路径。
3. **BPB Wizard private link**：安装成功页复制过的一键链接。
4. 都没有 → 重跑 Wizard 覆盖安装（生成新路径，旧订阅作废需重导）。

## 通用坑（bsk / 页面自动化）

- aria 树 checkbox 状态与实际相反，关键开关 screenshot 核对。
- 导航后 @ref 全部失效，必须重新 snapshot。
- 内联 PowerShell 的 `$_` 会被 bash 吞，复杂脚本写 `.ps1` 再 `powershell -File` 跑。
- Gmail 搜索用 hash URL 导航会 30s 超时但实际成功，evaluate 读 `document.title` 确认。

详细 UI 组件操作细节见 [reference.md](reference.md)。
