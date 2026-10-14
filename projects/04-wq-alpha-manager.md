# 04 WQ Alpha 本地管理工具

提供 Alpha 查询、详情、指标展示和任务日志的前后端应用。当前技术：React、TypeScript、Vite、FastAPI、PostgreSQL。

本文只展示部分原始源码；省略导入、调用方和其余模块，不能作为完整程序直接运行。行号指提取时的原文件，哈希用于版本核对。

## 将子进程输出转为事件流

后端逐行读取任务输出，将日志、退出码与异常转换为事件流。这里展示通用任务输出机制，不含实际命令参数、账户信息或业务结果。

原文件 `backend/main.py`，第 102–141 行。

```python
def _stream_process(
    cmd: list[str],
    *,
    stdin_text: str | None = None,
    lock: threading.Lock | None = None,
) -> Iterable[str]:
    env = dict(os.environ)
    env["PYTHONIOENCODING"] = "utf-8"
    env["PYTHONUNBUFFERED"] = "1"
    try:
        process = subprocess.Popen(
            cmd,
            stdin=subprocess.PIPE if stdin_text is not None else None,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            encoding="utf-8",
            errors="replace",
            cwd=str(PROJECT_DIR),
            env=env,
        )
        if stdin_text is not None and process.stdin is not None:
            process.stdin.write(stdin_text)
            process.stdin.flush()
            process.stdin.close()
        if process.stdout is not None:
            for line in process.stdout:
                line = line.rstrip()
                if line:
                    yield f"data: {json.dumps({'log': line}, ensure_ascii=False)}\n\n"
        process.wait()
        yield f"data: {json.dumps({'done': True, 'exit_code': process.returncode}, ensure_ascii=False)}\n\n"
    except Exception as exc:
        yield f"data: {json.dumps({'error': str(exc)}, ensure_ascii=False)}\n\n"
    finally:
        if lock is not None and lock.locked():
            try:
                lock.release()
            except RuntimeError:
                pass
```
## 切换视图时保留页面状态

视图保持挂载，通过显示状态切换，以保留任务日志和组件状态。该段是组件内部节选。

原文件 `frontend/src/App.tsx`，第 81–86 行。

```tsx
            {/* All views always mounted — only active one visible via display toggle */}
            {allViews.map(({ key, el }) => (
              <div key={key} style={{ flex: 1, minWidth: 0, overflow: 'hidden', display: view === key ? '' : 'none' }}>
                {el}
              </div>
            ))}
```
## 验证与边界

2026-10-15：11 个 Python 文件语法解析通过，TypeScript 构建检查通过，Vite 前端构建通过。构建出现较大 bundle 提示。未发现现成单元测试目录；本次未启动后端、连接 PostgreSQL 或执行平台操作，不能称端到端验证通过。

本次依据代码确认使用 PostgreSQL；原旧说明中的 SQLite 描述不沿用。
