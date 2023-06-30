# memos on fly.io

[English](README.md) | 中文

> 在[fly.io](https://fly.io/)上运行自托管的 [memos](https://github.com/usememos/memos)，并附有 [litestream](https://litestream.io/)自动备份数据库到你个人的 S3 / [Backblaze B2](https://www.backblaze.com/b2/cloud-storage.html)。

🙏 感谢 [memos](https://github.com/usememos/memos) 和 [linkding-on-fly](https://github.com/fspoettel/linkding-on-fly)，本项目受其启发！如果你想要在本地搭建（包含 Litestream 功能），请参考 [hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream)。

## 前提

  - [fly.io](https://fly.io/) 帐户
  - [Backblaze](https://www.backblaze.com/)帐户或其他 S3 服务帐户
  - [可选] 如果要构建自己的Docker镜像，请从[hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream) 克隆存储库，找到最新的 release 的 commit，按你所需自行构建

### ⚠️ **警告**
[hu3rror/memos-on-fly-build](https://github.com/hu3rror/memos-on-fly-build) 已被弃用，新的项目镜像维护更新将被转移到 [hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream)

如果你以前使用过这个镜像，你可以简单地在你的 fly.toml 中的 build image 部分中删除旧的镜像，并改为新的镜像，像这样：

```diff
[build]
- image = "hu3rror/memos-fly:latest"
+ image = "ghcr.io/hu3rror/memos-litestream:latest"
```

## 安装指南

1. 按照[说明](https://fly.io/docs/getting-started/installing-flyctl/)安装 fly 的命令行界面 `flyctl`。
2. 按照[说明](https://fly.io/docs/getting-started/log-in-to-fly/) 登录 flyctl。
    ```sh
    flyctl auth login
    ```

## 初始化 fly app

> 注意⚠️： **不要** 设置 Postgres，**不要** 立即部署！

  ```sh
  flyctl launch
  ```

该命令将创建一个 `fly.toml` 文件。

## 编辑 `fly.toml`

您可以参考本存储库中的 [fly.example.toml](fly.example.toml) 并根据注释内容的提示进行修改。

### 手动修改的详细信息

#### 1. 添加 `build` 部分。

  ```toml
  [build]
    image = "ghcr.io/hu3rror/memos-litestream:latest"
  ```

#### 2. 添加 `env` 部分。

  ```toml
  [env]
    LITESTREAM_REPLICA_BUCKET = "<filled_later>"  # 更改为您的 litestream bucket 名称
    LITESTREAM_REPLICA_ENDPOINT = "<filled_later>"  # 更改为您的 litestream ENDPOINT 网址
    LITESTREAM_REPLICA_PATH = "memos_prod.db"  # 建议保持默认值
  ```

#### 3. 配置 litestream 备份

> ℹ️ 如果您想使用其他存储提供程序，请检查 litestream 的 ["Replica Guides"](https://litestream.io/guides/) 部分并根据需要调整配置。

  1. 登录 B2 并[创建一个bucket](https://litestream.io/guides/backblaze/#create-a-bucket)。
  2. 将 `LITESTREAM_REPLICA_ENDPOINT` 和 `LITESTREAM_REPLICA_BUCKET` 的值设置到您的 `[env]` 部分。
  3. 为此 bucket 创建[访问密钥](https://litestream.io/guides/backblaze/#create-a-user)。将密钥添加到fly的密钥存储中（⚠️ 不包含 `<` 和 `>`）。
      ```sh
      flyctl secrets set LITESTREAM_ACCESS_KEY_ID="<keyId>" LITESTREAM_SECRET_ACCESS_KEY="<applicationKey>"
      ```

#### 4. 添加持久卷

  1. 创建[持久卷](https://fly.io/docs/reference/volumes/)。fly.io 的免费套餐包含 `3GB`存储空间，而`memos` 若采用纯文本+外挂图床的方式，则所需要的存储空间非常少，大多数情况下`1GB`的卷已经足够了。您也可以在稍后更改卷的大小。如何更改可以在下面的_"scale persistent volume"_部分找到。
      ```sh
      flyctl volumes create memos_data --region <your_region> --size <size_in_gb>
      ```
      例子：
        ```sh
        flyctl volumes create memos_data --region hkg --size 1
        ```

  2. 在 `fly.toml` 添加 `mounts` 部分，将持久卷附加到容器。
      ```toml
      [[mounts]]
        source = "memos_data"
        destination = "/var/opt/memos"
      ```

#### 5. 将 `internal_port` 添加到 `[[services]]` 中

```toml
[http_service]
  internal_port = 5230   # change to 5230
```

#### 6. 部署到 fly.io

  ```sh
  flyctl deploy
  ```

如果一切正常，您现在可以通过运行 `flyctl open`来访问 memos。您应该会看到它的登录页面。

## 完成了！

🎊 尽情使用 memos 吧！

## 其他

### 如何更新到最新的 memos 版本

如果最新的 docker 镜像已经发布到 Docker Hub，您可以通过在项目文件夹中运行`flyctl deploy`来轻松升级memos。

### 自定义域名

如果您希望，您可以参考 [fly.io 自定义域名](https://fly.io/docs/app-guides/custom-domains-with-fly/)。

### 验证安装

 - 您应该能够登录到您的 memos 实例。
 - 在您的B2存储桶浏览一下，应该可以看到一个数据库的初始副本。
 - 您的用户数据应该在 VM 重新启动后仍然存在。

#### 验证备份/扩展持久卷

Litestream 通过将其 [WAL（Write-ahead logging）](https://en.wikipedia.org/wiki/Write-ahead_logging) 持久化到B2上的备份来实现数据持久化。如果您想验证备份是否正常工作，可以手动模拟故障并恢复。

1. 在 fly.io 的控制台中，使用 "Stop" 按钮停止您的应用程序。
2. 更改您的 `memos` 数据库中的某些数据。
3. 使用 "Start" 按钮重新启动应用程序。
4. 检查您的更改是否仍然存在。如果是，则意味着已经成功数据持久化。

如果您想测试持久卷的扩展，请按照以下步骤操作：

1. 使用 `flyctl volumes resize` 命令来增加持久卷的大小。
2. 使用 SSH 连接到您的 fly.io VM，然后运行以下命令来扩展文件系统。
   ```sh
   cd memos-on-fly
   fly ssh console
   sudo resize2fs /dev/vda1
   ```
   注意：`/dev/vda1`是文件系统所在的设备名称。如果您的设备名称不同，请相应地更改此命令。

3. 检查文件系统是否已成功扩展。您可以运行`df -h`命令来查看文件系统的容量和使用情况。如果一切正常，您应该看到文件系统的可用空间已经增加了。

### Memos Telegram 机器人不响应，不回复

由于 fly.toml 更新到 v2 版本，默认开启 auto_stop_machines，你可以手动修改 fly.toml:

```toml
[http_service]
  auto_stop_machines = false   # 修改为 `false`，如果你使用 telegram bot
```
