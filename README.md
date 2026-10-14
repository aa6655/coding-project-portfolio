# Coding 项目与源码分析节选

五项技术案例：四个本地应用项目节选，以及一个注明原作者的开源源码分析案例。每项保留少量技术代码和说明，便于核验实现思路、边界处理与验证范围。

| 项目 | 展示内容 | 验证依据 |
| --- | --- | --- |
| [表格对账工作台](projects/01-csv-reconciliation.md) | Decimal 数字处理、输入文件别名保护 | 完整本地副本 93 项测试通过；合成示例通过 |
| [WQ 回测工作台](projects/02-wq-backtest-workbench.md) | 状态落盘、跨进程修改、暂停与恢复 | 离线 33 项测试通过；本地 HTTP 检查通过 |
| [WQ 数据集目录工具](projects/03-wq-dataset-catalog.md) | 按范围导入、事务更新、数据语义回归 | 3 项合成数据回归通过；本地 HTTP 检查通过 |
| [WQ Alpha 本地管理工具](projects/04-wq-alpha-manager.md) | 后端事件流、前端视图状态保留 | Python 语法、TypeScript 检查、Vite 构建通过 |
| [ExplorerPatcher 源码分析](projects/05-explorerpatcher-source-review.md) | 资源处理返回语义、语言列表匹配 | 上游固定提交逐行核对；不作为个人开发贡献证明 |

## 阅读说明

代码以 Markdown 中的短节选展示，省略其余模块和依赖，不能作为完整程序运行。测试结果来自对应本地完整项目副本，验证日期为 2026-10-15；测试通过不代表生产上线或资质审核通过。

每段提供相对文件名、原始行号与 [哈希清单](evidence/excerpt-manifest.json)。个人承担的职责与开发归属应按实际贡献说明；本仓库不声明全部代码从零原创。ExplorerPatcher 单独标明第三方作者、版本与 GPL-2.0 许可。

## 公开范围

只发布说明、节选和核验摘要。账户配置、密钥、会话、真实数据库、Alpha/PnL 历史、安装包、个人路径和完整源码均不在本仓库。详见 [公开范围](SCOPE.md) 与 [检查记录](evidence/review.md)。

开发和整理使用 AI 辅助；申请人可结合实际负责的需求、设计、实现、审查和验收工作补充个人说明。
