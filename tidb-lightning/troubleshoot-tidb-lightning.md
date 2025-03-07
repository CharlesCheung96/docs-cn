---
title: TiDB Lightning 故障处理
summary: 本文档总结了使用 TiDB Lightning 过程中常见的运行故障及解决方案。
aliases: ['/docs-cn/dev/troubleshoot-tidb-lightning/','/docs-cn/dev/how-to/troubleshoot/tidb-lightning/','/docs-cn/dev/reference/tools/error-case-handling/lightning-misuse-handling/','/docs-cn/dev/tidb-lightning/tidb-lightning-misuse-handling/','/zh/tidb/dev/troubleshoot-tidb-lightning/']
---

# TiDB Lightning 故障处理

本文档总结了使用 TiDB Lightning 过程中常见的运行故障及解决方案。

## TiDB Lightning 导入速度太慢

TiDB Lightning 的正常速度为每条线程每 2 分钟导入一个 256 MB 的数据文件，如果速度远慢于这个数值就是有问题。导入的速度可以检查日志提及 `restore chunk … takes` 的记录，或者观察 Grafana 的监控信息。

导入速度太慢一般有几个原因：

**原因 1**：`region-concurrency` 设定太高，线程间争用资源反而减低了效率。

1. 从日志的开头搜寻 `region-concurrency` 能知道 TiDB Lightning 读到的参数是多少。
2. 如果 TiDB Lightning 与其他服务（如 TiKV Importer）共用一台服务器，必需**手动**将 `region-concurrency` 设为该服务器 CPU 数量的 75%。
3. 如果 CPU 设有限额（例如从 Kubernetes 指定的上限），TiDB Lightning 可能无法自动判断出来，此时亦需要**手动**调整 `region-concurrency`。

**原因 2**：表结构太复杂。

每条索引都会额外增加键值对。如果有 N 条索引，实际导入的大小就差不多是 Dumpling 文件的 N+1 倍。如果索引不太重要，可以考虑先从 schema 去掉，待导入完成后再使用 `CREATE INDEX` 加回去。

**原因 3**：单个文件过大。

把源数据分割为单个大小约为 256 MB 的多个文件时，TiDB Lightning 会并行处理数据，达到最佳效果。如果导入的单个文件过大，TiDB Lightning 可能无响应。

如果源数据是 CSV 格式文件，并且所有的 CSV 文件内都不存在包含字符换行符的字段 (U+000A 及 U+000D)，则可以启用 `strict-format`，TiDB Lightning 会自动分割大文件。

```toml
[mydumper]
strict-format = true
```

**原因 4**：TiDB Lightning 版本太旧。

建议试试最新的版本，可能会有改善。

## `tidb-lightning` 进程意外退出

这种情况可能是启动方式不正确，导致收到 SIGHUP 信号而退出。此时 `tidb-lightning.log` 通常有如下日志：

```
[2018/08/10 07:29:08.310 +08:00] [INFO] [main.go:41] ["got signal to exit"] [signal=hangup]
```

不推荐在命令行中直接使用 `nohup` 启动进程，推荐[使用脚本启动 `tidb-lightning`](/tidb-lightning/deploy-tidb-lightning.md)。

另外，如果从 TiDB Lightning 的 log 的最后一条日志显示遇到的错误是 "Context canceled"，需要在日志中搜索第一条 "ERROR" 级别的日志。在这条日志之前，通常也会紧跟有一条 "got signal to exit"，表示 Lightning 是收到中断信号然后退出的。

## 使用 TiDB Lightning 后，TiDB 集群变慢，CPU 占用高

如果 `tidb-lightning` 异常退出，集群可能仍处于“导入模式” (import mode)，该模式不适用于生产环境。此时可执行以下命令查看当前使用的模式：

{{< copyable "shell-regular" >}}

```sh
tidb-lightning-ctl --config tidb-lightning.toml --fetch-mode
```

可执行以下命令强制切换回“普通模式” (normal mode)：

{{< copyable "shell-regular" >}}

```sh
tidb-lightning-ctl --config tidb-lightning.toml --switch-mode=normal
```

## TiDB Lightning 报错

### `could not find first pair, this shouldn't happen`

报错原因是遍历本地排序的文件时出现异常，可能在 TiDB Lightning 打开的文件数量超过系统的上限时发生报错。在 Linux 系统中，可以使用 `ulimit -n` 命令确认此值是否过小。建议在导入期间将此设置调整为 `1000000`（即 `ulimit -n 1000000`）。

### `checksum failed: checksum mismatched remote vs local`

**原因**：本地数据源跟目标数据库某个表的校验和不一致。这通常有更深层的原因，可以通过检查日志中包含 `checksum mismatched` 的行进一步定位。

包含 `checksum mismatched` 的行中有 `total_kvs: x vs y` 的信息，`x` 表示导入集群在完成导入后计算出的键值对（KV pairs）数目，`y` 表示本地数据源产生的键值对数目。

- `x` 大，即导入集群键值对更多：
    - 可能这张表在导入前已有数据，因此影响了数据校验。这也包括 TiDB Lightning 之前失败停机过，但没有正确重启。
- `y` 大，即本地数据源键值对更多：
    - 如果目标数据库的校验和全是 0，表示没有发生任何导入，有可能是集群太忙无法接收任何数据。
    - 可能导出数据中包含重复数据，例如唯一键和主键 (UNIQUE and PRIMARY KEYs) 有重复的值、下游表结构为大小写不敏感而数据为大小写敏感。
- 其他情况
    - 如果数据源是由机器生成而不是从 Dumpling 备份的，需确保数据符合表的限制，例如自增 (AUTO_INCREMENT) 的列需要为正数，不能为 0。

**解决办法**：

1. 使用 `tidb-lightning-ctl` 把出错的表删除，检查表结构与数据，重启 TiDB Lightning 重新导入此前出错的表。

    {{< copyable "shell-regular" >}}

    ```sh
    tidb-lightning-ctl --config conf/tidb-lightning.toml --checkpoint-error-destroy=all
    ```

2. 把断点存放在外部数据库（修改 `[checkpoint] dsn`），减轻目标集群压力。

3. 参考[如何正确重启 TiDB Lightning](/tidb-lightning/tidb-lightning-faq.md#如何正确重启-tidb-lightning)中的解决办法。

### `Checkpoint for … has invalid status:`（错误码）

**原因**：[断点续传](/tidb-lightning/tidb-lightning-checkpoints.md)已启用。TiDB Lightning 或 TiKV Importer 之前发生了异常退出。为了防止数据意外损坏，TiDB Lightning 在错误解决以前不会启动。

错误码是小于 25 的整数，可能的取值是 0、3、6、9、12、14、15、17、18、20、21。整数越大，表示异常退出所发生的步骤在导入流程中越晚。

**解决办法**：

如果错误原因是非法数据源，使用 `tidb-lightning-ctl` 删除已导入数据，并重启 TiDB Lightning。

{{< copyable "shell-regular" >}}

```sh
tidb-lightning-ctl --config conf/tidb-lightning.toml --checkpoint-error-destroy=all
```

其他解决方法请参考[断点续传的控制](/tidb-lightning/tidb-lightning-checkpoints.md#断点续传的控制)。

### `ResourceTemporarilyUnavailable("Too many open engines …: …")`

**原因**：并行打开的引擎文件 (engine files) 超出 `tikv-importer` 里的限制。这可能由配置错误引起。即使配置没问题，如果 `tidb-lightning` 曾经异常退出，也有可能令引擎文件残留在打开的状态，占据可用的数量。

**解决办法**：

1. 提高 `tikv-importer.toml` 内 `max-open-engines` 的值。这个设置主要由内存决定，计算公式为：

    最大内存使用量 ≈ `max-open-engines` × `write-buffer-size` × `max-write-buffer-number`

2. 降低 `table-concurrency` + `index-concurrency`，使之低于 `max-open-engines`。

3. 重启 `tikv-importer` 来强制移除所有引擎文件 (默认值为 `./data.import/`)。这样也会丢弃导入了一半的表，所以启动 TiDB Lightning 前必须清除过期的断点记录：

    {{< copyable "shell-regular" >}}

    ```sh
    tidb-lightning-ctl --config conf/tidb-lightning.toml --checkpoint-error-destroy=all
    ```

### `cannot guess encoding for input file, please convert to UTF-8 manually`

**原因**：TiDB Lightning 只支持 UTF-8 和 GB-18030 编码的表架构。此错误代表数据源不是这里任一个编码。也有可能是文件中混合了不同的编码，例如，因为在不同的环境运行过 `ALTER TABLE`，使表架构同时出现 UTF-8 和 GB-18030 的字符。

**解决办法**：

1. 编辑数据源，保存为纯 UTF-8 或 GB-18030 的文件。
2. 手动在目标数据库创建所有的表。
3. 设置 `[mydumper] character-set = "binary"` 跳过这个检查。但是这样可能使数据库出现乱码。

### `[sql2kv] sql encode error = [types:1292]invalid time format: '{1970 1 1 …}'`

**原因**: 一个 `timestamp` 类型的时间戳记录了不存在的时间值。时间值不存在是由于夏时制切换或超出支持的范围（1970 年 1 月 1 日至 2038 年 1 月 19 日）。

**解决办法**:

1. 确保 TiDB Lightning 与数据源时区一致。

    * 手动部署的话，通过设定 `$TZ` 环境变量强制时区设定。

        强制使用 Asia/Shanghai 时区：

        {{< copyable "shell-regular" >}}

        ```sh
        TZ='Asia/Shanghai' bin/tidb-lightning -config tidb-lightning.toml
        ```

2. 导出数据时，必须加上 `--skip-tz-utc` 选项。

3. 确保整个集群使用的是同一最新版本的 `tzdata` (2018i 或更高版本)。

    如果你使用的是 CentOS 机器，你可以运行 `yum info tzdata` 命令查看 `tzdata` 的版本及是否有更新。然后运行 `yum upgrade tzdata` 命令升级 `tzdata`。

### `[Error 8025: entry too large, the max entry size is 6291456]`

**原因**：TiDB Lightning 生成的单行 KV 超过了 TiDB 的限制。

**解决办法**:

目前无法绕过 TiDB 的限制，只能忽略这张表，确保其它表顺利导入。

### switch-mode 时遇到 `rpc error: code = Unimplemented ...`

**原因**：集群中有不支持 switch-mode 的节点。目前已知的组件中，4.0.0-rc.2 之前的 TiFlash [不支持 switch-mode 操作](https://github.com/pingcap/tidb-lightning/issues/273)。

**解决办法**：

- 如果集群中有 TiFlash 节点，可以将集群更新到 4.0.0-rc.2 或更新版本。
- 如果不方便升级，可以临时禁用 TiFlash。

### `tidb lightning encountered error: TiDB version too old, expected '>=4.0.0', found '3.0.18'`

TiDB Lightning Local-backend 只支持导入到 v4.0.0 及以上版本的 TiDB 集群。如果尝试使用 Local-backend 导入到 v2.x 或 v3.x 的集群，就会报以上错误。此时可以修改配置使用 Importer-backend 或 TiDB-backend 进行导入。

部分 `nightly` 版本的 TiDB 集群的版本可能类似 4.0.0-beta.2。这种版本的 TiDB Lightning 实际支持 Local-backend，如果使用 `nightly` 版本遇到该报错，可以通过设置配置  `check-requirements = false` 跳过版本检查。在设置此参数之前，请确保 TiDB Lightning 的配置支持对应的版本，否则无法保证导入成功。

### `restore table test.district failed: unknown columns in header [...]`

出现该错误通常是因为 CSV 格式的数据文件不包含 header（第一行也是数据），因此需要在 TiDB Lightning 的配置文件中增加如下配置项：

```
[mydumper.csv]
header = false
```

### `Unknown character set`

由于 TiDB 只支持部分 MySQL 字符集，因此，在导入流程中，如果创建表结构时使用了 TiDB 不支持的字符集，TiDB Lightning 会报这个错误。你可以结合数据内容选择 [TiDB 支持的字符集](/character-set-and-collation.md)，预先在下游创建表结构以绕过这个错误。

### `invalid compression type ...`

TiDB v6.4.0 及之后版本的 TiDB Lightning 不支持带有非 `.bak` 后缀的数据文件并报错。你需要提前修改文件名，或将该类文件移出导入数据目录来避免此类错误。更多详情请参考[压缩导出的数据文件](/tidb-lightning/tidb-lightning-data-source.md#压缩文件)。
