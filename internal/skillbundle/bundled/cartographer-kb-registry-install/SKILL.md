---
name: cartographer-kb-registry-install
description: Install an existing Git-backed Cartographer Knowledge Base from a registry-neutral JSON manifest URL.
version: "1.0"
---
# Registry KB Install — Skill

## Purpose

Use when an operator gives you an external registry or marketplace manifest for an existing
Cartographer Knowledge Base (KB). The registry is only an HTTPS URL serving JSON. This MVP installs
only Git sources and delegates the mount to `cartographer kb clone`, retaining its destination,
clone, cleanup and OKF-layout checks.

The manifest is metadata, not a trust signature. Fetch it over HTTPS, show the resolved source, and
get confirmation before cloning. Do not put credentials in the manifest URL, remote URL or output.

## Manifest v1

```json
{
  "schema_version": 1,
  "name": "aws-finops",
  "version": "1.0.0",
  "source": {
    "kind": "git",
    "remote": "https://git.example.com/example/aws-finops.git",
    "branch": "cartographer"
  }
}
```

Required: numeric `schema_version: 1`, nonempty strings `name`, `version` and
`source.remote`, and `source.kind: git`. `source.branch` is optional. `source.ref` is an
alias for it; use exactly one. In this MVP either must name a remote branch, because
`cartographer kb clone --branch` does not install a tag or detached commit. Unknown top-level
fields are allowed for registry presentation metadata; unknown `source` fields choose nothing.

## Steps

### 1. Fetch and validate before changing anything

Require an HTTPS manifest URL. Download it to a private temporary file; do not use shell command
substitution for JSON values.

```sh
manifest_url='<operator-provided HTTPS URL>'
manifest_file=$(mktemp "${TMPDIR:-/tmp}/cartographer-kb-manifest.XXXXXX") || exit 1
trap 'rm -f "$manifest_file"' EXIT
curl --fail --location --proto '=https' --proto-redir '=https' --tlsv1.2 --silent --show-error \
  "$manifest_url" -o "$manifest_file"
jq -e '
  .schema_version == 1 and
  (.name | type == "string" and length > 0) and
  (.version | type == "string" and length > 0) and
  (.source | (
    type == "object" and
    .kind == "git" and
    (.remote | type == "string" and length > 0) and
    ((has("branch") | not) or (.branch | type == "string" and length > 0)) and
    ((has("ref") | not) or (.ref | type == "string" and length > 0)) and
    ((has("branch") | not) or (has("ref") | not))
  ))
' "$manifest_file" >/dev/null
```

If `curl` or `jq` is unavailable, malformed, or exits nonzero, stop. Do not guess a remote,
retry another URL, or install another source type.

### 2. Resolve and review the plan

Extract separately, then print values as data—not a shell command:

```sh
name=$(jq -r '.name' "$manifest_file")
version=$(jq -r '.version' "$manifest_file")
remote=$(jq -r '.source.remote' "$manifest_file")
branch=$(jq -r '.source.branch // .source.ref // empty' "$manifest_file")
printf 'Manifest: %s\nVersion: %s\nSource: git\nRemote: %s\nBranch: %s\n' \
  "$name" "$version" "$remote" "${branch:-default branch}"
```

Tell the operator that this will clone the shown remote into the local Cartographer data directory
and mount it only if it passes OKF validation. The manifest does **not** make the local clone
read-only or immutable: a local administrator or a client with `rw` scope can modify it. Git
permissions protect the publisher's remote, not this clone. Ask for confirmation now.

### 3. Clone through Cartographer

After confirmation, use the existing command. Do not run `git clone` directly.

```sh
if [ -n "$branch" ]; then
  cartographer kb clone "$remote" "$name" --branch "$branch"
else
  cartographer kb clone "$remote" "$name"
fi
```

Pass `--data`, `--config`, `--local` or `--restart` only when the deployment requires them.
If the remote is not an OKF KB, stop; `cartographer-kb-import` converts a non-OKF corpus and
does not bypass this validation.

### 4. Verify

If a local service was restarted, check `cartographer kb list` or service health and connect only
intended agents/KBs. Report the manifest URL, version, remote, branch and mounted KB name.
Cartographer records Git origin, but not manifest URL or registry version.

## Limits

- Only HTTPS JSON manifests and `source.kind: git` are supported.
- `source.ref` is a remote-branch alias, not an arbitrary tag or commit.
- There is no package download, checksum/signature verification, registry authentication, update,
  marketplace-specific behavior or server-enforced local read-only mode.
- Do not promise immutability. Use read-only client tokens where needed and Git permissions on the
  publisher's source repository.
