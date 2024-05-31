# Deploy Memos Official Image on fly.io

## Prerequisites

- An account on fly.io, accessible at https://fly.io/ (A valid credit card is required).

## Installation

1. Install the flyctl Command-Line Interface (CLI) by referring to the guide at https://fly.io/docs/getting-started/installing-flyctl/.
2. Log in to flyctl using the command:

```sh
flyctl auth login
```

3. Clone the repository using the following command:

```shell
git clone https://github.com/hu3rror/memos-on-fly
```

4. Rename the `fly_no_litestream_example.toml` file to `fly.toml` and input or modify the necessary values.
5. Launch and initialize the application with the command:

```shell
flyctl launch
```

6. When prompted with `Would you like to copy its configuration to the new app?`, type `Y` to confirm.
7. Choose a unique and non-conflicting name for the app.
8. When asked to set up a Postgresql database or an Upstash Redis database, type `N` to decline, as they are not required for this app.
9. Review the deployment configuration in `fly.toml`. If correct, type `Y` to deploy the app; otherwise, type `N` to edit the file.
10. If modifications were made to the `fly.toml` file, deploy the app using:

```shell
flyctl deploy
```

11. Wait a few minutes for the app to deploy, then access it at `https://<your-app-name>.fly.dev`. Alternatively, open the app in your browser with the command:

```shell
flyctl open
```

## Specific Suggestions:

- To deploy with only one machine, use the following command (Note: This is not recommended):

```shell
flyctl deploy --ha=false
```

- To scale up the memory (MiB) for the app, use the command:

```shell
flyctl scale memory 512
```

We hope this information proves beneficial.
