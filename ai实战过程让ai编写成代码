#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
dsh_plugin_toolkit.py —— DSH 插件安装 + 插件数据搬迁工具
=========================================================

把「给 DeepSeek Harness 装插件」和「把插件相关文件搬到 D 盘」这两件事，
整理成一套可复用、可审计、默认安全的流程。

本脚本还原自一次真实操作（2026-10-01 ~ 10-02），下面每条都是实际踩过的坑：

  1. 装插件要用【桌面版自带的 CLI】。PATH 上那个 %APPDATA%\\npm 里的
     全局 dsh 是旧版本，用它装/启动会直接崩。
  2. DSH 有【兼容性闸门】：插件（或它依赖的核心包）声明的 peer 版本
     和运行时对不上，安装会被拒绝并【自动回滚】。
     放行要用 allow-version，而且豁免只写进目标 profile 自己的
     compatibility.json —— 别写进你日常在用的那个 profile。
  3. 换了 pnpm 的 store 位置后，必须让 pnpm 真正重建 node_modules，
     否则装包时报 ERR_PNPM_UNEXPECTED_STORE（pnpm install 会误判"已最新"）。
  4. 搬目录一律走：复制 → 校验 → 改名留底 → 建 Junction → 验证 → 删留底。
     任何一步失败就回滚当前这一个目录，不影响其它。
  5. 有三类目录【不要搬】，见 CANNOT_MOVE —— 搬了会让插件起不来，
     而且症状很隐蔽（启动时静默少加载若干入口）。
  6. 长路径和符号链接要用 robocopy（/E /XJ），不要用 shutil.copytree：
     前者会静默跳过超长路径、后者会跟随链接把目标整棵树复制一遍。
  7. 删除 Junction 只能用 os.rmdir()。千万别用 shutil.rmtree() ——
     它会顺着链接把【真实目标目录】里的文件全删掉。

用法：
    python dsh_plugin_toolkit.py status              # 现状体检（只读，先跑这个）
    python dsh_plugin_toolkit.py install <包名>      # 装一个插件
    python dsh_plugin_toolkit.py move <目录名>       # 搬一个插件数据目录
    python dsh_plugin_toolkit.py move-all            # 搬所有"可搬"的数据目录
    python dsh_plugin_toolkit.py readme              # 重新生成 说明.txt

    任意命令后加 --dry-run  = 只打印计划，不动任何文件。

作者备注：脚本里的路径是本机的实际情况，换机器请改 CONFIG 段。
"""

from __future__ import annotations

import argparse
import json
import os
import shutil
import stat
import subprocess
import sys
from pathlib import Path

# ============================================================================
# CONFIG —— 换机器只需要改这一段
# ============================================================================

DSH_ROOT = Path(r"D:\deepseek_manager")          # 桌面版安装目录
DSH_HOME = Path(os.environ.get("DSH_HOME", Path.home() / ".dsh"))
PROFILE = "desktop"                              # 要管理的 profile 名

# 插件相关文件统一搬到这个目录下
PLUGIN_DIR = Path(r"D:\deepseek插件")
DATA_SUBDIR = PLUGIN_DIR / "插件数据"            # 插件运行时数据
STORE_DIR = PLUGIN_DIR / ".pnpm-store"           # pnpm 内容寻址缓存

# 桌面版自带的 CLI 与 pnpm（注意：不是 PATH 上那个全局 dsh！）
DSH_CLI = DSH_ROOT / "resources" / "runtime" / "cli" / "bin" / "dsh.cmd"
ELECTRON = DSH_ROOT / "DeepSeek Harness.exe"
PNPM_MJS = DSH_ROOT / "resources" / "runtime" / "pnpm" / "bin" / "pnpm.mjs"

# 托管这些数据的插件 / 含义。用于生成说明文件，也用于判断"这是谁的"。
DATA_DIRS = {
    "skins": "主题（dsh-web-all）",
    "skin-center": "皮肤中心的壁纸",
    "speech-to-text": "语音转文字模型",
    "task-board": "任务板",
    "dsh-usage": "用量统计",
    "dsh-session-archive": "会话归档",
    "boot-animation": "开机动画的选择记录",
    "whale-audio": "鲸鱼挂件",
    "whale-bubble-imgs": "鲸鱼挂件",
    "whale-roles": "鲸鱼挂件",
}

# 【不要搬】这些是 DSH 核心数据：正被应用频繁读写，体积也很小，搬了收益低、
# 风险高（文件被占用会导致改名失败，甚至损坏会话）。
CANNOT_MOVE = {
    "profiles": "各 profile 的配置与安装区",
    "sessions": "会话记录（核心）",
    "attachments": "附件（核心）",
    "cache": "缓存",
    "storages": "存储域",
    "llm-deepseek": "模型提供商数据",
}

# 【不要搬】插件安装区里，依赖"向上查找兄弟目录"的那种，搬了会坏。
# 实例：dsh-tui 插件在代码里 import 宿主的 @deepseek-ai/* 核心包，
# 靠 Node 从 profiles/<name>/node_modules 往上走到 profiles/node_modules
# 才能解析到；搬到 D 盘这条路径就断了，启动时 8 个入口 failed to import。
# 而把 @deepseek-ai 补进 profile 自己的 node_modules 又会顶掉宿主的核心行。
# 结论：这类 profile 的 node_modules 保持原位。
NEVER_MOVE_INSTALL_AREA = {"dsh-tui"}

DRY_RUN = False


# ============================================================================
# 基础工具
# ============================================================================

def run(cmd: list[str], cwd: Path | None = None, env_extra: dict | None = None,
        check: bool = True) -> subprocess.CompletedProcess:
    """执行外部命令。默认继承当前环境，可额外注入环境变量。"""
    env = dict(os.environ)
    if env_extra:
        env.update(env_extra)
    print(f"  $ {' '.join(str(c) for c in cmd)}")
    if DRY_RUN:
        return subprocess.CompletedProcess(cmd, 0, "", "")
    return subprocess.run([str(c) for c in cmd], cwd=cwd, env=env,
                          capture_output=True, text=True, check=check)


def pnpm(args: list[str], cwd: Path) -> subprocess.CompletedProcess:
    """调用桌面版自带的 pnpm（跑在 Electron 的 node 模式里）。"""
    return run([ELECTRON, "--expose-internals", PNPM_MJS, *args],
               cwd=cwd, env_extra={"ELECTRON_RUN_AS_NODE": "1"})


def is_link(path: Path) -> bool:
    """
    是否是符号链接 / Junction。
    必须查 Windows 的 reparse point 属性 —— 因为 os.path.islink() 对
    Junction 的判定在各 Python 版本里不一致，而本脚本的安全性（尤其是
    "删除时只删链接、不动目标"）完全依赖这个判断准确。
    """
    if path.is_symlink():
        return True
    if os.name == "nt":
        try:
            attrs = os.stat(path, follow_symlinks=False).st_file_attributes
            return bool(attrs & stat.FILE_ATTRIBUTE_REPARSE_POINT)
        except (OSError, AttributeError):
            return False
    return False


def link_target(path: Path) -> str:
    """安全地读出链接 / Junction 的目标；读不出来就返回占位符，不让脚本崩。"""
    try:
        return os.readlink(path)
    except OSError:
        return "(未知)"


def dir_stat(path: Path) -> tuple[int, float]:
    """返回 (文件数, MB)。注意：长路径可能让 Python 也数不全，仅作交叉校验。"""
    if not path.exists():
        return (0, 0.0)
    n, total = 0, 0
    for root, _dirs, files in os.walk(path):
        for f in files:
            try:
                total += os.path.getsize(os.path.join(root, f))
                n += 1
            except OSError:
                pass                                    # 锁住/超长，跳过
    return (n, round(total / 1024 / 1024, 2))


def robocopy_copy(src: Path, dst: Path) -> None:
    """
    用 robocopy 复制目录。
      /E   含子目录（含空目录）
      /XJ  排除 Junction 和符号链接 —— 关键！否则会顺着链接把目标整棵树复制进来
           （真实教训：pnpm store 里有个指向 profile 的链接，结果复制出 516 MB 垃圾）
    """
    if DRY_RUN:
        print(f"  [dry-run] robocopy {src} -> {dst}")
        return
    r = subprocess.run(["robocopy", str(src), str(dst), "/E", "/XJ",
                        "/NFL", "/NDL", "/NJH", "/NJS", "/R:1", "/W:1"],
                       capture_output=True, text=True)
    if r.returncode > 7:                            # robocopy: 0~7 都算成功
        raise RuntimeError(f"robocopy 失败 (exit={r.returncode})\n{r.stdout}\n{r.stderr}")


def make_junction(link: Path, target: Path) -> None:
    """建目录链接。mklink /J 不需要管理员权限。"""
    if DRY_RUN:
        print(f"  [dry-run] mklink /J {link} -> {target}")
        return
    r = subprocess.run(f'cmd /c mklink /J "{link}" "{target}"',
                       capture_output=True, text=True, shell=True)
    if r.returncode != 0:
        raise RuntimeError(f"建链接失败: {r.stdout} {r.stderr}")


def remove_junction(link: Path) -> None:
    """
    删除链接本身，【不动目标】。
    只能用 os.rmdir —— shutil.rmtree 会顺着链接删掉真实目标里的文件。
    """
    if DRY_RUN:
        print(f"  [dry-run] 删除链接 {link}")
        return
    os.rmdir(link)


# ============================================================================
# 任务一：安装插件
# ============================================================================

def install_plugin(spec: str, profile: str = PROFILE, exemptions: list[str] | None = None) -> bool:
    """
    装一个插件。

    spec 可以是：
      - npm 包名            "dsh-whale-widget"
      - npm 带版本          "@deepseek-harness-tui/dsh-tui@0.12.0"
      - 本地目录（绝对路径） "D:\\deepseek插件\\dshmarket"   → pnpm 会记成 link:
      - GitHub              "github:owner/repo"

    exemptions: 遇到 "incompatible" 拒绝时，允许放行的精确版本列表，
                形如 ["@deepseek-ai/dsh-agent-presets@0.1.1-rc.2"]。
                只写进这个 profile 的 compatibility.json，不影响别的 profile。
    """
    print(f"\n[1] 安装 {spec} 到 profile '{profile}'")

    # 1a. 先记下安装前的清单，出问题好对比
    manifest = DSH_HOME / "profiles" / profile / "package.json"
    before = json.loads(manifest.read_text(encoding="utf-8")) if manifest.exists() else {}

    # 1b. 正式安装（DSH 会在装完后做兼容性检查；不通过会自动回滚）
    r = run([DSH_CLI, "plugin", "--profile", profile, "add", spec], check=False)
    print(r.stdout)
    if r.returncode == 0:
        return verify_install(profile, spec, before)

    # 1c. 被兼容性闸门拒绝了 —— 只在调用方明确给出豁免时才放行
    if "incompatible" not in (r.stdout + r.stderr):
        print("  ✗ 安装失败，且不是版本兼容问题，请自行排查上面的输出")
        return False
    if not exemptions:
        print("  ✗ 被兼容性闸门拒绝。确认要放行的话，把确切版本填进 exemptions 再跑。")
        return False

    runtime = get_runtime_version()
    for pkg in exemptions:
        print(f"  → 授予豁免 {pkg}（仅限 profile '{profile}'）")
        run([DSH_CLI, "plugin", "--profile", profile, "allow-version",
             pkg, "--dsh-version", runtime, "--accept-risk"])

    # 1d. 重试
    r = run([DSH_CLI, "plugin", "--profile", profile, "add", spec], check=False)
    print(r.stdout)
    if r.returncode != 0:
        print("  ✗ 仍然失败")
        return False
    return verify_install(profile, spec, before)


def get_runtime_version() -> str:
    """读桌面运行时版本（app.asar 里的 dsh 包版本）。"""
    try:
        p = DSH_ROOT / "resources" / "app.asar" / "dsh" / "package.json"
        return json.loads(p.read_text(encoding="utf-8")).get("version", "unknown")
    except Exception:
        return "unknown"          # 读不到就让调用方自己填


def verify_install(profile: str, spec: str, before: dict) -> bool:
    """装完后的三道校验：清单、bundle 选择、文件真的在 D 盘。"""
    base = DSH_HOME / "profiles" / profile
    after = json.loads((base / "package.json").read_text(encoding="utf-8"))

    deps = after.get("dependencies", {})
    bundles = after.get("dsh", {}).get("profile", {}).get("bundles", [])
    print(f"  ✓ dependencies: {json.dumps(deps, ensure_ascii=False)}")
    print(f"  ✓ bundles     : {bundles}")

    # 只有声明了 dsh.bundle.patch 的包才会成为 profile 的一层
    # 没声明的会被 DSH 提示 "declares no dsh.bundle"，属于正常（就是个普通依赖）
    new = set(deps) - set(before.get("dependencies", {}))
    print(f"  新增依赖: {sorted(new)}")

    # 复验"安装区确实在 D 盘"（我们的迁移目标）
    nm = base / "node_modules"
    if not is_link(nm):
        print("  ⚠ profile 的 node_modules 不是链接 —— 新插件可能装到 C 盘了")
    return True


def probe_routes(urls: list[str]) -> None:
    """运行时验证：探测插件注册的 HTTP 路由（比"文件存在"强得多的证据）。"""
    import urllib.request
    for u in urls:
        try:
            with urllib.request.urlopen(u, timeout=10) as resp:
                body = resp.read(200).decode("utf-8", "replace")
                print(f"  OK  {resp.status}  {u}  {body}")
        except Exception as e:
            print(f"  ✗   {u}  -> {e}")


# ============================================================================
# 任务二：搬迁目录（插件安装区 / 插件数据）
# ============================================================================

def migrate_dir(name: str, src: Path, dst_parent: Path) -> bool:
    """
    把一个目录搬到 D 盘，原位置留一个链接。六步走，任何一步失败就回滚。

    这一步等价于：
        C:\\Users\\...\\.dsh\\<name>   ──[Junction]──►   D:\\deepseek插件\\...\\<name>
    插件照原路径读写，完全感觉不到差别。
    """
    if name in CANNOT_MOVE:
        print(f"  ✗ 跳过 {name}：{CANNOT_MOVE[name]}（核心数据，不建议搬）")
        return False
    if name in NEVER_MOVE_INSTALL_AREA:
        print(f"  ✗ 跳过 {name}：这是插件的安装区，且该插件依赖"
              f"「向上查找兄弟目录」来解析宿主核心包，"
              f"搬了会导致启动时入口加载失败。")
        return False

    dst = dst_parent / name
    bak = src.with_name(f"{src.name}.old-{os.getpid()}")
    print(f"\n[2] 搬迁 {name}\n    源: {src}\n    目标: {dst}")

    if not src.exists():
        print("  ✗ 源目录不存在"); return False
    if is_link(src):
        print("  · 已经是链接，跳过"); return False

    # 1) 复制
    if dst.exists() or is_link(dst):
        print(f"  · 目标已存在，先清掉：{dst}")
        if not DRY_RUN:
            if is_link(dst):
                remove_junction(dst)              # 是链接就只删链接，绝不 rmtree
            else:
                shutil.rmtree(dst, ignore_errors=True)
    robocopy_copy(src, dst)

    # 2) 校验（文件数必须一致，否则说明有长路径被跳过）
    n_src, mb_src = dir_stat(src)
    n_dst, mb_dst = dir_stat(dst)
    print(f"  校验: 源 {n_src} 文件 / {mb_src} MB    目标 {n_dst} 文件 / {mb_dst} MB")
    if n_src != n_dst:
        print("  ✗ 文件数不一致 —— 可能有超长路径被跳过，放弃并清理")
        if not DRY_RUN:
            shutil.rmtree(dst, ignore_errors=True)
        return False

    # 3) 改名留底（不删！这样出问题能一键还原）
    if DRY_RUN:
        print(f"  [dry-run] 改名 {src} -> {bak}")
    else:
        os.rename(src, bak)

    # 4) 建链接
    try:
        make_junction(src, dst)
    except Exception as e:
        print(f"  ✗ 建链接失败，回滚：{e}")
        if not DRY_RUN:
            os.rename(bak, src)
            shutil.rmtree(dst, ignore_errors=True)
        return False

    # 5) 透过链接读一遍，确认真的通
    n_via, _ = dir_stat(src)
    print(f"  透过链接可见: {n_via} 文件")
    if n_via != n_src:
        print("  ✗ 链接读到的内容不对，回滚")
        if not DRY_RUN:
            remove_junction(src)
            os.rename(bak, src)
            shutil.rmtree(dst, ignore_errors=True)
        return False

    # 6) 删留底
    if not DRY_RUN:
        shutil.rmtree(bak, ignore_errors=True)
    print(f"  ✓ 完成：{name}")
    return True


def migrate_all_data() -> None:
    """把所有"可搬"的插件数据目录搬到 D 盘。"""
    for name in DATA_DIRS:
        migrate_dir(name, DSH_HOME / name, DATA_SUBDIR)
    print("\n提示：搬完请重启应用，或至少让相关插件重新读一次数据，再确认功能正常。")


def migrate_pnpm_store() -> None:
    """
    把 pnpm 的全局 store 也挪到 D 盘。
    注意坑：只改配置不重建 node_modules，之后装包会报
    ERR_PNPM_UNEXPECTED_STORE —— 因为 pnpm 把 store 路径记在
    node_modules/.modules.yaml 里做一致性校验，而 pnpm install 会误判"已最新"。
    """
    print("\n[3] 迁移 pnpm store")
    old = Path(os.environ["LOCALAPPDATA"]) / "pnpm" / "store"
    robocopy_copy(old, STORE_DIR)
    pnpm(["config", "set", "store-dir", str(STORE_DIR)], cwd=DSH_HOME)

    # 关键：让 pnpm 真的重建 node_modules，否则后续装包会 store 不一致
    nm = DSH_HOME / "profiles" / PROFILE / "node_modules"
    print(f"  → 重建 {nm}（清空后重装，让 .modules.yaml 记下新的 store）")
    if not DRY_RUN and nm.exists():
        for item in nm.iterdir():
            if is_link(item):
                remove_junction(item)          # 链接：只删链接
            elif item.is_dir():
                shutil.rmtree(item, ignore_errors=True)
            else:
                item.unlink(missing_ok=True)
    pnpm(["install"], cwd=DSH_HOME / "profiles" / PROFILE)


# ============================================================================
# 任务三：现状体检 / 生成说明
# ============================================================================

def status() -> None:
    """只读体检：现在东西都在哪、哪些搬了、哪些还在 C 盘。"""
    print("=" * 66)
    print("DSH 插件现状体检")
    print("=" * 66)

    nm = DSH_HOME / "profiles" / PROFILE / "node_modules"
    print(f"\nprofile '{PROFILE}' 的安装区:")
    if is_link(nm):
        print(f"  {nm}\n    ──[链接]──► {link_target(nm)}")
    else:
        print(f"  {nm}  (实体目录，仍在 C 盘)")

    print(f"\n~/.dsh 下的目录:")
    for p in sorted(DSH_HOME.iterdir()):
        if not p.is_dir():
            continue
        n, mb = dir_stat(p)
        if is_link(p):
            print(f"  [链接] {p.name:<22} {n:>5} 文件 {mb:>8} MB  -> {link_target(p)}")
        else:
            tag = "核心" if p.name in CANNOT_MOVE else "实体"
            print(f"  [{tag}] {p.name:<22} {n:>5} 文件 {mb:>8} MB")

    print(f"\n{PLUGIN_DIR} 结构:")
    for p in sorted(PLUGIN_DIR.iterdir()):
        print(f"  {'[目录]' if p.is_dir() else '[文件]'} {p.name}")


def write_readme() -> None:
    """生成给"未来的自己"看的说明文件（UTF-8 BOM，记事本打开不乱码）。"""
    lines = [
        f"{PLUGIN_DIR} —— 插件与插件数据总目录",
        "=" * 50, "",
        "【目录说明】", "",
        "node_modules\\            插件本体（插件市场装的插件都在这里；pnpm 规定名，不能改）",
        "插件数据\\                插件运行时数据（原在 C 盘，现搬到 D 盘并在原位置留链接）",
    ]
    for name, desc in DATA_DIRS.items():
        n, mb = dir_stat(DATA_SUBDIR / name)
        lines.append(f"    {name:<22} {mb:>8} MB   {desc}")
    lines += [
        ".pnpm-store\\             pnpm 的下载缓存",
        "dsh-tui.cmd              启动 dsh-TUI 终端的启动器",
        "说明.txt                 本文件",
        "", "-" * 50,
        "【仍然留在 C 盘的（核心数据，不建议搬）】", "-" * 50,
    ]
    for name, desc in CANNOT_MOVE.items():
        n, mb = dir_stat(DSH_HOME / name)
        lines.append(f"  {name:<22} {mb:>8} MB   {desc}")
    lines += [
        f"  profiles\\dsh-tui\\node_modules   约 53 MB   见下",
        "", "-" * 50,
        "【为什么 dsh-tui 的安装区不能搬到 D 盘】", "-" * 50,
        "该插件在代码里直接 import 宿主的 @deepseek-ai/* 核心包。",
        "它待在 profiles\\dsh-tui\\node_modules 时，Node 会沿目录向上找到",
        "profiles\\node_modules\\@deepseek-ai（共享回退区）从而成功；",
        "一旦搬到 D 盘，这条向上查找的路径断了，启动时 8 个入口 failed to import。",
        "而把 @deepseek-ai 补进 profile 自己的 node_modules，又会让 DSH 把宿主的",
        "核心插件行（llm、session 等）判为版本不兼容并禁用。两条路都不通。",
        "", "-" * 50, "【重要提醒】", "-" * 50,
        "1. 不要改名 / 移动 / 删除 node_modules\\ 和 插件数据\\，它们都被链接指着。",
        "2. 不要移动或改名整个 D:\\deepseek插件 文件夹，也不要让 D 盘不可用。",
        "3. 在市场里装 / 卸插件会自动反映到 node_modules\\，不用手动管。",
        "4. 插件的设置与数据仍按原路径读写，只是背后真实落在 D 盘。",
    ]
    text = "\n".join(lines) + "\n"
    out = PLUGIN_DIR / "说明.txt"
    if DRY_RUN:
        print(f"[dry-run] 会写入 {out}（{len(text)} 字符）")
        return
    out.write_text(text, encoding="utf-8-sig")     # BOM：记事本友好
    print(f"已写入 {out}")


# ============================================================================
# main
# ============================================================================

def main() -> int:
    global DRY_RUN
    ap = argparse.ArgumentParser(description="DSH 插件安装 + 数据搬迁工具")
    ap.add_argument("--dry-run", action="store_true", help="只打印计划，不动文件")
    sub = ap.add_subparsers(dest="cmd", required=True)

    sub.add_parser("status", help="现状体检（只读）")

    p_ins = sub.add_parser("install", help="装一个插件")
    p_ins.add_argument("spec", help="包名 / 本地路径 / github:owner/repo")
    p_ins.add_argument("--profile", default=PROFILE)
    p_ins.add_argument("--allow", action="append", default=[],
                       help="放行的精确版本，如 @deepseek-ai/dsh-agent-presets@0.1.1-rc.2")

    p_mv = sub.add_parser("move", help="搬一个插件数据目录")
    p_mv.add_argument("name")

    sub.add_parser("move-all", help="搬所有可搬的插件数据目录")
    sub.add_parser("move-store", help="把 pnpm store 也搬到 D 盘")
    sub.add_parser("readme", help="重新生成 说明.txt")

    args = ap.parse_args()
    DRY_RUN = args.dry_run

    if args.cmd == "status":
        status()
    elif args.cmd == "install":
        ok = install_plugin(args.spec, args.profile, args.allow)
        return 0 if ok else 1
    elif args.cmd == "move":
        ok = migrate_dir(args.name, DSH_HOME / args.name, DATA_SUBDIR)
        return 0 if ok else 1
    elif args.cmd == "move-all":
        migrate_all_data()
    elif args.cmd == "move-store":
        migrate_pnpm_store()
    elif args.cmd == "readme":
        write_readme()
    return 0


if __name__ == "__main__":
    sys.exit(main())
