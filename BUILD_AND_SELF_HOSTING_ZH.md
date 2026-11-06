# Mindustry 源码、编译与自行架设指南

本文依据本仓库当前的 `master` 分支（提交 `3a5481351355735bfe1a2e7e48a5baaa81d214d0`，核对日期：2026-09-23）以及官方文档整理。后续版本可能调整 JDK、Android SDK 或命令，请同时查看项目根目录的 [`README.md`](README.md) 和 [Mindustry 官方服务器文档](https://mindustrygame.github.io/wiki/servers/)。

## 一、客户端和服务器端是否全部开源？

结论：**Mindustry 自身的客户端、共享游戏逻辑和专用服务器源码都已公开，可以自行编译和架设；它不是传统的“网页前端 + Web API 后端”系统。**

仓库采用 [GNU GPL v3](LICENSE)。GitHub 官方仓库也将项目标记为 GPL-3.0：<https://github.com/Anuken/Mindustry>。

| 通常所说的部分 | 本项目中的模块 | 是否在仓库中 |
| --- | --- | --- |
| 桌面客户端/界面 | `desktop/` + `core/` | 是 |
| Android 客户端 | `android/` + `core/` | 是 |
| iOS 客户端 | `ios/` + `core/` | 是 |
| 游戏规则、网络协议及公共逻辑 | `core/` | 是 |
| 无界面专用服务器 | `server/` + `core/` | 是 |
| 注解处理器、资源生成工具及测试 | `annotations/`、`tools/`、`tests/` | 是 |

因此，运行多人游戏不需要另找一个未公开的官方“后端”。专用服务器是独立的 headless Java 程序，客户端直接连接它。服务器状态保存在本地文件中，不依赖 MySQL、Redis 或浏览器管理后台。

“全部开源”仍需注意以下边界：

- 仓库自己的代码受 GPL v3 约束；修改后向他人分发二进制时，应一并满足 GPL 对许可证声明和对应源码提供等要求。本文不是法律意见。
- Gradle 下载的 Arc、Rhino、JLine 等第三方依赖各有自己的许可证，不应把根许可证理解成对所有第三方软件重新授权。
- JDK、Android SDK、Xcode、Apple 签名证书、Google Play/Steam 等工具或发行服务不属于本仓库。它们的使用条款和开源状态需分别判断。
- “源码可自行编译和运行”不等于自动取得应用商店账号、Steam 服务、官方服务器列表收录或项目名称/标识的额外商标权利。

## 二、项目结构和运行关系

```text
桌面客户端 ─┐
Android 客户端 ─┼── 使用 core（游戏内容、规则、网络协议）── 连接专用服务器
iOS 客户端 ───┘                                      server + core
```

自行对外架设时只需运行 `server-release.jar`。玩家使用相同或兼容版本的桌面/移动客户端，通过服务器域名或 IP 和端口连接。

## 三、编译前准备

### 1. 必需软件

- Git。
- **JDK 17**。官方 `README.md` 明确要求 JDK 17；不要让 Gradle 误用 JDK 8、11 或其他版本。
- 可访问 Maven Central、Google Maven 和 JitPack 的网络。首次构建需要下载 Gradle 和依赖。
- 建议至少预留 8 GiB 内存和数 GiB 磁盘空间；仓库的 `gradle.properties` 将 Gradle 最大堆设为 8 GiB。

先验证：

```bash
java -version
```

输出应显示 17。如果机器安装了多个 JDK，应在当前终端设置 `JAVA_HOME`。

Windows PowerShell 示例：

```powershell
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17"
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
java -version
```

Linux/macOS 示例（路径按实际安装位置调整）：

```bash
export JAVA_HOME=/path/to/jdk-17
export PATH="$JAVA_HOME/bin:$PATH"
java -version
```

### 2. 获取源码

```bash
git clone https://github.com/Anuken/Mindustry.git
cd Mindustry
```

若要制作与某个正式版本严格对应的客户端和服务器，建议切到同一个发布标签。例如：

```bash
git fetch --tags
git checkout v160.5
```

直接编译 `master` 得到的是开发中版本，不保证能和正式版客户端联机。服务器和客户端最好从同一提交一起构建。

项目在找不到同级目录的 Arc 源码时，会从依赖仓库下载 Arc。这是正常情况。如果 JitPack 不稳定，可按 `settings.gradle` 的提示，将兼容版本的 <https://github.com/Anuken/Arc> 克隆到与 `Mindustry` 同级的 `Arc` 目录；不要随意混用不匹配的 Arc 提交。

## 四、编译桌面客户端和服务器

### Windows

```powershell
.\gradlew.bat desktop:dist server:dist
```

### Linux/macOS

```bash
chmod +x ./gradlew
./gradlew desktop:dist server:dist
```

首次桌面构建可能同时执行资源/精灵图打包，耗时会比后续构建长。成功后产物为：

- 桌面客户端：`desktop/build/libs/Mindustry.jar`
- 专用服务器：`server/build/libs/server-release.jar`

分别运行：

```bash
java -jar desktop/build/libs/Mindustry.jar
java -jar server/build/libs/server-release.jar
```

只想在源码目录内开发运行时，可用：

```bash
# Windows
.\gradlew.bat desktop:run

# Linux/macOS
./gradlew desktop:run
```

如果需要在生成的版本信息中写入构建号，可在检出对应标签后传入属性，例如：

```bash
./gradlew desktop:dist server:dist -Pbuildversion=160.5
```

不要给修改过或不对应的代码冒用官方构建号。省略该参数时会显示 `custom build`；源码也会提示，自定义构建会关闭常规的客户端版本检查，所以更应主动保证客户端与服务器来自兼容提交。

### 本仓库上的实测结果

在 Windows 11、Temurin JDK `17.0.19` 和当前提交上运行以下命令已成功：

```powershell
.\gradlew.bat server:dist desktop:dist --no-daemon
```

实测得到的两个 JAR 均位于上述路径。默认环境若仍把 `JAVA_HOME` 指向 JDK 8，必须先切换到 JDK 17。

## 五、编译 Android 客户端（可选）

除 JDK 17 外，还需 Android SDK。当前 `android/build.gradle` 使用：

- `compileSdk` / `targetSdk`：36
- Build Tools：36.0.0
- 最低 Android API：21

用 Android command-line tools 安装组件并接受许可证：

```bash
sdkmanager "platform-tools" "platforms;android-36" "build-tools;36.0.0"
sdkmanager --licenses
```

设置 `ANDROID_HOME` 指向 Android SDK 根目录，然后编译调试包：

```bash
# Windows
.\gradlew.bat android:assembleDebug

# Linux/macOS
./gradlew android:assembleDebug
```

APK 位于 `android/build/outputs/apk/` 下。连接已启用开发者模式/USB 调试的设备后，可运行：

```bash
./gradlew android:installDebug android:run
```

发布版 APK 需要自己的 keystore。不要使用他人的签名材料。可从 `android/build.gradle` 查看 `RELEASE_STORE_FILE`、`RELEASE_STORE_PASSWORD`、`RELEASE_KEY_ALIAS` 和 `RELEASE_KEY_PASSWORD` 等构建属性。

## 六、iOS 构建说明（可选）

iOS 启动代码和构建配置也在仓库中，但它不是普通 Windows/Linux 环境下的常规构建目标。当前模块使用 RoboVM；实际生成可安装 IPA 还需要 macOS、Xcode、Apple Developer 证书和 provisioning profile，并遵守 Apple 的签名/分发要求。

相关 Gradle 目标为 `ios:createIPA`，签名参数名可在 `ios/build.gradle` 中查看。由于官方根 `README.md` 没有给出完整、受支持的 iOS 发布流程，建议仅在熟悉 Apple 签名体系时构建；这不影响桌面客户端或专用服务器的编译和自行架设。

## 七、最简专用服务器架设

### 1. 准备独立目录

把编译出的 JAR 复制到独立运行目录并改成简短名称。不要直接在源码目录里长期运行生产服务器。

Linux 示例：

```bash
sudo install -d -m 0755 /opt/mindustry
sudo cp server/build/libs/server-release.jar /opt/mindustry/server.jar
cd /opt/mindustry
java -Xms512m -Xmx2g -jar server.jar
```

Windows 示例：

```powershell
New-Item -ItemType Directory -Force C:\MindustryServer
Copy-Item .\server\build\libs\server-release.jar C:\MindustryServer\server.jar
Set-Location C:\MindustryServer
java -Xms512m -Xmx2g -jar server.jar
```

内存参数只是适合小型服务器的起点，应按地图、模组和人数调整。

### 2. 首次配置并开服

服务启动后，在控制台逐条输入：

```text
help
config name 我的服务器
config desc 欢迎游玩
config port 6567
config autosave true
config autosaveSpacing 300
config messageRateLimit 1
config showConnectMessages false
playerlimit 20
maps all
host
```

`host` 会以生存模式选择地图。指定地图和模式的语法为：

```text
host 地图名 survival
```

地图名含空格时，控制台列出的名称通常可用下划线代替空格。其他常用命令：

```text
status                 # 查看状态
players                # 查看在线玩家
save manual-backup     # 手动保存
stop                   # 停止当前对局但不退出进程
exit                   # 正常保存配置并退出
config                 # 列出全部服务器配置
help <命令>            # 查看某个命令帮助
```

希望进程启动后自动开服，可在首次交互配置时设置：

```text
config startCommands host
```

也可直接把启动命令作为 JAR 参数：

```bash
java -Xms512m -Xmx2g -jar server.jar host
```

以上两种自动开服方式选择一种即可，不要同时配置。多个启动命令以英文逗号分隔；复杂命令建议先在交互控制台验证。

### 3. 数据目录

专用服务器把数据写入**当前工作目录下的 `config/`**。其中会逐步出现：

- `config/maps/`：自定义地图；复制后在控制台执行 `reloadmaps`。
- `config/mods/`：模组/服务器插件；一般需重启以加载。
- `config/saves/`：手动存档和自动存档。
- `config/logs/`：日志。
- `config/rules.hjson`：全局规则。
- 其他设置、玩家管理、封禁和白名单数据。

备份时停止服务器或先执行 `save`，然后备份整个 `config/` 目录。升级前同时备份当前 `server.jar`，这样出现不兼容时可以回滚程序与数据。

## 八、网络和防火墙

默认游戏端口是 **6567**，必须同时放行：

- TCP 6567
- UDP 6567

Linux 使用 UFW 的示例：

```bash
sudo ufw allow 6567/tcp
sudo ufw allow 6567/udp
```

如果使用云主机，还要在云厂商安全组中同时放行 TCP/UDP 6567。如果服务器位于家庭路由器后，还需把两个协议的 6567 端口转发到服务器的局域网 IP；运营商 CGNAT 环境可能无法直接做公网入站转发。

玩家在“加入游戏”中填写：

```text
服务器公网 IP:6567
```

局域网内测试应先使用服务器的局域网 IP。公网测试最好从手机流量或另一条外部网络进行。仅用 TCP 端口检测网站不能证明 UDP 已经可用。

如修改端口，例如：

```text
config port 7000
```

则防火墙、安全组、路由器端口转发和客户端地址都必须同步改成 7000。官方服务器文档同样说明默认端口需放行 TCP 和 UDP：<https://mindustrygame.github.io/wiki/servers/>。

## 九、Linux 使用 systemd 常驻运行

先创建不可登录的专用用户，并让它拥有运行目录：

```bash
sudo useradd --system --home /opt/mindustry --shell /usr/sbin/nologin mindustry
sudo chown -R mindustry:mindustry /opt/mindustry
```

新建 `/etc/systemd/system/mindustry.service`：

```ini
[Unit]
Description=Mindustry Dedicated Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mindustry
Group=mindustry
WorkingDirectory=/opt/mindustry
ExecStart=/usr/bin/java -Xms512m -Xmx2g -jar /opt/mindustry/server.jar host
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

加载、启动并查看日志：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mindustry
sudo systemctl status mindustry
sudo journalctl -u mindustry -f
```

systemd 模式不便交互输入，因此应先在命令行完成持久配置，再启用服务。需要执行管理命令时，最简单稳妥的做法是安排维护窗口，停止 systemd 服务后以前台方式启动并操作。服务器也提供默认关闭的本地命令 socket，但若启用，务必只绑定 `localhost`，不要向公网开放。

上面的 unit 已通过命令行参数 `host` 自动开服，因此不要再设置 `config startCommands host`。如果希望完全依赖持久化的 `startCommands`，则从 `ExecStart` 末尾删除 `host`。

更新流程：

```bash
sudo systemctl stop mindustry
# 备份 /opt/mindustry/config 和旧 server.jar
# 用同版本线的新 JAR 替换 /opt/mindustry/server.jar
sudo chown mindustry:mindustry /opt/mindustry/server.jar
sudo systemctl start mindustry
sudo journalctl -u mindustry -n 100 --no-pager
```

## 十、Docker 运行（可选）

官方仓库当前未提供 Dockerfile；下面是基于已编译 `server-release.jar` 的最小示例，不代表官方镜像。

把 JAR 放进一个空目录，并命名为 `server.jar`。在同目录创建：

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /opt/mindustry
COPY server.jar /opt/mindustry/server.jar
EXPOSE 6567/tcp 6567/udp
ENTRYPOINT ["java", "-Xms512m", "-Xmx2g", "-jar", "/opt/mindustry/server.jar"]
CMD ["host"]
```

`compose.yaml`：

```yaml
services:
  mindustry:
    build: .
    restart: unless-stopped
    stdin_open: true
    tty: true
    ports:
      - "6567:6567/tcp"
      - "6567:6567/udp"
    volumes:
      - ./data:/opt/mindustry/config
```

启动和看日志：

```bash
docker compose up -d --build
docker compose logs -f
```

首次需要交互配置时，可临时用 `docker compose up` 前台启动，或使用 `docker attach`；从 attach 状态脱离而不停止容器通常使用 `Ctrl+P`、`Ctrl+Q`。Dockerfile 的 `CMD ["host"]` 已负责自动开服，不要再重复设置 `config startCommands host`。务必持久化 `/opt/mindustry/config`，否则重建容器会丢失配置和存档。

## 十一、常见问题

### `JAVA_HOME is set to an invalid directory` 或版本不是 17

修正 `JAVA_HOME` 后重新打开终端，确认 `java -version` 与 Gradle 使用的 JVM 都是 17：

```bash
./gradlew --version
```

查看输出中的 `Launcher JVM` / `Daemon JVM`。

### Linux/macOS 提示 `Permission denied`

```bash
chmod +x ./gradlew
```

### 找不到 `mindustry.gen`

这是构建期通过注解处理器和资源生成任务产生的代码，不需要也不应手工创建或编辑。先让 Gradle 完成一次构建。

### 依赖从 JitPack 下载失败

先重试并检查网络/代理；仍失败时，按本指南前文和 `settings.gradle` 的说明准备与当前 Mindustry 提交兼容的同级 `Arc` 源码目录。

### 局域网能连，公网不能连

依次检查云安全组、系统防火墙、路由器 TCP/UDP 端口转发、公网 IP 和 CGNAT。服务端控制台执行 `status`，并确认进程确实在运行。

### 客户端显示版本不兼容

使用同一正式发布标签重新构建客户端和服务器，或直接使用同一 Git 提交的两端产物。不要把 `master` 开发版服务端与商店正式版客户端随意混用。

## 十二、上线检查清单

- [ ] 客户端和服务器来自同一标签/兼容提交。
- [ ] 构建与运行实际使用 JDK 17。
- [ ] `config name`、描述、人数、自动保存和管理规则已设置。
- [ ] TCP/UDP 6567 均已在系统防火墙与云安全组放行。
- [ ] 家庭网络已完成两种协议的端口转发，或使用具备公网入站能力的主机。
- [ ] 已从外部网络实际连接测试。
- [ ] `config/` 有定期异机备份。
- [ ] 更新前备份数据和旧 JAR，并保持客户端/服务器版本一致。
- [ ] 未把命令 socket、管理终端或敏感文件暴露到公网。

## 参考资料

- 官方源码、许可证与基础编译命令：<https://github.com/Anuken/Mindustry>
- 官方专用服务器说明、命令和端口：<https://mindustrygame.github.io/wiki/servers/>
- 官方发行版：<https://github.com/Anuken/Mindustry/releases>
- 官方持续构建产物：<https://github.com/Anuken/MindustryBuilds/releases>
