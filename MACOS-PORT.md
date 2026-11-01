# macOS 兼容移植分支 / macOS Port Branch

> 派生自 [T-Auto/dsh-ops](https://github.com/T-Auto/dsh-ops)，提交
> `2c0b0397ce4b48436ff76cf5c5b039054240a45c`，版本 `dsh-ops@0.5.0-alpha3`。
>
> 本分支记录一次在 **macOS（Apple Silicon）+ DSH 0.2.1-rc.3** 上的可运行性验证，
> 保留最小改动集。**不用于日常 profile。**

完整说明：[docs/MACOS-PORT.md](./docs/MACOS-PORT.md)

---

## 改了什么

两个文件，无源码改动。

| 文件 | 改动 |
| --- | --- |
| `package.json` | 删除 `dsh.client` 段 |
| `cordis.patch.yml` | 首轮保守配置：只开 Standard 常驻文件工具，`hideOfficialTools: false` |

## 为什么

`dsh.client.inject` 与宿主 `@deepseek-ai/dsh-client-modules` 期望的
`dsh.client.external` 形状不符；且宿主 562 个安装包中不含该段引用的
`@deepseek-ai/dsh-client-ui-primitives` 与 `dsh-client-ui-slots`。
删除后浏览器半边不加载，Host 工具不受影响。

`engines.dsh` 声明的 `>=0.2.0-rc.2 <0.2.2-0` 覆盖宿主 `0.2.1-rc.3`，
源码无需改动。

## 结果

- 构建通过：222 个 TS 文件 → `lib/`；Rust worker `4,260,336 B`，
  `sha256 ee4609db…`
- 5 个 `ops_*` 工具在模型中正常发布并可调用
- 降级实测：worker 缺失时官方 `read` 自动回退
  （`src/publication/replacements.ts` 的 `plan()` 只在 OPS 对应工具可见时才隐藏官方工具）
- 测试：`162` 套件中 3 个真实失败，全部为浅克隆/平台沙箱/主动删 `dsh.client` 之举，
  无一项指向宿主 API 不兼容或运行时崩溃

## Token 实测（结论：不省）

| 项 | 官方 | OPS |
| --- | --- | --- |
| 5 个工具 schema | 835 | 1248 (**+49%**) |
| `grep` 全目录 | 2939 | 3045 (−3.6%) |
| `glob` 全部 `.ts` | 3043 | 1600 (+47.4%) |
| `read` 单文件 | 2332 | 2505 (−7.4%) |

净账在 **3 次调用处盈亏平衡**，超过即亏损：

```
轮次  1      2      3       5       10
净差  +8.2%  +3.4%  -0.7%   -7.2%   -17.8%
```

## 许可证

与上游一致，**MIT**。见 [LICENSE](./LICENSE)、[NOTICE](./NOTICE)。
