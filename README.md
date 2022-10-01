# Gitea backup

An Ansible role to backup your Kubernetes-based Gitea instance to an S3 bucket or SFTP server.

## Requirements

* The role must be configured to access Kubernetes, and exec into the Gitea pod.
* The `kubectl` command must be installed and executable by the user running the role.
* The `s3cmd` command must be installed and executable by the user running the role.
* The `scp` command must be installed and executable by the user running the role.
* The `ssh` command must be installed and executable by the user running the role.

Please see [s3cmd's documentation](https://s3tools.org/download) on how to install the command.

## Dependencies

The following collections must be installed:

* cloud.common
* amazon.aws
* community.general
* community.kubernetes

## Role Variables

This role requires one dictionary as configuration, `gitea_backup`:

```yaml
    gitea_backup:
      s3cmd: "/usr/local/bin/s3cmd"
      debug: true
      stopOnFailure: false
      sources: {}
      remotes: {}
      backups: []
```

Where:
* `s3cmd` is the full path to the `s3cmd` executable. Optional, defaults to `s3cmd`.
* `debug` is `true` to enable debugging output. Optional, defaults to `false`.
* `stopOnFailure` is `true` to stop the entire role if any one backup fails. Optional, defaults to `false`.
* `sources` is a dictionary of sites and environments. Required.
* `remotes` is a dictionary of remote upload locations. Required.
* `backups` is a list of backups to perform. Required.

### Specifying Sources

In this role, "sources" specify the source from which to download backups. Each must have a unique key which is later used in the `gitea_backup.backups` list.

```yaml
gitea_backup:
  sources:
    my-gitea:
      namespace: "gitea"
      containerName: "gitea"
      lablelSelector: "app=gitea"
      retryCount: 3
      retryDelay: 30
```

Where, in each entry:
* `host` is the hostname of the database server. 
* `port` is the port with which to connect to the database.
* `namespace` is the namespace in which Gitea is running. Required.
* `containerName` is the container name in which Gitea is running. Required.
* `lablelSelector` is the Kubernetes label select with which to find the Gitea pod. Required.
* `retryCount` is the number of times to attempt the backup before failing. Optional, defaults to `1`.
* `retryDelay` is the number of seconds to wait between backup attemps. Optional, defaults to `30`.

### Specifying remotes

In this role "remotes" are upload destinations for backups. This role supports S3 or SFTP as remotes. Each remote must have a unique key which is later used in the `gitea_backup.backups` list.

```yaml
    - hosts: servers
      vars:
        gitea_backup:
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              hostBucket: "my-example-bucket.s3.example.com"
              s3Url: "https://my-example-bucket.s3.example.com"
              region: "us-east-1"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
```

For `s3` type remotes:
* `bucket` is the name of the S3 bucket. 
* `accessKeyFile` is the path to a file containing the access key. Optional if `accessKey` is specified.
* `accessKey` is the value of the access key necessary to access the bucket. Ignored if `accessKeyFile` is specified.
* `secretKeyFile` is the path to a file containing the secret key. Optional if `secretKey` is specified.
* `secretKey` is the value of the access key necessary to secret the bucket. Ignored if `secretKeyFile` is specified.
* `region` is the AWS region in which the bucket resides. Required if using AWS S3, may be optional for other providers.
* `endpoint` is the S3 endpoint to use. Optional if using AWS, required for other providers.

For `sftp` type remotes:
* `host` is the hostname of the SFTP server. Required.
* `user` is the username necessary to login to the SFTP server. Required.
* `keyFile` is the path to a file containing the SSH private key. Required.
* `pubKeyFile` si the path to a file containing the SSH public key. Required for database backups, ignored for file backups.

### Specifying backups

The `gitea_backup.backups` list specifies the database backups perform, referencing the `gitea_backup.sources` and `gitea_backup.remotes` sections for connectivity details.

```yaml
gitea_backup:
  backups:
    - name: "gitea.example.com"
      source: "my-gitea"
      prefix: "gitea-dump"
      format: "tar.gz"
      disabled: false
      targets: []
```

Where:
* `name` is the display name of the backup. Optional, but makes the logs easier.
* `source` is the name of the key under `gitea_backups.sources` from which to generate the backup. Required.
* `prefix` is the prefix of the dump file to generate. Avoid special characters except for `-`. Optional, defaults to `gitea-backup`.
* `format` is the format of the Gitea dump. Optional, defaults to `tar.gz`.
* `disabled` is `true` to disable (skip) the backup. Optional, defaults to `false`.
* `targets` is a list of remotes and additional destination information about where to upload backups. Required.

### Backup targets

Backup targets reference a key in `gitea_backup.remotes`, and combine that with additional information used to upload this specific backup.

```yaml
gitea_backup:
  backups:
    - name: "gitea.example.com"
      source: "my-gitea"
      prefix: "gitea-dump"
      format: "tar.gz"
      disabled: false
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/gitea-dumps"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/gitea-dumps"
          disabled: false
```

Where:
* `remote` is the key under `gitea_backup.remotes` to use when uploading the backup. Required.
* `path` is the path on the remote to upload the backup. Optional.
* `disabled` is `true` to skip uploading to the specifed `remote`. Optional, defaults to `false`.

### Ping URL on completion

When a backup completes, you have the option to ping an URL via HTTP:

```yaml
gitea_backup:
  backups:
    - name: "gitea.example.com"
      source: "my-gitea"
      prefix: "gitea-dump"
      format: "tar.gz"
      healthcheckUrl: "https://pings.example.com/path/to/service"
      disabled: false
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/gitea-dumps"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/gitea-dumps"
          disabled: false
```

Where:
* `healthcheckUrl` is the URL to ping when the backup completes successfully. Optional.

### Rotating backups

Backups are uploaded to the remote with the `&lt;prefix&gt;-0.sql.gz`. Often, you'll want to retain previous backups in the case an older backup can aid in research or recovery. This role supports retaining and rotating multiple backups using the `retainCount` key.

```yaml
pantheon_backup:
  backups:
    - name: "gitea.example.com"
      source: "my-gitea"
      prefix: "gitea-dump"
      format: "tar.gz"
      healthcheckUrl: "https://pings.example.com/path/to/service"
      disabled: false
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/gitea-dumps"
          retainCount: 3
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/gitea-dumps"
          retainCount: 3
          disabled: false
```

Where:
* `retainCount` is the total number of backups to retain in the directory. Optional. Defaults to `1`, or no rotation.

During a backup, if `retainCount` is set:
1. The backup with the ending `&lt;retainCount - 1&gt;.tar.gz` is deleted.
2. Starting with `&lt;retainCount - 2&gt;.tar.gz`, each backup is renamed incremending the ending index.
3. The new backup is uploaded with a `0` index as `&lt;prefix&gt;-0.sql.gz`.

This feature works both in S3 and SFTP.

## Example Playbook

```yaml
    - hosts: servers
      vars:
        gitea_backup:
          sources:
            my-gitea:
              namespace: "gitea"
              containerName: "gitea"
              lablelSelector: "app=gitea"
              retryCount: 3
              retryDelay: 30
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              provider: "AWS"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              hostBucket: "my-example-bucket.s3.example.com"
              s3Url: "https://my-example-bucket.s3.example.com"
              region: "us-east-1"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
          backups:
            - name: "gitea.example.com"
              source: "my-gitea"
              prefix: "gitea-dump"
              format: "tar.gz"
              healthcheckUrl: "https://pings.example.com/path/to/service"
              disabled: false
              targets:
                - remote: "example-s3-bucket"
                  path: "example.com/gitea-dumps"
                  retainCount: 3
                  disabled: true
                - remote: "sftp.example.com"
                  path: "backups/example.com/gitea-dumps"
                  retainCount: 3
                  disabled: false
      roles:
         - { role: ten7.gitea_backup }
```

## Docker Container

If you prefer to run this role via a Docker container, see (https://hub.docker.com/repository/docker/ten7/gitea-backup)[ten7/gitea-backup] on Docker Hub and (https://github.com/ten7/gitea-backup-docker)[on github].

## Helm chart

To run this role from Kubernetes itself, see (https://github.com/ten7/gitea-backup-helm)[github.com/ten7/gitea-backup-helm].

## License

GPL v3

## Author Information

This role was created by [TEN7](https://ten7.com/).
