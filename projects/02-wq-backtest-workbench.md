# 02 WQ 回测工作台

围绕回测任务的本地 Web 工作台，包含任务调度、结果记录与暂停恢复。技术：Flask、SQLite、Python、原生 JavaScript。

本文只展示部分原始源码；省略导入、调用方和其余模块，不能作为完整程序直接运行。行号指提取时的原文件，哈希用于版本核对。

## 状态落盘与跨进程修改

状态写入临时文件后替换目标文件；状态修改在跨进程文件锁下进行。

原文件 `engine/consultant_backtester/services/task_control.py`，第 16–23 行。

```python
def atomic_json(path: Path, payload: dict) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    temp = path.with_name(path.name + "." + uuid.uuid4().hex + ".tmp")
    with temp.open("w", encoding="utf-8") as handle:
        json.dump(payload, handle, ensure_ascii=False, indent=2)
        handle.flush()
        os.fsync(handle.fileno())
    os.replace(temp, path)
```

原文件 `engine/consultant_backtester/services/task_control.py`，第 37–45 行。

```python
    def mutate(self, update):
        with InterProcessFileLock(str(self.path) + ".lock"):
            value = self.read()
            if value.get("version") != CONTROL_VERSION:
                raise ValueError("task runner does not support cooperative control")
            result = update(value)
            value["updated_at"] = dt.datetime.now(dt.timezone.utc).isoformat()
            atomic_json(self.path, value)
            return result if result is not None else value
```
## 暂停、排空与恢复判据

暂停先进入 draining；恢复要求没有未确认的在途提交。完成判定同时检查在途项、失败项和结果数量。

原文件 `engine/consultant_backtester/services/task_control.py`，第 51–58 行。

```python
    def request_pause(self):
        def update(value):
            if value["state"] in {"paused", "draining"}:
                return
            if value["state"] not in {"running", "starting"}:
                raise ValueError("only an active controlled task can be paused")
            value.update(desired_pause=True, state="draining")
        return self.mutate(update)
```

原文件 `engine/consultant_backtester/services/task_control.py`，第 60–65 行。

```python
    def prepare_resume(self):
        def update(value):
            if value["state"] != "paused" or value.get("in_flight"):
                raise ValueError("resume requires a paused task with no unknown in-flight submissions")
            value.update(desired_pause=False, state="starting", resume_count=value.get("resume_count", 0) + 1)
        return self.mutate(update)
```

原文件 `engine/consultant_backtester/services/task_control.py`，第 99–112 行。

```python
    def finish(self, expected):
        def update(value):
            if value["in_flight"]:
                value["state"] = "interrupted"
                value["resume_blocked_reason"] = "存在状态未知的在途提交；请先核实平台结果，禁止自动重发。"
            elif value.get("desired_pause"):
                value["state"] = "paused"
            elif any(str(row.get("status", "")).upper() not in SUCCESS for row in value["results"].values()):
                value["state"] = "failed"
            elif len(value["results"]) < expected:
                value["state"] = "interrupted"
            else:
                value["state"] = "completed"
        return self.mutate(update)
```
## 验证与边界

2026-10-15 对公开候选副本运行现有 33 项测试，全部通过且无跳过；本地页面与版本接口返回 HTTP 200。验证时阻断外部网络请求，未触发真实 BRAIN 回测、账号登录或提交。

项目复用了本地回测引擎和通用客户端，不能据此称所有底层模块均为从零原创。当前节选只用于审阅任务控制机制，不提供仿真接口或账户配置。
