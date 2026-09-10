# 数据字段说明

`data/sample.json` 与 `data/real.json` 结构完全一致，顶层：

```json
{ "meta": { "exportedAt": "YYYY-MM-DD", "source": "..." },
  "hosts": [ { ...一台主机... } ] }
```

## 字段对照表（原始 Excel 列名 → JSON key）

### 基本信息

| Excel 列 | JSON key | 说明 |
|---|---|---|
| 项目 | `project` | 项目编码串，如 `hg45-省交易中心-浪潮云-王凯N018779`，是分组主键 |
| 资源序号 | `seq` | 如 `G1-S1-1`、`zzqmjk01` |
| EU | `eu` | 最终用户 |
| ISV | `isv` | 软件开发商 |
| 用途 | `purpose` | 业务系统名称 |
| 部署架构 | `arch` | 单机 / 流复制 / 高可用 / 高可用+读写分离 / DCS |
| 标识码 | `code` | 集群标识，如 `sdjyzx`、`hgdb01` |
| 主备 | `role` | 主 / 备 / 主（暂时）/ 备（暂时）/ - |
| 安装时间 | `installDate` | `YYYY-MM-DD` |
| DBID | `dbid` | 数据库实例 ID |

### 厂商与网络

| Excel 列 | JSON key | 说明 |
|---|---|---|
| 服务器厂商 | `vendor` | 鲲鹏 / 海光 等 |
| 云服务厂商 | `cloud` | 浪潮云 / 移动云 / 华为云 / 中电云 等 |
| 网卡 | `nic` | `eth0` / `ens3` / `enp3s0` |
| IP | `ip` | 含掩码，如 `10.246.26.82/24` |
| MAC | `mac` | — |
| FIP | `fip` | 浮动 IP |
| VIP | `vip` | 高可用虚拟 IP |

### CPU / 内存

| JSON key | 说明 |
|---|---|
| `cpuModel` | `Kunpeng-920` / `Hygon Dhyana Processor` / `Hygon C86-4G` |
| `cpuArch` | `aarch64` / `x86_64` |
| `cpuLogic` `threadsPerCore` `coresPerSocket` `sockets` `numa` | 拓扑 |
| `freqMax` | 最大频率 MHz |
| `l1d` `l1i` `l2` `l3` | 缓存 |
| `virt` | 虚拟化，如 `KVM / full`、`AMD-V / KVM / full` |
| `memTotal` `memAvail` `swap` | 单位 GB，数字类型 |

### 系统与数据库

| JSON key | 说明 |
|---|---|
| `os` | 操作系统版本（已压缩为单行） |
| `hostname` `osUser` `osPass` `locale` | **口令字段受脱敏开关控制** |
| `dbVersion` | `hgdb-45103` / `hgdb-45X3` / `hgdb-4585` |
| `licenseExpiry` | **授权到期日，驱动预警模块，务必填** |
| `plugins` | 扩展插件，如 `PostGIS 340` |
| `ha` | `hghac-427` / `stream` / `single` |
| `scope` `rwSplit` `rwPort` | 高可用与读写分离 |
| `backupTime` `backupKeep` `backupScript` | 定时备份 |
| `dataDir` `pkg` | 数据目录、安装包 |

### 嵌套对象

```jsonc
"disks": [{          // 磁盘数组，一台主机可有多块
  "usage": "数据盘", "dev": "/dev/sdc", "type": "HDD",
  "lvm": "否", "lv": "-", "fs": "xfs",
  "size": "500 GB", "mount": "/data", "uuid": "..."
}],

"perf": {            // pgbench 实测
  "write3k": "12.9 MB/s",  "read3k": "4.6 GB/s",  "write8m": "245 MB/s",
  "lat1": 6.5,      "tps1Conn": 1539.41,  "tps1NoConn": 1539.90,   // 10 客户端 / 1000 事务
  "lat2": 25.03,    "tps2Conn": 7990.44,  "tps2NoConn": 7990.57    // 200 并发 / 120 秒
},

"tuning": {          // 数据库优化参数，详情抽屉自动转成 PG 原生参数名显示
  "maxConnections": "2000", "sharedBuffers": "8GB",
  "effectiveCacheSize": "24GB", "workMem": "16MB", "hugePages": "try"
}
```

`tuning` 的 key 用驼峰，渲染时按 `tuneLabel()` 映射为 `max_connections`、`shared_buffer` 等原生名。

## 资料库表结构与实体 ID

本地 JSON 是**扁平**的（一台主机一个对象）；存入资料库时会拆成 **10 张关联表**，
每层实体有自己的主键。页面在加载/保存时各做一次翻译（`assembleHosts` / `saveHostToLibrary`）。

| 实体 | 资料库主键 | 格式 | 上级外键 | 对应 JSON 字段 |
|---|---|---|---|---|
| 项目 | `项目ID` | `PJ01` | — | `project`（项目编码） |
| 服务器 | `服务器ID` | `SRV001` | `项目ID` | `id` / `seq` |
| 磁盘 | `硬盘ID` | `DSK001` | `服务器ID` | `disks[].dev/usage/...` |
| 数据库 | `数据库ID` | `DB001` | `服务器ID` | `dbVersion` / `dbid` / `ha` |
| 性能测试 | —（1:N 挂硬盘） | — | `硬盘ID` | `perf` |
| 操作系统 | —（1:1 挂服务器） | — | `服务器ID` | `os` / `osUser` / `osPass` / `locale` |
| 安装 / 授权 / 备份 / 参数 | —（挂数据库） | — | `数据库ID` | `installDate` / `licenseExpiry` / `backup*` / `tuning` |

ID 规则：按 `real.json` 中 hosts 的原始顺序依次编号，**稳定可复现**；
页面新增数据时按已有最大序号 +1 续号（如 `SRV031` → `SRV032`）。

> 关联键**不用** `seq`（资源序号）：它跨项目会重复（`G0-S1-1` 出现在 7 个项目里），
> 用它关联会导致子表数据串表。`seq` 仅作展示。

完整表清单、database_id、维护脚本见 [`TABLES.md`](TABLES.md)。

## 校验建议

新增数据后在浏览器控制台跑一遍，确认无空指针：

```javascript
const bad = state.hosts.filter(h => !h.id || !h.project);
console.log('缺主键的记录:', bad.length);
```
