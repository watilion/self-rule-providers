# self-rule-providers

自维护的代理规则集仓库。监听上游 [Loyalsoldier/clash-rules](https://github.com/Loyalsoldier/clash-rules) 的规则变更，自动转换为 mihomo 的 `.mrs` 二进制格式，并连同仓库内的 `self-direct.txt` 一起发布到 GitHub Release。

## 规则来源

| 名称 | 来源 | 转换 |
|------|------|------|
| `Loyalsoldier-direct` | Loyalsoldier/clash-rules `direct.txt` | `mihomo convert-ruleset domain yaml` → `.mrs` |
| `Loyalsoldier-reject` | Loyalsoldier/clash-rules `reject.txt` | `mihomo convert-ruleset domain yaml` → `.mrs` |
| `self-direct` | 仓库内 `self-direct.txt` | 原样发布，runtime 自动编译 |

> 上游地址可在 `.github/upstreams.json` 中自由增减。

## 工作流

仓库根目录下的 `.github/workflows/sync-rules.yml` 会自动：

1. 每 6 小时（或 `self-direct.txt` push 时）拉取上游规则
2. 用 SHA256 对比上次缓存，发现变更才继续（节省资源）
3. 调用 `mihomo convert-ruleset <behavior> <format> <input> <output>` 转成 `.mrs`
4. 覆盖发布到名为 **`ruleset-latest`** 的 GitHub Release

### 为什么 self-direct.txt 不预先转 `.mrs`

`self-direct.txt` 是 classical yaml（含 `DOMAIN-SUFFIX,...` / `IP-CIDR,...` 等完整规则），但 mihomo `convert-ruleset` 的 `classical` 子命令存在一个跨版本 CLI bug：

```
panic: runtime error: invalid memory address or nil pointer dereference
github.com/metacubex/mihomo/rules/provider.(*classicalStrategy).payloadToRule
        .../classical_strategy.go:57
```

根因在 `rules/provider/mrs_converter.go:16`：`newStrategy(behavior, nil)` 给 classical 策略传了 nil parse 函数，到 `payloadToRule` 行57 `c.parse(...)` 就 panic。`domain` / `ipcidr` 不需要 parse 所以正常。

修复方案：`self-direct.txt` 原样发布，mihomo runtime 在加载远程 rule-provider 时**自动编译**成 MRS —— 这条路径用的 `ParseRule` 是真实注入的，没有 nil bug。

### 手动触发

在 Actions 页面 → "Sync Upstream Rules" → Run workflow。

### 添加更多上游

编辑 `.github/upstreams.json`，按现有格式追加一项即可：

```json
{
  "name": "Loyalsoldier-private",
  "url": "https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/private.txt",
  "behavior": "domain",
  "format": "yaml"
}
```

字段说明：

- `name`：输出文件名前缀（不含扩展名）
- `url`：上游原始规则 URL
- `behavior`：`domain` / `ipcidr` / `classical`（传给 mihomo 的 `convert-ruleset`）
- `format`：`text` / `yaml`（输入格式）
- `is_local`（可选，默认 `false`）：设为 `true` 时从仓库内本地路径读取（`url` 即为仓库内相对路径）

## Release 资产地址

发布后可用以下 URL 引用（`TAG` 固定为 `ruleset-latest`）：

```
https://github.com/watilion/self-rule-providers/releases/download/ruleset-latest/Loyalsoldier-direct.mrs
https://github.com/watilion/self-rule-providers/releases/download/ruleset-latest/Loyalsoldier-reject.mrs
https://github.com/watilion/self-rule-providers/releases/download/ruleset-latest/self-direct.txt
```

mihomo rule-provider 示例：

```yaml
rule-providers:
  # 上游 Loyalsoldier-direct 已转好的 mrs (classical 没问题)
  loyalsoldier-direct:
    type: http
    behavior: domain
    format: mrs
    url: "https://github.com/watilion/self-rule-providers/releases/download/ruleset-latest/Loyalsoldier-direct.mrs"
    path: ./ruleset/loyalsoldier-direct.mrs
    interval: 86400

  loyalsoldier-reject:
    type: http
    behavior: domain
    format: mrs
    url: "https://github.com/watilion/self-rule-providers/releases/download/ruleset-latest/Loyalsoldier-reject.mrs"
    path: ./ruleset/loyalsoldier-reject.mrs
    interval: 86400

  # self-direct 是 classical yaml,mihomo runtime 会自动编译为 mrs
  # 直接引用 main 分支的 .txt 文件
  self-direct:
    type: http
    behavior: classical
    format: yaml
    url: "https://raw.githubusercontent.com/watilion/self-rule-providers/main/self-direct.txt"
    path: ./ruleset/self-direct.mrs
    interval: 86400
```

## License

仅作为个人使用，规则源归原作者所有。