# Memos Deployment Guide

English | [中文](README_zh-CN.md)

Deploy the self-hosted memo service, [memos](https://github.com/usememos/memos), on [fly.io](https://fly.io/). Automate database backups to [B2](https://www.backblaze.com/b2/cloud-storage.html) using [litestream](https://litestream.io/).

Acknowledgments to [linkding-on-fly](https://github.com/fspoettel/linkding-on-fly) for inspiring this project.

## IMPORTANT NOTICE

To deploy memos without litestream on fly.io (Memos Official images), refer to [README_no_litestream](README_no_litestream.md) for instructions. Skip the remainder of this README document.

## Prerequisites

- [fly.io](https://fly.io/) account
- [Backblaze](https://www.backblaze.com/) account or another B2 service account
- [Optional] If building your docker image, clone the repository from [hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream).

### ⚠️ **WARNING**

[hu3rror/memos-on-fly-build](https://github.com/hu3rror/memos-on-fly-build) is deprecated. Maintenance has moved to [hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream).

If you previously used this image, change the build image section in your fly.toml to the new image:

```diff
[build]
-  image = "hu3rror/memos-fly:latest"
+  image = "ghcr.io/hu3rror/memos-litestream:stable"
```

The new image is universal for both fly.io and local runs.

## Installation

1. Follow [the instructions](https://fly.io/docs/getting-started/installing-flyctl/) to install fly's command-line interface `flyctl`.
2. [Log into flyctl](https://fly.io/docs/getting-started/log-in-to-fly/).

```sh
flyctl auth login
```

## Launch a fly application

> **Do not** set up Postgres and **do not** deploy yet!

```sh
flyctl launch
```

This command creates a `fly.toml` file.

## Edit your `fly.toml`

Use [fly.with.litestream.example.toml](fly.with.litestream.example.toml) in this repository as a reference and modify according to the comments.

### Details of manual modifications

#### 1. Add a `build` section.

```toml
[build]
  image = "ghcr.io/hu3rror/memos-litestream:stable"
```

#### 2. Add an `env` section.

```toml
[env]
  LITESTREAM_REPLICA_BUCKET = "<filled_later>"  # change to your litestream bucket name
  LITESTREAM_REPLICA_ENDPOINT = "<filled_later>"  # change to your litestream endpoint URL
  LITESTREAM_REPLICA_PATH = "memos_prod.db"  # keep the default or change to whatever path you want
```

#### 3. Configure litestream backups

> ℹ️ If you want to use another storage provider, check litestream's ["Replica Guides"](https://litestream.io/guides/) section and adjust the config as needed.

1. Log into B2 and [create a bucket](https://litestream.io/guides/backblaze/#create-a-bucket). Instead of adjusting the litestream config directly, add storage configuration to `fly.toml`.
2. Set the values of `LITESTREAM_REPLICA_ENDPOINT` and `LITESTREAM_REPLICA_BUCKET` to your `[env]` section.
3. Create [an access key](https://litestream.io/guides/backblaze/#create-a-user) for this bucket. Add the key to fly's secret store (Don't add `<` and `>`).

```sh
flyctl secrets set LITESTREAM_ACCESS_KEY_ID="<keyId>" LITESTREAM_SECRET_ACCESS_KEY="<applicationKey>"
```

#### 4. Add a persistent volume

1. Create a [persistent volume](https://fly.io/docs/reference/volumes/). Fly's free tier includes `3GB` of storage across your VMs. Since `memos` is very light on storage, a `1GB` volume will be more than enough for most use cases. Change the volume size later if needed. Find instructions in the _"scale persistent volume"_ section below.

```sh
flyctl volumes create memos_data --region <your_region> --size <size_in_gb>
```

For example:

```sh
flyctl volumes create memos_data --region hkg --size 1
```

2. Attach the persistent volume to the container by adding a `mounts` section to `fly.toml`.

```toml
[[mounts]]
  source = "memos_data"
  destination = "/var/opt/memos"
```

#### 5. Add `internal_port` to `[[services]]`

```toml
[http_service]
  internal_port = 5230   # change to 5230
```

#### 6. Deploy to fly.io

```sh
flyctl deploy
```

If all is well, access memos by running `flyctl open`. You should see its login page.

## All done!

🎊 Enjoy using memos!

## Other

### How to update to the latest memos release

Check the [status](https://github.com/hu3rror/memos-on-fly/actions/workflows/build-and-push-release-image.yml) of memos's docker image built by GitHub Actions.

If the latest docker image is on Docker Hub, upgrade memos with `flyctl deploy` in your project's folder.

### Custom Domains

If desired, [configure a custom domain for your install](https://fly.io/docs/app-guides/custom-domains-with-fly/).

### Verify the installation

- Log into your memos instance.
- Find an initial replica of your database in your B2 bucket.
- Confirm your user data survives a VM restart.

#### Verify backups / scale persistent volume

Litestream continuously backs up your database by persisting its [WAL](https://en.wikipedia.org/wiki/Write-ahead_logging) to B2, once per second.

Two ways to verify backups:

1.  Run the docker image locally or on a second VM. Verify the DB restores correctly.
2.  Swap the fly volume for a new one and verify the DB restores correctly.

Focus on _2_ as it simulates an actual data loss scenario. This procedure can also scale your volume to a different size.

Start by making a manual backup of your data:

1.  SSH into the VM and copy the DB to a remote. If only you use your instance, export bookmarks as HTML.
2.  Make a snapshot of the B2 bucket in the B2 admin panel.

List all fly volumes and note the id of the `memos_data` volume. Then, delete the volume.

```sh
flyctl volumes list
flyctl volumes delete <id>
```

This will result in a **dead** VM after a few seconds. Create a new `memos_data` volume. Your application should automatically attempt to restart. If not, restart it manually.

When the application starts, you should see the successful restore in the logs.

```
[info] No database found, attempt to restore from a replica.
[info] Finished restoring the database.
[info] Starting litestream & memos service.
```

### Pricing

Assuming one 256MB VM and a 3GB volume, this setup fits within Fly's free tier. [^0] Backups with B2 are free as well. [^1]

[^0]: Otherwise, the VM is ~$2 per month. $0.15/GB per month for the persistent volume.'
[^1]: The first 10GB are free, then $0.005 per GB.

## Troubleshooting

### Litestream is logging 403 errors

Check that your B2 secrets and environment variables are correct.

### `fly ssh console` does not connect

Check the output of `flyctl doctor`. Every line should be marked as **PASSED**. If `Pinging WireGuard` fails, try `flyctl wireguard reset` and `flyctl agent restart`.

### Fly does not pull in the latest version of memos

Run `flyctl deploy --no-cache`.

### Memos Telegram bot does not respond

Due to the fly.toml v2 update, modify your fly.toml like:

```toml
[http_service]
  auto_stop_machines = false   # change to `false` if you use the Telegram bot
```
