# 03 WQ 数据集目录工具

按地区、Delay 和股票池查询数据集与字段，管理本地缓存和范围导入。技术：Flask、SQLite、Pandas、原生 JavaScript。

本文只展示部分原始源码；省略导入、调用方和其余模块，不能作为完整程序直接运行。行号指提取时的原文件，哈希用于版本核对。

## 按范围更新使用画像

指定 scope 时，删除与替换以来源、地区和 Delay 为边界；插入和删除在同一连接事务内进行。省略了数据库初始化和其他查询模块。

原文件 `scripts/catalog_db.py`，第 407–495 行。

```python
    def import_alpha_usage(
        self,
        *,
        source_path: str | Path,
        dataset_records: list[dict[str, Any]],
        field_records: list[dict[str, Any]],
        details: dict[str, Any] | None = None,
        scope_keys: set[str] | None = None,
    ) -> dict[str, int]:
        imported_at = self.now()
        source = str(Path(source_path).resolve())
        with closing(self._connect()) as connection:
            with connection:
                if scope_keys:
                    scopes: list[tuple[str, int]] = []
                    for scope_key in sorted(scope_keys):
                        try:
                            region, delay_text = str(scope_key).rsplit("_", 1)
                            scopes.append((region.upper(), int(delay_text)))
                        except (TypeError, ValueError):
                            continue
                    for region, delay in scopes:
                        connection.execute(
                            "DELETE FROM alpha_dataset_usage "
                            "WHERE source_path = ? AND region = ? AND delay = ?",
                            (source, region, delay),
                        )
                        connection.execute(
                            "DELETE FROM alpha_field_usage "
                            "WHERE source_path = ? AND region = ? AND delay = ?",
                            (source, region, delay),
                        )
                else:
                    connection.execute("DELETE FROM alpha_dataset_usage WHERE source_path = ?", (source,))
                    connection.execute("DELETE FROM alpha_field_usage WHERE source_path = ?", (source,))
                for record in dataset_records:
                    connection.execute(
                        """
                        INSERT OR REPLACE INTO alpha_dataset_usage
                        (region, delay, universe, dataset_id, stats_json, source_path, imported_at)
                        VALUES (?, ?, ?, ?, ?, ?, ?)
                        """,
                        (
                            str(record["region"]).upper(),
                            int(record["delay"]),
                            str(record["universe"]),
                            str(record["dataset_id"]),
                            self._safe_json(record.get("stats") or {}),
                            source,
                            imported_at,
                        ),
                    )
                for record in field_records:
                    connection.execute(
                        """
                        INSERT OR REPLACE INTO alpha_field_usage
                        (region, delay, universe, dataset_id, field_id, stats_json, source_path, imported_at)
                        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                        """,
                        (
                            str(record["region"]).upper(),
                            int(record["delay"]),
                            str(record["universe"]),
                            str(record.get("dataset_id") or ""),
                            str(record["field_id"]),
                            self._safe_json(record.get("stats") or {}),
                            source,
                            imported_at,
                        ),
                    )
                connection.execute(
                    """
                    INSERT OR REPLACE INTO catalog_sources
                    (source_id, source_path, source_kind, file_size, imported_at,
                     status, details_json)
                    VALUES (?, ?, 'data_all_alpha_usage', ?, ?, 'imported', ?)
                    """,
                    (
                        self.query_key({"source_path": source, "kind": "data_all_alpha_usage"}),
                        source,
                        Path(source).stat().st_size if Path(source).exists() else None,
                        imported_at,
                        self._safe_json(details or {}),
                    ),
                )
        return {
            "dataset_usage_count": len(dataset_records),
            "field_usage_count": len(field_records),
        }
```
## 合成数据验证局部更新

测试先建立两个地区的记录，再只刷新一个地区，核对另一个地区仍保留。所有标识均是测试合成值。

原文件 `tests/test_catalog.py`，第 18–35 行。

```python
    def test_scope_import_preserves_other_scopes(self) -> None:
        with tempfile.TemporaryDirectory() as temp_dir:
            root = Path(temp_dir)
            db = CatalogDB(root / "catalog.sqlite3")
            records = [
                {"region": "USA", "delay": 1, "universe": "TOP3000", "dataset_id": "d", "stats": {}},
                {"region": "JPN", "delay": 1, "universe": "TOP1600", "dataset_id": "d", "stats": {}},
            ]
            source = root / "archive.pickle"
            db.import_alpha_usage(source_path=source, dataset_records=records, field_records=[])
            db.import_alpha_usage(
                source_path=source,
                dataset_records=[records[1]],
                field_records=[],
                scope_keys={"JPN_1"},
            )
            self.assertEqual(len(db._read_alpha_dataset_usage(region="USA")), 1)
            self.assertEqual(len(db._read_alpha_dataset_usage(region="JPN")), 1)
```
## 验证与边界

2026-10-15 对独立副本运行 3 项数据语义回归，全部通过且无跳过：范围更新保留、字段归属未知处理、缺失 OS 样本保留。本地首页与健康接口返回 HTTP 200。

验证使用临时 SQLite 和合成数据，未同步真实平台数据。该工具与回测工作台复用部分通用客户端；本项目展示重点是目录管理和导入语义。
