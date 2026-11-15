# Droidspaces 软件包构建（Fedora 45 GNOME）

为 [Droidspaces RootFS](../) 构建 **Fedora 45 + GNOME（Anland Wayland）** 所需的 ARM64 预编译包，共两条流水线。

本仓库只保留这两条流水线；KDE、Niri、Hyprland、Mango、Denial、Anland Next、systemd 257、Hangover Wine、TUI 发布的工作流与脚本已移除。

## 1. Anland GNOME（Mutter + Xwayland）

工作流：`.github/workflows/build-anland-packages.yml`

- 输入：`anland_repo`（默认 `https://github.com/wzk0/anland.git`）、`anland_ref`（默认 `legacy`）、`gnome_target`（`Fedora45`）、`publish_cnb`（默认关闭）
- 在原生 ARM64 runner 上用 `fedora:45` 容器重建 Fedora 当前 Mutter SRPM（51.x），套用 Anland 补丁并重打 Xwayland
- 产物发布到滚动 Release **`anland-gnome-packages`**：
  - `anland-gnome-fedora45-mutter-<版本>-aarch64.tar.gz`
  - `anland-gnome-manifest`（平台 → 资产映射，含 sha256）
  - `install-anland-gnome.sh`

配方来源：`wzk0/anland` 的 `producers/gnome/Fedora45_v5`（该 fork 可删除，配方已备份为本仓库的 `anland-fedora45-recipe.patch`）
（Mutter 补丁采用与 mutter 51 对齐的版本；`rpm -K` 使用 `--nosignature`，因为 Fedora 45 的
`%_pkgverify_level` 为 `all`，未签名的本地产物会被判签名失败）。

## 2. Mesa 分发

工作流：`.github/workflows/distribute-mesa.yml`

把上游 [`lfdevs/mesa-for-android-container`](https://github.com/lfdevs/mesa-for-android-container)
的正式 Release 资产重新分发到本仓库的固定 Release **`mesa-for-android-container`**，共 8 个平台
（archlinux / debian_trixie / fedora_43 / fedora_44 / **fedora_45** / ubuntu_noble / ubuntu_questing / ubuntu_resolute），
并附带 `msm_drm_drv_video.so` 视频解码驱动、`install-mesa.sh` 以及 `mesa-versions-manifest`。

上游最新 Release 已包含 `fedora_45` 资产，直接运行即可；上游发新版本时手动重跑一次。

## 重建 Anland 包（配方备份）

Anland 配方不是本仓库的代码，而是打在 [SuperTurtleDev/anland](https://github.com/SuperTurtleDev/anland)
`legacy` 分支上的两个提交。仓库里备份为 `anland-fedora45-recipe.patch`，任何人可在没有
`wzk0/anland` 的情况下复原：

```bash
git clone -b legacy https://github.com/SuperTurtleDev/anland.git my-anland
cd my-anland
git am /path/to/anland-fedora45-recipe.patch
# 推到自己账号后，用 Actions 里的 anland_repo / anland_ref 指过去
```

配方内容：`producers/gnome/Fedora45_v5`（复制 Fedora44_v5，版本门改 45，Mutter 补丁换成与
mutter 51 对齐的版本，spec 注入 `gnome-shell >= 51, < 52`）+ `rpm -K --nosignature` 修复。

## 运行方式

Actions → 选择对应工作流 → `Run workflow`（分支必须是 `main`）。两者相互独立：

1. 先跑 **构建 Anland GNOME（Fedora 45）** 拿到 Mutter 包；
2. 需要 GPU 支持时跑 **分发 Mesa for Android Container**；
3. 然后在 Droidspaces RootFS 仓库里构建镜像，把 `wayland_package_repository` 指向本仓库。
