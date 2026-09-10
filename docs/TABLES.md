# HGDB 实施工作台 · 资料库数据表清单

> 更新于 2026-09-10。资料库空间为个人空间（默认「我的文档」）。
> **页面**：https://www.workbuddy.cn/space/d/EupwDcuS2LNPSqYY31XHIm
> **说明**：资料库数据表是类 Notion 多维表，**无数据库原生外键**，关联靠文本字段存父实体 ID + 页面侧按 ID 分组。

## 一、ID 体系（每个实体有自己的主键）

| 实体 | 主键字段 | 格式 | 外键字段 |
|---|---|---|---|
| 项目 | `项目ID` | `PJ01`、`PJ02` … | — |
| 服务器 | `服务器ID` | `SRV001`、`SRV002` … | `项目ID` → 项目 |
| 磁盘 | `硬盘ID` | `DSK001`、`DSK002` … | `服务器ID` → 服务器 |
| 数据库 | `数据库ID` | `DB001`、`DB002` … | `服务器ID` → 服务器 |
| 性能测试 | —（从属硬盘） | — | `硬盘ID` → 磁盘；`服务器ID`（冗余便于查询） |

关联链：

```
项目基本信息 (项目ID)
└── 服务器信息 (服务器ID, 项目ID↑)
    ├── 操作系统信息 (服务器ID↑, 1:1)
    ├── 磁盘信息 (硬盘ID, 服务器ID↑)
    │   └── 性能测试 (硬盘ID↑, 1:N)
    └── 数据库信息 (数据库ID, 服务器ID↑)
        ├── 数据库安装信息 (数据库ID↑, 1:1)
        ├── 数据库授权信息 (数据库ID↑, 1:1)
        ├── 数据库备份任务 (数据库ID↑, 1:N)
        └── 数据库参数信息 (数据库ID↑ + 参数名, 1:N 竖表)
```

> **为什么不用原始的资源序号**：台账里的 `seq`（如 `G1-S1-1`）**跨项目会重复**——`G0-S1-1` 在 7 个项目里都出现。
> 用它做关联键会导致子表数据串到别的主机。因此改为按序生成的全局唯一 ID（`SRV001`…），
> `资源序号` 保留为纯展示字段。

## 二、表清单

| # | 表名 | database_id | 字段数 | 主键 | 外键 |
|---|---|---|---|---|---|
| 1 | HGDB-项目基本信息 | `AtskUwWzdZAZcL0y8Q3sCm` | 12 | 项目ID | — |
| 2 | HGDB-服务器信息 | `cTctQmz3hElSq0OEhYbWCa` | 33 | 服务器ID | 项目ID |
| 3 | HGDB-操作系统信息 | `W36xhikrlQjc0y5aQ5TR7j` | 8 | —（1:1） | 服务器ID |
| 4 | HGDB-磁盘信息 | `pOhIyVtY7nSX7X7dYGOwjC` | 13 | 硬盘ID | 服务器ID |
| 5 | HGDB-数据库信息 | `KVKqPlzlz26QRgOJkJ2Blr` | 12 | 数据库ID | 服务器ID |
| 6 | HGDB-性能测试 | `JexUmVRR51nRy6jY5RoQT5` | 15 | —（1:N） | 硬盘ID、服务器ID |
| 7 | HGDB-数据库安装信息 | `Ezq6ewALCtqIBMgTbSfpsI` | 10 | —（1:1） | 数据库ID |
| 8 | HGDB-数据库授权信息 | `a2VxpzWdbFlyhLSnHv6tIQ` | 9 | —（1:1） | 数据库ID |
| 9 | HGDB-数据库备份任务 | `AO0Hp6d3hLWRlQg2vyCqPy` | 9 | —（1:N） | 数据库ID |
| 10 | HGDB-数据库参数信息 | `hGiTOk8jFEIoSSwzkVdC2h` | 7 | —（1:N 竖表） | 数据库ID + 参数名 |

## 三、实际记录数（2026-09-10 核验，与 `data/real.json` 一致）

| 表 | 记录数 | 表 | 记录数 |
|---|---|---|---|
| 项目基本信息 | 14 | 性能测试 | 10 |
| 服务器信息 | 31 | 数据库安装信息 | 31 |
| 操作系统信息 | 31 | 数据库授权信息 | 31 |
| 磁盘信息 | 30 | 数据库备份任务 | 17 |
| 数据库信息 | 31 | 数据库参数信息 | 104 |
| | | **合计** | **330** |

## 四、页面读写机制（index.html）

页面**对外仍保持扁平的 `state.hosts` 结构**，读写两侧各做一次翻译，现有渲染逻辑零改动：

- **读**：`assembleHosts(t)` 把 10 张表按 ID 组装回 `hosts[]`，同时把每条记录的 `_id` 挂到
  `h._rid = {project, server, os, db, install, license, backup, param:{参数名→_id}}`
  与 `h.disks[i] = {rid, diskId, perfRid, perf}`，供写回时定位。
- **写**：`saveHostToLibrary(h)` 顺序 upsert 各表——有记录 id 走 `updateRecord`（增量），
  没有走 `addRecord` 并把新 id 回填；子表按 diff 处理（参数竖表按参数名增改删、磁盘按硬盘ID增改删）。
- **删**：`deleteHostFromLibrary(h)` 级联删除该主机名下全部子表记录；该项目下已无其它服务器时连项目记录一起删。
- **ID 续号**：`nextId('SRV', state.ids.server, 3)` —— 从已加载数据里取同前缀最大序号 +1，新增时不撞号。
- **同步状态**：页头指示灯（资料库已同步 / 同步中… / 同步失败 / 本机数据）；写库失败时降级存本地并如实提示。

关键函数：`dbLoadAll`（翻页拉全量）、`assembleHosts`、`saveHostToLibrary`、`deleteHostFromLibrary`、
`upsert`、`delRecord`、`saveParams`、`saveDisks`、`dstr`（日期归一化）、`setSync` / `renderSync`。

## 五、维护脚本（backup/ 目录）

| 脚本 | 用途 |
|---|---|
| `schemas/01~10_*.json` | 10 张表的建表 schema |
| `reshape_tables.py` | 表结构重构：`verify`（看字段）/ `clear`（清空）/ `reshape`（删旧字段+加新字段）/ `add`（补字段） |
| `split_records.py` | 把 `data/real.json` 拆成 10 张表的记录，生成 `records/*.json` |
| `push_records.py` | 批量写入记录（每批 100 条，批间 sleep 1.2s） |
| `clear_table.py` | 清空指定表的全部记录 |
| `check.js` | 冒烟自检：JS 语法、DOM 元素存在性、关键函数、渲染函数调用环（铁律 9） |
| `test_roundtrip.js` | 读写 round-trip 测试：组装→拆分→再组装，比对字段有无漂移 |
| `upload_page.py` | 把 index.html 作为新版本提交到已托管 Page 节点 |
| `library_dump/` | 资料库全量记录快照（回滚用，非 git 跟踪） |

运行方式：
```bash
export LIB_TOKEN="<取票得到的 token>"
python backup/reshape_tables.py verify
python backup/split_records.py && python backup/push_records.py
node backup/check.js && node backup/test_roundtrip.js
```

## 六、已知坑

1. **批量写入是「一条不合格则整批拒绝」**：接口返回 `{"error":"records[N] 校验后为空..."}`，
   一条空记录会导致整批 0 条写入。生成记录时必须用 `put()` 过滤 `None`，并剔除全空记录。
2. **空值语义**：原始数据里 `-` 表示「未登记」，写入前统一当空值跳过，
   否则 select 字段会因选项不存在而拒绝。
3. **date 字段类型不对称**：写入用 `"YYYY-MM-DD"`，**读出来是 ISO 时间戳** `"...T00:00:00Z"`，
   页面侧必须归一化（`dstr()`），直接 `.slice()` 会抛 TypeError。
4. **select 字段的值必须在预置选项内**：用户在表单里填了选项外的值会写入失败，
   页面会把后端错误如实提示出来。
5. **`upsert`/`delRecord` 的 databaseId 必须写成 `DATABASE_ID` 常量形式**，
   否则过不了 `lint_database_sdk_usage.py` 的 DSDK002。删磁盘时不要图省事复用性能表的常量，
   会删错表。
6. **页面更新走编辑事务**：`create_page_transaction → get_page_upload_url → PUT → commit`。
   commit 的 stdout 可能有多行，解析失败不代表未提交——用 `create_page_transaction` 看 `baseVersion` 是否自增来确认。
