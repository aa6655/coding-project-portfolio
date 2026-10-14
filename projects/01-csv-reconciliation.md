# 01 表格对账工作台

把两份 CSV 转成可逐项核对的差异报告，保留输入哈希、行位置和数值规则。技术：Python、Decimal、Tkinter、JSON/HTML。

本文只展示部分原始源码；省略导入、调用方和其余模块，不能作为完整程序直接运行。行号指提取时的原文件，哈希用于版本核对。

## 有限数字与整数末尾零

空值与无效数字分别处理；数字展示只去掉小数尾零，避免把整数 100 显示成 1。

原文件 `reconcile/compare.py`，第 56–67 行。

```python
def _parse_number(value):
    """返回 (ok, Decimal|None, 说明)。空值视为特殊：ok=True, dec=None, 'empty'。"""
    s = str(value).strip()
    if s == "":
        return True, None, "empty"
    try:
        d = Decimal(s)
    except InvalidOperation:
        return False, None, f"无法解析为数字: {value!r}"
    if not d.is_finite():
        return False, None, f"非有限数字: {value!r}"
    return True, d, None
```

原文件 `reconcile/compare.py`，第 70–81 行。

```python
def _trim_decimal_zeroes(d):
    """Decimal 的展示文本：只去掉小数部分尾随零，绝不删除整数末尾有效零。

    只有小数部分可以去除尾零；例如 10.50 显示为 10.5，
    整数 100 和容差 10 必须保留各自的有效零。
    """
    s = format(d, "f")
    if "." in s:
        s = s.rstrip("0").rstrip(".")
        if s in ("", "-"):
            s = "0"
    return s
```
## 识别指向同一文件的路径

输出保护需要考虑规范路径、文件身份和实际解析路径。仅比较字符串不足以识别硬链接等别名。

原文件 `reconcile/report.py`，第 37–65 行。

```python
def same_target_as_any(path, target_paths):
    """判断路径是否与 target_paths 中的任一文件指向同一文件。

    三层校验（2026-10-13 核查 T1 修复基础上把核心抽为独立函数，供批次
    跨组输入冲突检查复用）：
    1. normcase(abspath) 字符串相等：覆盖大小写别名（Windows 上
       INPUT.CSV 与 input.csv 同文件）与路径规范化差异；
    2. os.path.samefile：覆盖同 inode 的所有别名——硬链接、指向
       目标的符号链接、junction/重解析点（文件已存在或路径是链接
       时按真实文件身份比较）；
    3. normcase(realpath) 相等：覆盖悬空符号链接（os.path.exists
       为 False 但 open('w') 仍会跟随链接创建目标）与多层链接。

    返回命中的目标（target_paths 中的原元素）或 None。
    """
    ap = os.path.abspath(path)
    for target in target_paths:
        tp = os.path.abspath(target)
        if os.path.normcase(ap) == os.path.normcase(tp):
            return target
        if os.path.exists(ap) or os.path.islink(ap):
            try:
                if os.path.samefile(ap, tp):
                    return target
            except OSError:
                pass  # 无法 stat（如悬空链接），交给 realpath 层
        if os.path.normcase(os.path.realpath(ap)) == os.path.normcase(os.path.realpath(tp)):
            return target
    return None
```
## 验证与边界

2026-10-15 对所选完整源码副本运行现有测试：93 项通过，0 跳过；单组与两组批次合成样例通过，6 个输入 CSV 哈希不变。该结果适用于完整本地项目，当前节选不包含完整测试集。

覆盖 CSV、精确匹配与绝对容差；未实测十万行规模，批次写出失败恢复仍需完善。没有真实客户记录或生产收益证明。
