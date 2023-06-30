# memos on fly

English | [中文](README_zh-CN.md)

> Run the self-hosted memo service [memos](https://github.com/usememos/memos) on [fly.io](https://fly.io/). Automatically backup the database to [B2](https://www.backblaze.com/b2/cloud-storage.html) with [litestream](https://litestream.io/).

🙏 Thanks for [linkding-on-fly](https://github.com/fspoettel/linkding-on-fly), the project is inspired by it. If you want to deploy memos with litestream function locally, please visit [hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream)

## Prerequisites

  - [fly.io](https://fly.io/) account
  - [Backblaze](https://www.backblaze.com/) account or other B2 service account 
  - [Optional] *If you want to build your own docker image, clone repository from [hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream).* 
  
### ⚠️ **WARNING**
[hu3rror/memos-on-fly-build](https://github.com/hu3rror/memos-on-fly-build) has been deprecated and the maintenance is moved to [hu3rror/memos-litestream](https://github.com/hu3rror/memos-litestream)

If you have used this image before, you can simply change the build image section of your fly.toml to the new image like this:

```diff
[build]
-  image = "hu3rror/memos-fly:latest"
+  image = "ghcr.io/hu3rror/memos-litestream:latest"
```

The new image is universal for both fly.io and local runs~

## Installation

1. Follow [the instructions](https://fly.io/docs/getting-started/installing-flyctl/) to install fly's command-line interface `flyctl`.
2. [log into flyctl](https://fly.io/docs/getting-started/log-in-to-fly/).
  ```sh
  flyctl auth login
  ```

## Launch a fly application

> **do not** setup Postgres and **do not** deploy yet!

  ```sh
  flyctl launch
  ```

This command creates a `fly.toml` file. 

## Edit your `fly.toml`

You can take [fly.example.toml](fly.example.toml) in this repository as a reference and modify according to the comments.

### Details of manual modifications

#### 1. Add an `build` section.

  ```toml
  [build]
    image = "ghcr.io/hu3rror/memos-litestream:latest"
  ```

#### 2. Add an `env` section.

  ```toml
  [env]
    LITESTREAM_REPLICA_BUCKET = "<filled_later>"  # change to your litestream bucket name
    LITESTREAM_REPLICA_ENDPOINT = "<filled_later>"  # change to your litestream endpoint url
    LITESTREAM_REPLICA_PATH = "memos_prod.db"  # keep the default or change to whatever path you want
  ```

#### 3. Configure litestream backups

  > ℹ️ If you want to use another storage provider, check litestream's ["Replica Guides"](https://litestream.io/guides/) section and adjust the config as needed.

  1. Log into B2 and [create a bucket](https://litestream.io/guides/backblaze/#create-a-bucket). Instead of adjusting the litestream config directly, we will add storage configuration to `fly.toml`. 
  2. Now you can set the values of `LITESTREAM_REPLICA_ENDPOINT` and `LITESTREAM_REPLICA_BUCKET` to your `[env]` section.
  3. Then, create [an access key](https://litestream.io/guides/backblaze/#create-a-user) for this bucket. Add the key to fly's secret store (Don't add `<` and `>`).
      ```sh
      flyctl secrets set LITESTREAM_ACCESS_KEY_ID="<keyId>" LITESTREAM_SECRET_ACCESS_KEY="<applicationKey>"
      ```

#### 4. Add a persistent volume

  1. Create a [persistent volume](https://fly.io/docs/reference/volumes/). Fly's free tier includes `3GB` of storage across your VMs. Since `memos` is very light on storage, a `1GB` volume will be more than enough for most use cases. It's possible to change volume size later. A how-to can be found in the _"scale persistent volume"_ section below.
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

If all is well, you can now access memos by running `flyctl open`. You should see its login page.

## All done!

🎊 Just enjoy using memos!

## Other

### How to update to the latest memos release

You can check the [status](https://github.com/hu3rror/memos-on-fly/actions/workflows/build-and-push-release-image.yml) of memos's docker image built by GitHub Actions. 

If the latest docker image has been released to Docker Hub, you can upgrade memos easily by `flyctl deploy` in your project's folder.

### Custom Domains

If you wish, you can [configure a custom domain for your install](https://fly.io/docs/app-guides/custom-domains-with-fly/).

### Verify the installation

 - you should be able to log into your memos instance.
 - there should be an initial replica of your database in your B2 bucket.
 - your user data should survive a restart of the VM.

#### Verify backups / scale persistent volume

Litestream continuously backs up your database by persisting its [WAL](https://en.wikipedia.org/wiki/Write-ahead_logging) to B2, once per second.

There are two ways to verify these backups:

 1. run the docker image locally or on a second VM. Verify the DB restores correctly.
 2. swap the fly volume for a new one and verify the DB restores correctly.

We will focus on _2_ as it simulates an actual data loss scenario. This procedure can also be used to scale your volume to a different size.

Start by making a manual backup of your data:

 1. ssh into the VM and copy the DB to a remote. If only you are using your instance, you can also export bookmarks as HTML.
 2. make a snapshot of the B2 bucket in the B2 admin panel.

Now list all fly volumes and note the id of the `memos_data` volume. Then, delete the volume.

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

[^0]: otherwise the VM is ~$2 per month. $0.15/GB per month for the persistent volume.'
[^1]: the first 10GB are free, then $0.005 per GB.

## Troubleshooting

### litestream is logging 403 errors

Check that your B2 secrets and environment variables are correct.

### fly ssh does not connect

Check the output of `flyctl doctor`, every line should be marked as **PASSED**. If `Pinging WireGuard` fails, try `flyctl wireguard reset` and `flyctl agent restart`.

### fly does not pull in the latest version of memos

Just run `flyctl deploy --no-cache`

### Memos Telegram bot does not respond

This is due to the fly.toml v2 update, you can modify your fly.toml like:

```toml
[http_service]
  auto_stop_machines = false   # change to `false` if you use telegram bot
```
