# 05 可撤销文件整理器

对选定的 Windows 本地目录生成整理计划，执行非覆盖移动，保留记录，并支持检查后撤销与中断恢复。技术：Python 标准库、SQLite、Tkinter、Windows 文件系统接口。

作者说明：本项目代码由本人（aa6655）独立完成。

本文只展示部分原始源码；省略导入、装饰器、调用方和其余模块，不能作为完整程序直接运行。行号指提取时的原文件，哈希用于版本核对。

## 撤销结果的完成判据

撤销受阻、需要人工复核或发生错误时，不能把批次标为全部完成。调用方据此显示可重试或待处理状态。

原文件 `organizer/engine.py`，第 43–57 行。

```python
class UndoResult:
    batch_id: int
    undone: list = field(default_factory=list)
    blocked: list = field(default_factory=list)
    needs_review: list = field(default_factory=list)
    errors: list = field(default_factory=list)

    @property
    def ok(self):
        """撤销是否全部完成：仍有 blocked/needs_review/errors 即未完成。

        R4：撤销受阻（原位置被占、目标被改、目标丢失）时 blocked 非空，
        不得返回"全部成功"信号；调用方据此呈现未完成状态并可重试。
        """
        return not self.blocked and not self.needs_review and not self.errors
```
## 文件身份的三种状态

区分身份一致、确认不同和无法取得身份。无法取得身份不等于证明对象被替换；降级后的内容校验由后续调用方负责。

原文件 `organizer/engine.py`，第 60–87 行。

```python
def _identity_status(path, item):
    """核对文件身份，返回 (kind, note)。

    kind 三态：
      "ok"          身份可用且与计划一致；
      "mismatch"    双方都取得稳定 Windows 身份且明确不同（必须阻断）；
      "unavailable" 身份不可取得（不阻断，调用方记录退化说明）。
    只有"可取得稳定身份且明确不同"才允许判为 mismatch；计划或当前
    任意一侧无法取得 win: 身份时一律 unavailable，不得把 fallback
    当作"已经证明不同"（R5）。
    """
    if not item.file_id or not item.file_id.startswith("win:"):
        return ("unavailable", "计划未保存稳定 Windows 文件身份，"
                                "本次仅做内容级（大小/时间/哈希）校验")
    try:
        current = fs.get_file_id(path)
    except Exception as e:
        return ("unavailable", f"无法重新取得文件身份: {e}，"
                               f"本次仅做内容级校验")
    if not current or not current.startswith("win:"):
        return ("unavailable",
                f"当前无法取得稳定 Windows 身份（返回 {current or '空'}），"
                f"本次仅做内容级校验，未证明与计划是不同文件对象")
    if current != item.file_id:
        return ("mismatch",
                f"文件身份不一致（计划 {item.file_id}，"
                f"当前 {current}），不是计划时的同一文件对象")
    return ("ok", "")
```
## 固定计划时的目录边界

比较当前真实落点与计划时保存的根，避免目标根被 junction 替换后把新的解析位置当作原授权范围。

原文件 `organizer/windows_fs.py`，第 204–234 行。

```python
def ensure_within_planned(path, root, root_real=None):
    """以计划时的真实根为边界校验 path 实际落点（R1）。

    root_real 是计划时记录的真实根（resolve_real_path(root) 的结果）。
    - 若 root_real 有值：path 的真实位置必须位于该计划根内；
      “当前 root 变成 junction”不会改变计划边界，由此拦截根替换。
    - 若 root_real 为空（旧批次缺边界证据）：采用保守策略——
      当前 root 若为重解析点则拒绝（无法确认是计划所选范围），
      否则按当前真实位置校验。
    """
    if not is_inside(path, root):
        raise FileSystemError(
            f"路径 {path} 逃逸出目标根目录 {root}，已拒绝")
    if root_real:
        planned_root = Path(root_real)
        real = resolve_real_path(path)
        if not is_inside(real, planned_root):
            raise FileSystemError(
                f"路径 {path} 的真实位置 {real} 不在计划时的目标根 "
                f"{planned_root} 内（根可能在计划后被替换）")
    else:
        # 旧库无边界证据：不允许把重解析后的当前解析位置当作原确认边界
        if is_symlink_or_junction(root):
            raise FileSystemError(
                f"目标根 {root} 在当前为重解析点，且计划未保存真实根"
                f"边界证据，拒绝移动")
        real = resolve_real_path(path)
        real_root = resolve_real_path(root)
        if not is_inside(real, real_root):
            raise FileSystemError(
                f"路径 {path} 经重解析点解析后位于目标根目录之外: {real}")
```
## 验证与边界

2026-10-15 对所选完整源码副本运行现有 92 项测试，全部通过且无跳过；独立 CLI 验收在临时目录创建合成文件，完成计划、执行、文件位置与哈希核对、撤销回源，结果为 PASS。源码版本对应 `de676556362ef72318ace14adc53286e7ebddb5e`。

该结果适用于完整本地项目，当前节选不包含完整测试集。首版只处理第一层普通文件和同卷移动，未实现递归、跨盘和云同步占位识别；程序化 GUI 验证不等同于人工逐屏验收。
