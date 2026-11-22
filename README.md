# Stellex Loader

Stellex 是基于 NeoForge 21.1.251 源码的加载器分支：保留全部包名与二进制接口，在内核层加入 CPU 调度器、内存治理与客户端渲染剔除。安装器与发行包都在本仓库内，客户端下载不再依赖官方 NeoForge 仓库。

## 下载与安装

服务端发行包：`download/stellex-loader-1.0.0-server.zip`（Git LFS 存储）。

两种安装方式：

1. 使用安装器：`java -jar download/stellex-loader-installer.jar --installServer <目录>`。安装器内嵌本仓库构建的 universal 包。
2. 使用下载器：运行 `download/stellex-installer.jar`，或 `java -jar download/stellex-installer.jar --install <目录>`。下载器从本仓库拉取服务端包并解压。

安装完成后在目录内写入 `eula.txt`，用 JDK 21 启动 `run.bat`（Windows）或 `run.sh`。

## 内核功能

CPU 调度器：`-Dstellex.cpu.target=60`、`-Dstellex.cpu.threads=6`、`-Dstellex.tps.guard.ms=45`、`-Dstellex.pregen.radius=32`。命令 `/stellex status`、`/stellex target <1-100>`、`/stellex radius <4-256>`、`/stellex pause`、`/stellex resume`。

内存治理：`-Dstellex.memory.soft.percent=75`、`-Dstellex.memory.critical.percent=90`、`-Dstellex.memory.dump.on.critical=false`。命令 `/stellex memory`、`/stellex memory clean`、`/stellex memory dump`。

客户端剔除：`config/stellex-client.properties`，默认实体 128 格、方块实体 96 格、统计间隔 1200 tick。

## 兼容性

- 纯 NeoForge 21.1.251 客户端可以进入 Stellex 服务端。
- Stellex 客户端可以加载 NeoForge 模组；模组的必需通道仍按 NeoForge 规则校验。
- 加载器没有注册任何必需的自定义网络通道。

## 文档

论文见 `docs/PAPER.md` 与 `docs/PAPER.docx`，包含构建流程、内核实现、实验数据与兼容性验证。

## 许可

NeoForge 源码遵循 LGPL-2.1；本仓库的修改与新增部分沿用同一许可证。作者：Mornfrost Office。
