# Working with repositories

<!-- TOC -->

- [Listing snapshots](#listing-snapshots)
- [Copying snapshots between repositories](#copying-snapshots-between-repositories)
- [Filtering snapshots to copy](#filtering-snapshots-to-copy)
- [Ensuring deduplication for copied snapshots](#ensuring-deduplication-for-copied-snapshots)
- [Checking integrity and consistency](#checking-integrity-and-consistency)
- [Upgrading the repository format version](#upgrading-the-repository-format-version)

<!-- /TOC -->

A repository is a storage location for all of your snapshots.

The repository is created with the `init` command:

```console
$ rustic -r /srv/rustic-repo init
enter password for new repository:
enter password again:
created rustic repository 7a8c3b2a0c at /srv/rustic-repo
Please note that knowledge of your password is required to access
the repository. Losing your password means that your data is
irrecoverably lost.
```

The repository is now ready for use.

**Note**: In case you are using the rclone backend, please see the
:ref:`rclone-backend` section for additional information.

## Listing snapshots

To list all snapshots in the repository, use the `snapshots` command:

```console
$ rustic -r /srv/rustic-repo snapshots
enter password for repository:
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
40dc1520  2015-05-08 21:38:30  kasimir        /home/user/work
79766175  2015-05-08 21:40:19  kasimir        /home/user/work
bdbd3439  2015-05-08 21:45:17  luigi          /home/art
590c8fc8  2015-05-08 21:47:38  kazik          /srv
9f0bc19e  2015-05-08 21:46:11  luigi          /srv
```

You can filter the listing by directory path:

````console
$ rustic -r /srv/rustic-repo snapshots --path="/srv"
enter password for repository:
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
590c8fc8

## Listing all snapshots

Now, you can list all the snapshots stored in the repository:

```console
$ rustic -r /srv/rustic-repo snapshots
enter password for repository:
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
40dc1520  2015-05-08 21:38:30  kasimir        /home/user/work
79766175  2015-05-08 21:40:19  kasimir        /home/user/work
bdbd3439  2015-05-08 21:45:17  luigi          /home/art
590c8fc8  2015-05-08 21:47:38  kazik          /srv
9f0bc19e  2015-05-08 21:46:11  luigi          /srv
````

You can filter the listing by directory path:

```console
$ rustic -r /srv/rustic-repo snapshots --path="/srv"
enter password for repository:
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
590c8fc8  2015-05-08 21:47:38  kazik          /srv
9f0bc19e  2015-05-08 21:46:11  luigi          /srv
```

Or filter by host:

```console
$ rustic -r /srv/rustic-repo snapshots --host luigi
enter password for repository:
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
bdbd3439  2015-05-08 21:45:17  luigi          /home/art
9f0bc19e  2015-05-08 21:46:11  luigi          /srv
```

Combining filters is also possible.

Furthermore you can group the output by the same filters (host, paths, tags):

```console
$ rustic -r /srv/rustic-repo snapshots --group-by host

enter password for repository:
snapshots for (host [kasimir])
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
40dc1520  2015-05-08 21:38:30  kasimir        /home/user/work
79766175  2015-05-08 21:40:19  kasimir        /home/user/work
2 snapshots
snapshots for (host [luigi])
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
bdbd3439  2015-05-08 21:45:17  luigi          /home/art
9f0bc19e  2015-05-08 21:46:11  luigi          /srv
2 snapshots
snapshots for (host [kazik])
ID        Date                 Host    Tags   Directory
----------------------------------------------------------------------
590c8fc8  2015-05-08 21:47:38  kazik          /srv
1 snapshots
```

## Copying snapshots between repositories

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

## Filtering snapshots to copy

The list of snapshots to copy can be filtered by host, path in the backup and /
or a comma-separated tag list:

```console
rustic -r /srv/rustic-repo copy --repo2 /srv/rustic-repo-copy --host luigi --path /srv --tag foo,bar
```

It is also possible to explicitly specify the list of snapshots to copy, in
which case only these instead of all snapshots will be copied:

```console
rustic -r /srv/rustic-repo copy --repo2 /srv/rustic-repo-copy 410b18a2 4e5d5487 latest
```

## Ensuring deduplication for copied snapshots

Even though the copy command can transfer snapshots between arbitrary
repositories, deduplication between snapshots from the source and destination
repository may not work. To ensure proper deduplication, both repositories have
to use the same parameters for splitting large files into smaller chunks, which
requires additional setup steps. With the same parameters rustic will for both
repositories split identical files into identical chunks and therefore
deduplication also works for snapshots copied between these repositories.

The chunker parameters are generated once when creating a new (destination)
repository. That is for a copy destination repository we have to instruct rustic
to initialize it using the same chunker parameters as the source repository:

```console
rustic -r /srv/rustic-repo-copy init --repo2 /srv/rustic-repo --copy-chunker-params
```

Note that it is not possible to change the chunker parameters of an existing
repository.

## Checking integrity and consistency

Imagine your repository is saved on a server that has a faulty hard drive, or
even worse, attackers get privileged access and modify the files in your
repository with the intention to make you restore malicious data:

```console
echo "boom" > /srv/rustic-repo/index/de30f3231ca2e6a59af4aa84216dfe2ef7339c549dc11b09b84000997b139628
```

Trying to restore a snapshot which has been modified as shown above will yield
an error:

```console
$ rustic -r /srv/rustic-repo --no-cache restore c23e491f --target /tmp/restore-work
...
Fatal: unable to load index de30f323: load <index/de30f3231c>: invalid data returned
```

In order to detect these things before they become a problem, it's a good idea
to regularly use the `check` command to test whether your repository is healthy
and consistent, and that your precious backup data is unharmed. There are two
types of checks that can be performed:

- Structural consistency and integrity, e.g. snapshots, trees and pack files
  (default)
- Integrity of the actual data that you backed up (enabled with flags, see
  below)

To verify the structure of the repository, issue the `check` command. If the
repository is damaged like in the example above, `check` will detect this and
yield the same error as when you tried to restore:

```console
$ rustic -r /srv/rustic-repo check
...
load indexes
error: error loading index de30f323: load <index/de30f3231c>: invalid data returned
Fatal: LoadIndex returned errors
```

If the repository structure is intact, rustic will show that no errors were
found:

```console
$ rustic -r /src/rustic-repo check
...
load indexes
check all packs
check snapshots, trees and blobs
no errors were found
```

By default, the `check` command does not verify that the actual pack files on
disk in the repository are unmodified, because doing so requires reading a copy
of every pack file in the repository. To tell rustic to also verify the
integrity of the pack files in the repository, use the `--read-data` flag:

```console
$ rustic -r /srv/rustic-repo check --read-data
...
load indexes
check all packs
check snapshots, trees and blobs
read all data
[0:00] 100.00%  3 / 3 items
duration: 0:00
no errors were found
```

**Note**: Since `--read-data` has to download all pack files in the repository,
beware that it might incur higher bandwidth costs than usual and also that it
takes more time than the default `check`.

Alternatively, use the `--read-data-subset` parameter to check only a subset of
the repository pack files at a time. It supports three ways to select a subset.
One selects a specific part of pack files, the second and third selects a random
subset of the pack files by the given percentage or size.

Use `--read-data-subset=n/t` to check a specific part of the repository pack
files at a time. The parameter takes two values, `n` and `t`. When the check
command runs, all pack files in the repository are logically divided in `t`
(roughly equal) groups, and only files that belong to group number `n` are
checked. For example, the following commands check all repository pack files
over 5 separate invocations:

```console
rustic -r /srv/rustic-repo check --read-data-subset=1/5
rustic -r /srv/rustic-repo check --read-data-subset=2/5
rustic -r /srv/rustic-repo check --read-data-subset=3/5
rustic -r /srv/rustic-repo check --read-data-subset=4/5
rustic -r /srv/rustic-repo check --read-data-subset=5/5
```

Use `--read-data-subset=x%` to check a randomly choosen subset of the repository
pack files. It takes one parameter, `x`, the percentage of pack files to check
as an integer or floating point number. This will not guarantee to cover all
available pack files after sufficient runs, but it is easy to automate checking
a small subset of data after each backup. For a floating point value the
following command may be used:

```console
rustic -r /srv/rustic-repo check --read-data-subset=2.5%
```

When checking bigger subsets you most likely want to specify the percentage as
an integer:

```console
rustic -r /srv/rustic-repo check --read-data-subset=10%
```

Use `--read-data-subset=nS` to check a randomly chosen subset of the repository
pack files. It takes one parameter, `nS`, where 'n' is a whole number
representing file size and 'S' is the unit of file size (K/M/G/T) of pack files
to check. Behind the scenes, the specified size will be converted to percentage
of the total repository size. The behaviour of the check command following this
conversion will be the same as the percentage option above. For a file size
value the following command may be used:

```console
rustic -r /srv/rustic-repo check --read-data-subset=50M
rustic -r /srv/rustic-repo check --read-data-subset=10G
```

## Upgrading the repository format version

Repositories created using earlier rustic versions use an older repository
format version and have to be upgraded to allow using all new features.
Upgrading must be done explicitly as a newer repository version increases the
minimum rustic version required to access the repository. For example the
repository format version 2 is only readable using rustic 0.14.0 or newer.

Upgrading to repo version 2 is a two step process: first run
`migrate upgrade_repo_v2` which will check the repository integrity and then
upgrade the repository version. Repository problems must be corrected before the
migration will be possible. After the migration is complete, run `prune` to
compress the repository metadata. To limit the amount of data rewritten in at
once, you can use the `prune --max-repack-size size` parameter, see
:ref:`customize-pruning` for more details.

File contents stored in the repository will not be rewritten, data from new
backups will be compressed. Over time more and more of the repository will be
compressed. To speed up this process and compress all not yet compressed data,
you can run `prune --repack-uncompressed`.
