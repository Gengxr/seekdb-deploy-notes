# OceanBase 大赛：部署 seekdb

本仓库记录[赛题一：部署一个 seekdb](https://oceanbase.yuque.com/org-wiki-obtech-vh7w9r/hc3pio/lz6ixow11mar233g)的部署和核验方法。

验收要求：启动自己的 seekdb 实例，连接并执行 SQL，读取运行目录下 `run/telemetry.json` 的 `content.id`，将完整 UUID 提交到提测平台。

## macOS 原生部署

本次使用 macOS 15.6、Apple Silicon，通过 [OceanBase 官方 Homebrew 包](https://github.com/oceanbase/homebrew-seekdb)安装 seekdb 1.4.0.0。需要 macOS 15 或更新版本、ARM64、Homebrew 和 MySQL 客户端。

```bash
brew tap oceanbase/seekdb
brew install oceanbase/seekdb/seekdb
```

首次运行、前台调试：

```bash
seekdb_prefix="$(brew --prefix seekdb)"
seekdb_base="$(brew --prefix)/var/seekdb/data"
DYLD_FALLBACK_LIBRARY_PATH="$seekdb_prefix/libexec/compat:/usr/local/lib:/lib:/usr/lib" \
  "$seekdb_prefix/bin/seekdb" --nodaemon --base-dir="$seekdb_base" \
  --parameter memory_limit=2G --parameter cpu_count=4
```

如需脱离终端运行，可在前台实例停止后交给 macOS `launchd` 托管。本次部署使用此方式：

```bash
mkdir -p .local
seekdb_prefix="$(brew --prefix seekdb)"
seekdb_base="$(brew --prefix)/var/seekdb/data"
launchctl submit -l com.gengxr.seekdb.contest \
  -o "$PWD/.local/seekdb.log" -e "$PWD/.local/seekdb-error.log" \
  -- /usr/bin/env \
  "DYLD_FALLBACK_LIBRARY_PATH=$seekdb_prefix/libexec/compat:/usr/local/lib:/lib:/usr/lib" \
  "$seekdb_prefix/bin/seekdb" --nodaemon --base-dir="$seekdb_base" \
  --parameter memory_limit=2G --parameter cpu_count=4
```

`launchctl submit` 任务属于当前用户会话，重启或注销后需要重新执行启动命令。不要同时启动多个实例占用 2881 端口。

## SQL 核验

```bash
mysql --no-defaults --protocol=tcp -h127.0.0.1 -P2881 -uroot \
  --connect-timeout=5 --table < verify.sql
launchctl list com.gengxr.seekdb.contest
```

本次已验证版本为 `5.7.25-OceanBase seekdb-v1.4.0.0`，算术查询返回 `2`，数据库列表包含 `information_schema`、`mysql`、`oceanbase`、`test`。

## 提取并提交遥测 UUID

```bash
python3 - <<'PY'
import json
import subprocess
import uuid
from pathlib import Path

prefix = subprocess.check_output(["brew", "--prefix"], text=True).strip()
path = Path(prefix) / "var/seekdb/data/run/telemetry.json"
deployment_id = json.loads(path.read_text())["content"]["id"]
assert str(uuid.UUID(deployment_id)) == deployment_id
print(deployment_id)
PY
```

登录[大赛提测平台](https://open.oceanbase.com/train/detail/21?questionId=600071)，选择“seekdb 基础实践”，点击“立即提测”，在“部署 UUID”中填入上述输出并提交。无需提交整个 JSON、仓库分支或 Commit，也无需为部署项填写 Fork 和 PR。

平台会查询遥测上报，同一 UUID 不能被其他队伍重复认领。遥测同步可能存在延迟；以平台提测记录为最终计分依据。请将自己的 UUID 和原始遥测文件保存在本地，避免公开后被提前认领。

## 停止当前实例

```bash
launchctl remove com.gengxr.seekdb.contest
```

此命令停止托管任务，保留实例数据与遥测文件。

## 仓库用途

本仓库保存部署文档，部署赛题的提交产物是实例 UUID。本仓库当前并非官方 `oceanbase/seekdb` 的 Fork，不能直接用于基础实践中的 Fork 核验。若此仓库用于正式报名，平台说明要求报名答题仓库保持私有；请按队伍报名用途配置仓库。
