---
name: scoop-bucket-add
description: "向本 scoop bucket 仓库（scoop-ao）添加新应用的 manifest：给定 GitHub 仓库地址（可附带 release tag），自动获取 release 资产与 hash、探测安装包结构、生成 bucket/<name>.json（含 checkver/autoupdate），并提交推送到 GitHub。触发词：添加 scoop manifest、加入 bucket、把这个仓库加到 scoop、scoop install、github release 打包。"
---

# scoop-bucket-add

## 何时使用

- 用户给出一个 **GitHub 仓库地址**（可附带 release 页面/tag 链接，如 `https://github.com/{owner}/{repo}/releases/tag/v1.2.3`），要求将其添加到本 scoop bucket。
- 用户要求为某个 GitHub 开源项目**创建 scoop 安装配置**并推送。

不要在以下情况使用：

- 更新已有 manifest 到新版（直接用 `bin/checkver.ps1 -Update` 或手动改版本号即可，本 skill 的流程同样适用但可跳过结构探测）。
- 目标项目没有 GitHub Releases 或没有 Windows 资产——此时告知用户原因并停止。

## 总体流程

1. **解析输入**：从 URL 提取 `owner/repo` 与可选 tag。无 tag 时取最新 release。
2. **获取 release 信息**（GitHub API，无需认证，但受限速）：
   ```bash
   curl -s https://api.github.com/repos/{owner}/{repo}/releases/tags/{tag}
   # 或最新版：curl -s https://api.github.com/repos/{owner}/{repo}/releases/latest
   ```
   用 `grep -E '"(name|browser_download_url|digest)"'` 提取资产清单。
3. **选择 Windows 64 位资产**，按优先级与类型处理：
   - `*-Setup-*.exe` / `*setup*.exe` → 大概率 **electron-builder NSIS 包**，走「NSIS 模式」。
   - `*.msi` → 用 `url` 直连 + `"extract_dir"` 不需要，scoop 原生支持 msi（一般无需特殊处理）。
   - `*-windows-x64.zip` / `*-win*.zip` / 便携 `.exe` → 普通压缩包/便携模式。
   - 有 arm64 Windows 资产时在 `architecture` 下增加 `arm64` 节。
   - **无任何 Windows 资产 → 明确告知用户并停止**，不要硬塞 Linux/mac 资产。
4. **获取 hash**：优先用 API 返回的 `digest` 字段（`sha256:...`，去掉前缀）；**必须下载文件用 `sha256sum` 实测校验一次**（防止 API 与实际文件不一致）。
5. **探测包内结构**（确定 shortcuts 目标 exe 名）：
   - NSIS 包：
     ```bash
     7z l installer.exe | grep -iE 'PLUGINSDIR|\.7z'        # 确认存在 $PLUGINSDIR\app-64.7z
     7z e -y -oout installer.exe '$PLUGINSDIR\app-64.7z'
     7z l out/app-64.7z | grep -E '\.exe'                  # 找主程序 exe 名
     ```
   - zip 包：直接 `7z l` 找主程序 exe；注意是否有嵌套顶层目录（需 `extract_dir`）。
6. **获取仓库元数据**（description、license、homepage）：
   ```bash
   curl -s https://api.github.com/repos/{owner}/{repo} | grep -E '"(description|homepage)"'
   curl -s https://api.github.com/repos/{owner}/{repo}/license | grep '"spdx_id"'
   ```
   description 用中文转述（本 bucket 惯例），homepage 优先用仓库主页 GitHub URL。
7. **编写 manifest**：`bucket/<name>.json`，`<name>` 取仓库名小写。模板见下。
8. **校验**：JSON 语法必须合法；可用 `bin/checkver.ps1 <name>` 验证 checkver 能取到版本。
9. **提交推送**（惯例提交信息：`<name>: Add version x.y.z`）：
   ```bash
   git add bucket/<name>.json
   git commit -m "<name>: Add version x.y.z"
   git push
   ```
10. **清理**：删除下载到临时目录的安装包与解压产物。

## Manifest 模板

### NSIS（electron-builder）模式

适用于 `*-Setup-*.exe`（包内含 `$PLUGINSDIR\app-64.7z`）：

```json
{
    "version": "x.y.z",
    "description": "中文描述",
    "homepage": "https://github.com/{owner}/{repo}",
    "license": "MIT",
    "architecture": {
        "64bit": {
            "url": "https://github.com/{owner}/{repo}/releases/download/vx.y.z/App-Setup-x.y.z.exe#/dl.7z",
            "hash": "<实测 sha256>"
        }
    },
    "pre_install": [
        "Expand-7zipArchive \"$dir\\`$PLUGINSDIR\\app-64.7z\" \"$dir\"",
        "Remove-Item \"$dir\\`$PLUGINSDIR\", \"$dir\\Uninstall*\" -Force -Recurse -ErrorAction SilentlyContinue"
    ],
    "shortcuts": [
        [
            "<主程序>.exe",
            "<显示名>"
        ]
    ],
    "checkver": "github",
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/{owner}/{repo}/releases/download/v$version/App-Setup-$version.exe#/dl.7z"
            }
        }
    }
}
```

要点：

- URL 必须带 `#/dl.7z` 后缀让 scoop 用 7zip 解压 exe。
- `pre_install` 中的 `` `$PLUGINSDIR `` 的反引号转义不能丢。
- `checkver: "github"` 自动匹配仓库 release tag；tag 带 `v` 前缀时 scoop 会自动剥离。
- `autoupdate.url` 用 `$version` 替换版本号，注意与 tag 前缀（`v$version`）保持一致。

### 普通 zip 模式

```json
{
    "version": "x.y.z",
    "description": "中文描述",
    "homepage": "https://github.com/{owner}/{repo}",
    "license": "MIT",
    "architecture": {
        "64bit": {
            "url": "https://github.com/{owner}/{repo}/releases/download/vx.y.z/App-x.y.z-windows-x64.zip",
            "hash": "<实测 sha256>"
        }
    },
    "shortcuts": [
        [
            "<主程序>.exe",
            "<显示名>"
        ]
    ],
    "checkver": "github",
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/{owner}/{repo}/releases/download/v$version/App-$version-windows-x64.zip"
            }
        }
    }
}
```

- zip 内有单一顶层目录时加 `"extract_dir": "<目录名>"`。
- 参考现有样例：`bucket/benchlocal.json`（zip 模式）、`bucket/ensocode.json`（NSIS 模式）。

## 注意事项与边界

- **hash 必须实测**：API `digest` 字段仅作参考，写入前下载文件 `sha256sum` 校验。
- **shortcut 目标必须是包内真实存在的 exe 名**（通过 7z 列目录确认），不要凭仓库名猜测。
- **只处理 Windows 资产**；纯 Linux/mac 项目直接告知用户无法添加。
- 版本号统一**不带 `v` 前缀**写入 `"version"`；URL 中的 tag 前缀保留。
- 提交前确认没有引入临时文件；只 `git add` manifest 文件本身。
- 推送远端固定为本仓库默认 remote（origin/main），不要新建分支除非用户要求。
- 大文件（>80MB）下载探测属正常成本，但不要重复下载——hash 校验与结构探测用同一份文件。
