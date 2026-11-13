---
name: cartographer-kb-publish
description: Validate a local Cartographer Knowledge Base and publish its checked-out commit to a registry-ready Git branch.
version: "1.0"
---
# KB Publish — Skill

## Purpose

Use when an operator wants to distribute an existing Cartographer Knowledge Base (KB) through any
external registry or marketplace that resolves to a Git repository. It validates the KB, pushes the
checked-out commit to a conventional `cartographer` branch, and prints a registry-neutral manifest
snippet. It creates no registry account, uploads no package and does not make consumer clones
immutable.

## Inputs

Ask for manifest `name` and `version`. The name is the stable registry-facing KB identifier and
the version is publisher-managed metadata. Work from a local KB clone whose publishing remote is
`origin`.

## Steps

### 1. Validate the KB and working tree

Run validation through the Cartographer server mounting this clone. Resolve every error before
publishing.

```sh
cartographer call --provider <provider> validate --json '{}'
git status --porcelain
git remote get-url origin
git rev-parse HEAD
```

Replace `<provider>` with the configured client provider. On a multi-KB server, run this from a
folder bound to this KB or use the usual routed call that names it. If no server mounts the clone,
start one against it or use the matching MCP `validate` tool. Do not publish with validation errors
or nonempty `git status --porcelain`.

### 2. Review the commit

Confirm that the checked-out commit is what the operator intends to publish. The push is
non-forcing: it creates `cartographer` or fast-forwards it and refuses a diverged branch.

```sh
remote=$(git remote get-url origin)
commit=$(git rev-parse HEAD)
printf 'Remote: %s\nCommit: %s\nPublish branch: cartographer\n' "$remote" "$commit"
```

### 3. Publish the distribution branch

After confirmation, create or move the local branch to the checked-out commit and push without
force:

```sh
if git show-ref --verify --quiet refs/heads/cartographer &&
  ! git merge-base --is-ancestor cartographer HEAD; then
  echo 'local cartographer branch is not an ancestor of HEAD; resolve it first' >&2
  exit 1
fi
git branch -f cartographer HEAD
git push origin HEAD:refs/heads/cartographer
```

If the remote branch diverged, stop. Never use `--force`, replace the remote branch, or change the
KB's normal development branch. Resolve history with the operator, then repeat.

### 4. Return install metadata

Print this using the reviewed values:

```json
{
  "schema_version": 1,
  "name": "<name>",
  "version": "<version>",
  "source": {
    "kind": "git",
    "remote": "<origin remote>",
    "branch": "cartographer"
  }
}
```

Also report the published commit SHA. The v1 installer accepts the branch because
`cartographer kb clone --branch` requires a remote branch. Give consumers this JSON through their
chosen registry and direct them to `cartographer-kb-registry-install`.

## Limits

- This is a Git branch push, not signing, package/checksum, registry upload, authentication,
  payment, discovery or automatic update.
- The branch is mutable unless the Git remote's permissions or branch protection prevent changes.
- A consumer clone remains writable to its local administrator and `rw` clients. Git permissions
  protect the source repository; they do not enforce local immutability.
