# gitrub

`gitrub` lists GitHub repositories, creates restorable Git backups in S3, and
can optionally delete repositories after a successful archive.

## Requirements

- Python 3
- `requests`, `boto3`, and `prettytable` from `requirements.txt`
- A GitHub API token in `GH_API_TOKEN`
- `git` installed
- GitHub SSH authentication configured for `git@github.com`
- AWS credentials/profile with permission to upload to the destination bucket
- `git-lfs` installed, so LFS objects are included in backups and restores

## Modes

Explicitly choose one of these actions:

```bash
python3 gitrub --list --organization GrabCAD
python3 gitrub --archive --days 180 --organization GrabCAD --s3-bucket gc-github-repositories
python3 gitrub --delete --days 180 --organization GrabCAD \
  --s3-bucket gc-github-repositories --dry-run
python3 gitrub --restore --repo evaluator_service_prototype \
  --s3-bucket gc-github-repositories \
  --destination /tmp/evaluator_service_prototype.git
python3 gitrub --restore-online --repo evaluator_service_prototype \
  --organization GrabCAD \
  --s3-bucket gc-github-repositories --dry-run
```

Without `--repo`, archive and delete operations select repositories where
GitHub reports `archived: true` and whose newest `pushed_at`/`updated_at`
timestamp is older than `--days`. For archived repositories, this timestamp is
used as the practical approximation of when the repository was archived.

With `--repo`, the exact repository is selected. An exact archive request does
not require the repository to be archived or older than the threshold. An
exact delete-only request selects the named repository explicitly.

`--delete` is deliberately separate from `--archive`. `--s3-bucket` is required
for every delete operation. With both flags, gitrub archives first, verifies
the uploaded S3 object exists, and deletes only after that check succeeds. With
`--delete` alone, it searches the bucket for an existing dated archive and
refuses deletion if none is found. Always use `--dry-run` before deletion.

Restore selects the newest matching archive, verifies it, and recreates a bare
Git mirror. The destination must not already exist:

```bash
git clone /tmp/evaluator_service_prototype.git evaluator_service_prototype
```

`--restore-online` first verifies that the target repository does not exist,
downloads and validates the archive, creates the repository through the API
using the saved visibility and supported repository settings, then pushes the
full mirror and any LFS objects using Git smart HTTP authenticated by
`GH_API_TOKEN`. It is separate from local `--restore` and requires
`--organization`. If pushing the mirror or restoring settings fails, gitrub
deletes the newly created incomplete repository as a rollback.

## Full Git backup

```bash
export GH_API_TOKEN='...'

python3 gitrub \
  --aws-profile usr-backups \
  --archive \
  --repo evaluator_service_prototype \
  --s3-bucket gc-github-repositories
```

The archive is uploaded at the bucket root as:

```text
s3://gc-github-repositories/YYYY-MM-DD_evaluator_service_prototype.tar.gz
s3://gc-github-repositories/YYYY-MM-DD_evaluator_service_prototype.json
```

The JSON sidecar stores the GitHub repository API metadata used for online
recreation, including visibility and supported repository settings. Online
restore requires this sidecar.

The archive uses the repository's SSH URL with a bare `git clone --mirror`, preserving Git objects,
history, branches, tags, and remote refs. Restore it with:

```bash
tar -xzf 2026-09-19_evaluator_service_prototype.tar.gz
git clone evaluator_service_prototype.git restored-repository
```

The metadata sidecar does not include issues, pull requests, releases, Actions
history, secrets, collaborators, branch protections, webhooks, or other
non-repository GitHub data. Git LFS objects are fetched into the archive and
are retained by local restores or uploaded by online restores.

Bulk operations continue after an individual repository fails, then exit with
an error that lists all failed repositories.

## Logging

Successful archive and delete actions are appended to `gitrub.log`, one action
per line, with UTC timestamp, S3 bucket, repository, and action. Dry-run actions
are printed but are not logged as completed changes.
