# D-0002: Reuse the existing bucket and replace its objects through an owner-run, least-privilege deploy with version-based rollback

Status: proposed
Current: The new site replaces the objects in the existing bucket `rifas-atl-farolito` without changing its configuration; every write is run by the owner in a short-lived (15 to 60 minute) MFA session of a deploy identity scoped to this bucket, through a separate named profile (never `default`, never root), with no agent session running during that window and the CLI credential caches cleared right after, from an explicit file list in `site/`; rollback copies back each key's recorded pre-deploy VersionId, assets first and `index.html` last, then deletes the keys the deploy added, with a local snapshot as the second line.

## Context

The owner asked to reuse the bucket `rifas-atl-farolito`, which holds the previous site's files (8 objects in the bucket), at maturity tier production. Read-only `s3api` reads on 2026-11-24 (listed in the linked parent's Move 1 evidence) showed:

- Region eu-west-1. No website configuration.
- Block Public Access: all four flags off. The bucket policy allows `s3:GetObject` to everyone on every key and grants no `s3:ListBucket`. Policy status is public. Anything uploaded is world-readable at once.
- Object ownership `BucketOwnerEnforced`, so ACLs are disabled and an upload that sets an ACL is rejected.
- Versioning `Enabled`. Default encryption SSE-S3 with a bucket key.

At drafting time the CLI profile used resolved to the account root user, cached in a session that any process running as the owner, including an agent's shell, could use. A rule that "agents do not write" was then enforced by instructions alone, and a read-only inspection ran with full account power.

The S3 REST endpoint serves each object's stored Content-Type and Cache-Control as is, so wrong metadata reaches visitors directly.

## Decision

- **Reuse the bucket.** Do not create, rename or reconfigure it (policy, Block Public Access, ownership, encryption, versioning, website, CORS, lifecycle) as part of this work.
- **Identities.** The owner has no admin identity besides root (parent owner decision O8). No root credentials are used for this work after the owner creates three identities: an owner admin IAM user with MFA as the everyday admin identity, an agent read-only identity scoped to this bucket, and an owner deploy identity scoped to this bucket. The owner creates them as root with MFA, then ends the root session; after that, root is used only for tasks that only root can do, and no root access keys are ever created. The deploy identity may `s3:PutObject`, `s3:DeleteObject`, `s3:GetObject`, `s3:GetObjectVersion`, `s3:ListBucket` and `s3:ListBucketVersions`, so the rollback below works. It has no `s3:DeleteObjectVersion`, no ACL actions and no bucket-configuration actions. Bucket-level actions are granted on the bucket ARN and object actions on the bucket ARN with `/*`. Its Allow requires `"Bool": {"aws:MultiFactorAuthPresent": "true"}`, not `BoolIfExists`, so long-term keys without MFA are denied. Agents may draft these policies; only the owner creates them.
- **Deploy window.** The deploy identity is used only through a separate named CLI profile, never `default`, whose session lasts 15 to 60 minutes. While that session is valid it sits in the CLI cache, and any process running as the owner, including an agent shell, can use it. So no agent session runs during the deploy window, and the owner clears `~/.aws/cli/cache` and `~/.aws/login/cache` right after the deploy (or rollback) ends.
- **Who writes.** Every write to the bucket is run by the owner with the deploy profile, inside a deploy window. Agents run only read-only commands with the read-only identity.
- **Before the first write**, record per key: VersionId, ETag, size, Content-Type and Cache-Control. Take a local snapshot of every object and that metadata to a directory outside the repo. The snapshot is never uploaded to the bucket.
- **Upload rules.**
  - Upload only files from `site/`, by an explicit per-file list (or an `--exclude "*"` plus `--include` allow-list), never the repo root.
  - Show the owner a `--dryrun` file list before the real run.
  - Never pass `--acl`.
  - Set Content-Type explicitly on every file, with `charset=utf-8` on HTML (`text/html; charset=utf-8`).
  - Set `Cache-Control: no-cache` on `index.html`.
  - Upload assets first and `index.html` last, so the live page never references an asset that is not there yet.
  - New asset keys do not reuse the previous site's keys, other than `index.html`.
- **Objects not in the new build** are handled as the linked parent's scope says.
- **Rollback, primary.** For each recorded key, `s3api copy-object` with `--copy-source` set to that key at its recorded VersionId. This restores the bytes and the stored metadata. Order: restore the asset keys first and `index.html` last, so the restored page never references an asset that is not back yet; then delete the keys the deploy added that did not exist before.
- **Rollback, second line.** Re-upload from the local snapshot with the recorded Content-Type and Cache-Control.

## Consequences

- The blast radius of a deploy is the object set of one bucket, written by an identity that cannot reach anything else and cannot destroy object versions. Outside a deploy window, an agent session can read, not write. During the window, any process running as the owner can use the cached deploy session; that is controlled only by the owner running no agent session then, and by the short session and the cache clear that end the window.
- The deploy depends on the owner being present with an MFA device. Agents cannot write outside a deploy window.
- The three identities (the admin IAM user, the read-only identity and the deploy identity) are the only new AWS resources this work creates.
- Rollback depends on versioning staying Enabled and on no lifecycle rule expiring noncurrent versions within the rollback window. The bucket's lifecycle configuration had not been read when this was written.
- Deleting a key in a versioned bucket adds a delete marker. The previous version stays restorable. The bucket policy grants `s3:GetObject` only, not `s3:GetObjectVersion`, so noncurrent versions are not served to the public.
- Noncurrent versions are kept and stored. With about 336K of previous objects the cost is negligible.
- Public `s3:GetObject` on the whole bucket is load-bearing for serving the site (D-0003). It can be narrowed only after a move to CloudFront with OAC. Until then, the upload rules above are the only control on what becomes public.

## Links

Linked: #1
