## 将 Memos 官方镜像部署到 fly.io

### 前提条件

* 一个 fly.io 账户（需要信用卡）。

### 安装

1. 安装 flyctl CLI：https://fly.io/docs/getting-started/installing-flyctl/ (必要时可借助网页翻译工具)。
2. 登录 flyctl：https://fly.io/docs/getting-started/log-in-to-fly/ (必要时可借助网页翻译工具)。

```sh
flyctl auth login
```

3. 克隆此存储库：

```shell
git clone https://github.com/hu3rror/memos-on-fly
```

4. 将 `fly.no.litestream.example.toml` 文件重命名为 `fly.toml` 并填写或修改其中的内容。
5. 初始化应用程序：

```shell
flyctl launch
```

6. 在提示 `Would you like to copy its configuration to the new app?` 时，请输入 `Y` 确认。
7. 之后会提示输入你的应用程序名称。名称必须是唯一的且不冲突。
8. 在提示设置 Postgresql 数据库和 Upstash Redis 时，请输入 `N` 确认（此应用程序不需要这些数据库）。
9. 查看 `fly.toml` 中的部署配置。如果没有问题，可输入 `Y` 直接开始部署应用程序。否则，请输入 `N`，先正确编辑该文件。
10. 如果您已经完成编辑了 `fly.toml` 文件，可部署应用程序：

```shell
flyctl deploy
```

11. 等待几分钟让应用程序部署完毕，然后在 `https://<your-app-name>.fly.dev` 访问它。您也可以通过运行以下命令在浏览器中打开应用程序：

```shell
flyctl open
```

### 一些具体建议

- 如果您只想在 fly.io 上只使用一台机器部署 (V2默认会新建两个机器machines)，可以使用以下命令（注意：不推荐）：

```shell
flyctl deploy --ha=false
```

- 如果您想扩大应用程序的内存（MiB），可以使用以下命令：

```shell
flyctl scale memory 512
```

希望这对你有帮助！~