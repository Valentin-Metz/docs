# Copying snapshots between repositories

In case you want to transfer snapshots between two repositories, for example
from a local to a remote repository, you can use the `copy` command:

```console
$ rustic -r /srv/rustic-repo copy --repo2 /srv/rustic-repo-copy
repository d6504c63 opened successfully, password is correct
repository 3dd0878c opened successfully, password is correct

snapshot 410b18a2 of [/home/user/work] at 2020-06-09 23:15:57.305305 +0200 CEST)
    copy started, this may take a while...
snapshot 7a746a07 saved

snapshot 4e5d5487 of [/home/user/work] at 2020-05-01 22:44:07.012113 +0200 CEST)
skipping snapshot 4e5d5487, was already copied to snapshot 50eb62b7
```

The example command copies all snapshots from the source repository
`/srv/rustic-repo` to the destination repository `/srv/rustic-repo-copy`.
Snapshots which have previously been copied between repositories will be skipped
by later copy runs.

**Important**: This process will have to both download (read) and upload (write)
the entire snapshot(s) due to the different encryption keys used in the source
and destination repository. This *may incur higher bandwidth usage and costs*
than expected during normal backup runs.

**Important**: The copying process does not re-chunk files, which may break
deduplication between the files copied and files already stored in the
destination repository. This means that copied files, which existed in both the
source and destination repository, *may occupy up to twice their space* in the
destination repository. See below for how to avoid this.

The destination repository is specified with `--repo2` or can be read from a
file specified via `--repository-file2`. Both of these options can also set as
environment variables `$RESTIC_REPOSITORY2` or `$RESTIC_REPOSITORY_FILE2`
respectively. For the destination repository the password can be read from a
file `--password-file2` or from a command `--password-command2`. Alternatively
the environment variables `$RESTIC_PASSWORD_COMMAND2` and
`$RESTIC_PASSWORD_FILE2` can be used. It is also possible to directly pass the
password via `$RESTIC_PASSWORD2`. The key which should be used for decryption
can be selected by passing its ID via the flag `--key-hint2` or the environment
variable `$RESTIC_KEY_HINT2`.

**Note**: In case the source and destination repository use the same backend,
the configuration options and environment variables used to configure the
backend may apply to both repositories – for example it might not be possible to
specify different accounts for the source and destination repository. You can
avoid this limitation by using the rclone backend along with remotes which are
configured in rclone.
