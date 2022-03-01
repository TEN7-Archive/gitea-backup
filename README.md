# Gitea Backup for Kubernetes

 This helm chart provides a recurring, automated backup for [Gitea](https://gitea.io/en-us/) installations on Kubernetes to S3 and SFTP.

## How it works

This chart provides a container which remotely executes `gitea dump` on the gitea container itself. Then uses `kubectl cp` to move the resulting file to itself, before uploading it to one or more target destinations. A serviceAccount, role, and rolebinding are created by the chart to grant access.

## Features

* Uses Gitea's own command to backup Gitea.
* Works even if the persistent volumes backing Gitea are `ReadWriteOnce`.
* Supports multiple S3 and SFTP destinations.
* A number of backups can be retained, and old backups are deleted automatically.
* Can be installed multiple times in the same namespace, with different schedules, allowing you to make hourly/daily/weekly/monthly backups.

## Requirements

* Gitea must be installed in the same namespace as this chart, ideally using the official [Gitea Helm Chart](https://gitea.com/gitea/helm-chart).
* The `gitea dump` command is available in the Gitea container.
* Connection and credentials for target S3 and/or SFTP providers.

## Installation

```shell
helm repo add gitea-backup https://ten7.github.io/gitea-backup/
helm repo update
helm upgrade --install gitea-backup gitea-backup/gitea-backup --namespace=gitea -f path/to/my-values.yml
```

## Configuration

For a full list of values, see [values.yaml](https://raw.githubusercontent.com/ten7/gitea-backup/main/charts/gitea-backup/values.yaml).

## License

Gitea Backup is licensed under GPLv3. See [LICENSE](https://raw.githubusercontent.com/ten7/gitea-backup/main/LICENSE) for the complete language.
