# 在 fly.io 上部署 Memos 官方镜像

## 先决条件

- 拥有 fly.io 账户，可访问 https://fly.io/ （需要有效信用卡）。

## 安装步骤

1. 按照 https://fly.io/docs/getting-started/installing-flyctl/ 上的指南安装 flyctl 命令行工具。
2. 使用以下命令登录 flyctl：

```sh
flyctl auth login
```

3. 使用以下命令克隆存储库：

```shell
git clone https://github.com/hu3rror/memos-on-fly
```

4. 将 `fly_no_litestream_example.toml` 文件重命名为 `fly.toml` 并填写或修改必要的值。
5. 使用以下命令启动和初始化应用程序：

```shell
flyctl launch
```

6. 在提示 `Would you like to copy its configuration to the new app?` 时，键入 `Y` 以确认。
7. 输入一个独特且不冲突的应用程序名称。
8. 当要求设置 Postgresql 数据库或 Upstash Redis 数据库时，键入 `N` 以拒绝，因为它们对此应用程序不必要。
9. 检查 `fly.toml` 中的部署配置，如果正确，键入 `Y` 以部署应用程序；否则，键入 `N` 后再对文件进行编辑。
10. 最后，请使用以下命令部署应用程序：

```shell
flyctl deploy
```

11. 等待几分钟，应用程序部署完成后，访问 `https://<your-app-name>.fly.dev`。也可以使用以下命令在浏览器中打开应用程序：

```shell
flyctl open
```

## 一些建议：

- 如果只想使用一台机器部署，请使用以下命令（注意：不建议此操作）：

```shell
flyctl deploy --ha=false
```

- 如果要增加应用程序的内存（MiB），请使用以下命令：

```shell
flyctl scale memory 512
```

希望这些信息对您有所帮助！
