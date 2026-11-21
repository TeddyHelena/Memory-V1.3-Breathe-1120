# 安装 MemoryV1.3-Breathe 公开预览版

本指南用于在 macOS 上切换 **已配置 `dsh-memory-container`** 的 DSH 桌面 Profile。维护者记录的实测环境为 macOS + DSH 0.2.2-rc.3。宿主需提供 `ctx.on` 与 `webserver/index-inject`。如果还没有容器、插件配置或现有软链接，请先完成适用于你的 DSH 环境的初始配置。

## 1. 下载并核对版本

从 [公开 Release](https://github.com/TeddyHelena/Memory-V1.3-Breathe-1120/releases/tag/v0.16.3%2Bbreathe.1120.3) 的 Assets 下载 **`Memory-V1.3-Breathe-1120.3.zip`**。GitHub 同页还列出自动生成的 Source code 压缩包。按本指南操作时，使用上述文件名的附件。

| 核对项 | 值 |
| --- | --- |
| Release 标签 | `v0.16.3+breathe.1120.3` |
| ZIP 内的插件版本 | `0.16.3+breathe.1120.3` |
| ZIP 的 SHA-256 | `aa282ea35f41e52993f14df472e873ed8016f5c146c0d3ecdef8ad40573d57d7` |
| 解压后的插件根目录 | `dsh-memory-breathe/` |
| 仓库 `main` 的包清单版本 | `0.16.3+breathe.1120.4` |

ZIP 内的 `dsh-memory-breathe/` 包含 `package.json`、`lib/index.js`、`lib/client.js` 和 `src/`。发布页提供的附件只有安装 ZIP，下面的命令会直接校验该 ZIP 的摘要。

## 2. 备份并切换插件

1. 用 **Cmd+Q** 完全退出 DSH，操作完成前保持退出。
2. 打开当前 Profile 的 `cordis.patch.yml`，确认插件的 `containerRoot`。准备本地 Python 3，并确保备份磁盘空间充足。
3. 将下面首行的两个参数改成 ZIP 实际路径及 **现有容器的绝对路径**，再执行整段。若使用自定义 Profile，同时修改脚本中的 `PROFILE`。

脚本会先校验 ZIP、解压到临时目录并核对包清单，再备份 Profile 文件、容器及原软链接。插件进入一个新目录，已有插件目录会保留。请记下终端打印的备份路径。

```bash
bash -s -- "$HOME/Downloads/Memory-V1.3-Breathe-1120.3.zip" '/绝对路径/现有记忆容器' <<'BASH'
set -euo pipefail
umask 077
ZIP=$1
CONTAINER=$2
PROFILE="$HOME/.dsh/profiles/desktop"
LINK="$PROFILE/node_modules/dsh-memory-container"

[[ "$CONTAINER" = /* && "$CONTAINER" != / && -d "$CONTAINER" &&
   -f "$PROFILE/package.json" && -f "$PROFILE/cordis.patch.yml" &&
   -L "$LINK" && -d "$LINK" ]] || {
  echo '请核对容器、Profile 文件和现有插件软链接。' >&2
  exit 1
}
CONTAINER="$(cd "$CONTAINER" && pwd -P)"
[[ "$CONTAINER" != / ]]
EXPECTED=aa282ea35f41e52993f14df472e873ed8016f5c146c0d3ecdef8ad40573d57d7
ACTUAL="$(shasum -a 256 < "$ZIP" | awk '{print $1}')"
[[ "$ACTUAL" = "$EXPECTED" ]] || { echo 'ZIP 校验失败，停止安装。' >&2; exit 1; }

STAGE="$(mktemp -d "${TMPDIR:-/tmp}/breathe.XXXXXXXX")"
trap 'rm -rf -- "$STAGE"' EXIT
unzip -q "$ZIP" -d "$STAGE"
SOURCE="$STAGE/dsh-memory-breathe"
[[ -f "$SOURCE/lib/index.js" && -f "$SOURCE/lib/client.js" && -d "$SOURCE/src" ]]
python3 - "$SOURCE/package.json" "$PROFILE/package.json" "$CONTAINER" <<'PY'
import json
import sys
from pathlib import Path

plugin = json.loads(Path(sys.argv[1]).read_text())
profile = json.loads(Path(sys.argv[2]).read_text())
name = "dsh-memory-container"
container = Path(sys.argv[3])
backup_root = (Path.home() / ".dsh/backups").resolve()
if container == backup_root or container in backup_root.parents:
    raise SystemExit("请选择具体的记忆容器，避免把备份目录打包进自身")
if (plugin.get("name"), plugin.get("version")) != (name, "0.16.3+breathe.1120.3"):
    raise SystemExit("插件包名或版本与公开发布包不符")
if name not in profile.get("dependencies", {}):
    raise SystemExit("当前 Profile 未配置 dsh-memory-container")
PY

mkdir -p "$HOME/.dsh/backups" "$HOME/.dsh/plugins"
BACKUP="$(mktemp -d "$HOME/.dsh/backups/breathe.XXXXXXXX")"
cp -p "$PROFILE/package.json" "$PROFILE/cordis.patch.yml" "$BACKUP/"
cp -P "$LINK" "$BACKUP/original-link"
tar -czf "$BACKUP/memory-container.tgz" -C "$(dirname "$CONTAINER")" "$(basename "$CONTAINER")"
tar -tzf "$BACKUP/memory-container.tgz" >/dev/null
echo "备份路径：$BACKUP"

DEST="$(mktemp -d "$HOME/.dsh/plugins/dsh-memory-container-breathe-1120.3.XXXXXXXX")"
cp -R "$SOURCE/." "$DEST/"
python3 - "$PROFILE" "$LINK" "$DEST" <<'PY'
import json
import os
import shutil
import sys
import tempfile
from pathlib import Path

profile, link, dest = map(Path, sys.argv[1:])
manifest = profile / "package.json"
data = json.loads(manifest.read_text())
data["dependencies"]["dsh-memory-container"] = f"link:{dest}"
with tempfile.TemporaryDirectory(dir=profile) as manifest_stage, \
     tempfile.TemporaryDirectory(dir=link.parent) as link_stage:
    new_manifest = Path(manifest_stage) / "package.json"
    new_link = Path(link_stage) / "plugin"
    old_link = Path(link_stage) / "original"
    shutil.copy2(manifest, new_manifest)
    new_manifest.write_text(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
    new_link.symlink_to(dest)
    old_link.symlink_to(os.readlink(link))
    os.replace(new_link, link)
    try:
        os.replace(new_manifest, manifest)
    except OSError:
        os.replace(old_link, link)
        raise
PY
[[ "$(readlink "$LINK")" = "$DEST" ]]
echo "插件路径：$DEST"
BASH
```

## 3. 重启并验收

1. 启动 DSH，打开侧栏「记忆」。确认列表、知识球和关系图能加载，已有条目仍可查看。若 Profile 已允许编辑，可用测试条目验证建链和删链。
2. 如果界面返回 401，检查当前宿主是否能注入页面令牌，以及 Profile 的 `apiToken` 和 `apiPrefix` 配置。保留令牌校验。若宿主不兼容，退出并按下面步骤回滚。
3. 如需用命令行验收，请使用 DSH 实际的本地端口、配置的 API 前缀和对应令牌。只有设置了 `apiTokenFile` 时才有插件写出的令牌文件。可在桌面界面的网络面板核对列表请求是否返回 200，避免复制或分享请求中的令牌。

## 4. 回滚插件

完全退出 DSH。把首行的备份路径替换为安装时打印的实际路径。如使用自定义 Profile，请同步修改 `PROFILE`。以下步骤恢复原包清单和插件软链接，`cordis.patch.yml` 的备份也保存在同一目录，供曾手动编辑该文件的用户核对。

```bash
bash -s -- '/绝对路径/上一步打印的备份目录' <<'BASH'
set -euo pipefail
BACKUP=$1
PROFILE="$HOME/.dsh/profiles/desktop"
LINK="$PROFILE/node_modules/dsh-memory-container"
[[ -f "$BACKUP/package.json" && -L "$BACKUP/original-link" && -L "$LINK" ]]
STAGE="$(mktemp -d "$PROFILE/.breathe-rollback.XXXXXXXX")"
trap 'rm -rf -- "$STAGE"' EXIT
cp -p "$BACKUP/package.json" "$STAGE/package.json"
python3 - "$STAGE/package.json" "$PROFILE/package.json" "$BACKUP/original-link" "$LINK" <<'PY'
import os
import sys
import tempfile
from pathlib import Path

backup_manifest, manifest, backup_link, link = map(Path, sys.argv[1:])
with tempfile.TemporaryDirectory(dir=link.parent) as stage:
    restored = Path(stage) / "restored"
    current = Path(stage) / "current"
    restored.symlink_to(os.readlink(backup_link))
    current.symlink_to(os.readlink(link))
    os.replace(restored, link)
    try:
        os.replace(backup_manifest, manifest)
    except OSError:
        os.replace(current, link)
        raise
PY
readlink "$LINK"
BASH
```

重启后确认原插件和容器可读。如果还需要恢复数据，将 `memory-container.tgz` 解压到独立目录核对，并保留当前容器副本后再决定是否切换。归档只包含安装前的数据。
