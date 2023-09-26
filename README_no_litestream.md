# Deploy Memos official image to fly.io

## Prerequisites

* A fly.io: https://fly.io/ account (Credit card required).

## Installation

1. Install the flyctl CLI: https://fly.io/docs/getting-started/installing-flyctl/.
2. Log in to flyctl: https://fly.io/docs/getting-started/log-in-to-fly/.

```sh
flyctl auth login
```

3. Clone this repository:

```shell
git clone https://github.com/hu3rror/memos-on-fly
```

4. Rename the `fly.no.litestream.example.toml` file to `fly.toml` and fill in or modify the required values.
5. Launch and init the app:

```shell
flyctl launch
```

6. When prompted `Would you like to copy its configuration to the new app?` to copy the configuration to the new app, type `Y` to confirm.
7. Choose an app name. The name must be unique and non-conflicting.
8. When prompted to set up a Postgresql database or an Upstash Redis database, type `N` to confirm (these are no needed for this app).
9. Review the deployment configuration in `fly.toml`. If it is correct, type `Y` to deploy the app. Otherwise, type `N` to edit the file.
10. If you edited the `fly.toml` file, deploy the app:

```shell
flyctl deploy
```

11. Wait a few minutes for the app to deploy, then visit it at `https://<your-app-name>.fly.dev`. You can also open the app in your browser by running the following command:

```shell
flyctl open
```

## Some specific suggestions:

- If you want to deploy with only one machine, you can use the following command (Note: This is not recommended):

```shell
flyctl deploy --ha=false
```

- If you want to scale up the memory (MiB) for the app, you can use the following command:
```shell
flyctl scale memory 512
```