# Reference: UI 组件操作细节

各阶段页面控件的精确操作方法（均为实测验证）。

## DigitalPlat 注册页

- 国家下拉是原生 `<select>`：`bsk select @eN --value "CN"`（用两位码，用中文标签会报 option not found）。
- 注册成功判定：evaluate 读按钮/文案 `Registration completed`，页面跳回 `/auth/login`。
- 邮箱验证码弹窗：`[role=dialog]` 内 textbox 填 6 位码 + "验证代码"按钮。
- 免费名额注册弹窗：域名输入框已预填完整域名；委派模式选"外部名称服务器"，NS1/NS2 textbox 填占位值即可（后续会改）。

## Cloudflare 建 Token 页（React 自定义下拉）

页面结构：Account Resources 行两个 combobox（x≈212 Include / x≈346 值选择），Zone Resources 行三个（Include / scope / zone 值）。**aria 树的 @ref 坐标可能错位命中前一个下拉**，稳妥做法：

```js
// 枚举可见 combobox 并按坐标定位
const cs=[...document.querySelectorAll('[role=combobox]')].filter(x=>x.offsetParent);
// 派发完整事件序列打开下拉
const r=c.getBoundingClientRect();
const ev={bubbles:true,cancelable:true,clientX:r.x+r.width/2,clientY:r.y+r.height/2};
['pointerdown','mousedown','pointerup','mouseup','click'].forEach(t=>
  c.dispatchEvent(new (t.startsWith('pointer')?PointerEvent:MouseEvent)(t,ev)));
```

- 选项列表渲染在 body 末尾 portal：按精确 innerText 匹配（如 `Aistudent2078@gmail.com's Account`、`All zones`）后同样派发事件点击。
- 误开 Include/Exclude 菜单时 `bsk press Escape` 关闭。
- "Continue to summary" 无反应 = 上面有红色校验文案未消（`Choose an account resource` / `Choose a zone resource`）。
- 摘要页特征文本：`Permissions and options summary`。

## Cloudflare 站点/域名操作

- 添加站点：首页 "Add a domain" 卡片（`a[href]` 无 href，是 JS 路由到 `/add-site`）。
- 计划选择页直接是 Free/Pro 卡片 + "Select plan"（$0 那张）。
- Worker 自定义域名解绑：Domains tab → 行内 "More options"（aria-label）→ Remove → dialog 确认。
- 域名绑定冲突：同一 hostname 只能绑一个 Worker，先解绑旧的。

## BPB Panel 页面

- 面板控件多为原生 input + `getElementById` 读值，`el.value=...; dispatchEvent(new Event('input',{bubbles:true}))` 后真实 click Apply 即可保存；但**部分提交按钮 JS dispatch 无效**，一律用 `bsk click @ref`。
- Apply 成功无固定 toast，以下一次订阅内容变化为准。
- 订阅区按钮定位：
  ```js
  const h=[...document.querySelectorAll('h2')].find(x=>/Subscriptions/i.test(x.textContent));
  const bs=[...h.parentElement.querySelectorAll('button')].filter(x=>x.textContent.trim()==='content_copy');
  // 行顺序 = Normal, Fragment, Raw, Warp, Warp Pro（每行 qr/copy/download 三连）
  ```
- 订阅 YAML 是 JSON-flow 风格（`"name": "..."`），正则提取用 `"servername": "([^"]+)"` 等。

## Clash Party / mihomo-party 文件布局

```
%APPDATA%\mihomo-party\
├── profile.yaml          # 配置列表 items[] + current
├── profiles\<id>.yaml    # 每个配置的内容（remote 型=订阅原文缓存）
├── work\config.yaml      # 运行时合并配置（勿手改，重启重建）
├── work\cache.db         # store-selected 节点选择持久化（删掉=重置选择）
└── logs\core-*.log       # 内核拨号日志，排障首选
```

- 端口：7890 mixed / 7891 socks / 7892 http / 1053 DNS；GUI 与内核走 IPC，**无 HTTP controller**，切节点只能改 UI 或 cache。
- 查监听端口脚本（避开 bash 吞 `$_`/`$PSItem`）：
  ```powershell
  # ports.ps1
  Get-NetTCPConnection -State Listen -LocalAddress 127.0.0.1 | ForEach-Object {
    $p=(Get-Process -Id $_.OwningProcess).ProcessName
    if($p -match 'mihomo|Clash'){"{0}`t{1}" -f $_.LocalPort,$p} }
  ```

## GFW 阻断特征（判读用）

- `workers.dev`：DNS 污染（解析到 154.92.x.x 假 IP 超时）+ 真 CF IP + workers.dev SNI 也被 RST（curl exit 35）。→ 节点必须用自有域名 SNI。
- 自有域名走 CF：直连可达（面板/订阅 URL 直接 curl 即可验证，无需代理）。
- 诊断顺序：core 日志看 dial error → `--resolve 域名:443:CF_IP` 测真 IP+目标 SNI 握手 → 决定改 CDN 配置还是换节点。
