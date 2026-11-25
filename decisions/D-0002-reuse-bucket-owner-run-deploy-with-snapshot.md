# D-0002: Reuse the existing bucket and replace its objects through an owner-run, least-privilege deploy with version-based rollback

Status: proposed
Current: The new site replaces the objects in the existing bucket `rifas-atl-farolito` without changing its configuration; every write is run by the owner in a short-lived (15 to 60 minute) MFA session of the deploy role `rifas-deploy-role`, scoped to this bucket and assumable with MFA only by the deploy user `rifas-deploy`, through a separate named profile in the owner's `~/.aws` (never `default`, never root), with agents configured with only the read-only profile, no agent session running during that window, during an admin CLI session or while an admin console or CloudShell session is signed in where an agent's browser automation can act, and the CLI credential caches cleared right after, from an explicit file list in `site/`; rollback copies back each key's recorded pre-deploy VersionId, assets first and `index.html` last, then deletes the keys the deploy added, with a local snapshot as the second line.

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
- **Identities.** The owner had no admin identity besides root when O8 was decided (parent owner decision O8). No root credentials are used for this work after the owner creates three identities: an owner admin IAM user with MFA as the everyday admin identity, an agent read-only identity `rifas-readonly` scoped to this bucket, and an owner deploy identity. The owner creates the admin user as root with MFA, then ends the root session; the other two are created as root with MFA or as that admin user. After that, root is used only for tasks that only root can do, and no root access keys are ever created. The deploy identity is the IAM user `rifas-deploy`, whose access key can only call `sts:AssumeRole` on the role `rifas-deploy-role`, with MFA. The role holds the S3 actions: `s3:PutObject`, `s3:DeleteObject`, `s3:GetObject`, `s3:GetObjectVersion`, `s3:ListBucket` and `s3:ListBucketVersions`, so the rollback below works. It has no `s3:DeleteObjectVersion`, no ACL actions and no bucket-configuration actions. Bucket-level actions are granted on the bucket ARN and object actions on the bucket ARN with `/*`. MFA is required three times, each with `"Bool": {"aws:MultiFactorAuthPresent": "true"}`, not `BoolIfExists`: in the role's trust policy, in the user's permission to assume the role, and in the role's Allow. So the user's long-term key without MFA can neither assume the role nor write. The read-only identity keeps `s3:GetObjectVersion`. Agents may draft these policies; only the owner creates them.
- **Deploy window.** The deploy role is assumed only through a separate named CLI profile in the owner's `~/.aws`, never `default`, whose role session lasts 15 to 60 minutes. While that session is valid it sits in the CLI cache, and any process running as the owner, including an agent shell, can use it. So no agent session runs during the deploy window, and the owner clears `~/.aws/cli/cache` and `~/.aws/login/cache` right after the deploy (or rollback) ends.
- **Agent configuration and admin sessions.** Agents use separate AWS config files, `~/.aws-agents/config` and `~/.aws-agents/credentials`, that contain only the `rifas-readonly` profile. The admin and deploy profiles stay in the owner's `~/.aws`. Admin work is done in the console or CloudShell. A signed-in admin console or CloudShell session is a write-capable credential too: an agent with browser automation (for example claude-in-chrome, which acts inside the owner's existing Chrome session) can drive it. So the owner does admin console and CloudShell work in a browser profile that agents do not drive, or signs out of the console right after the admin work, and no agent browser automation runs in a browser profile while an admin console or CloudShell session is signed in there. If the admin CLI is used, the owner clears `~/.aws/login/cache` after every admin session, and no agent session runs while an admin CLI session is valid. This is configuration isolation, not an OS boundary: an agent process runs as the owner and can still read `~/.aws` and its caches, and can drive the owner's browser if given browser automation. So "no agent session during a deploy window or an admin CLI session, and no agent browser automation in a browser with a signed-in admin console or CloudShell session" stays an owner procedure with no mechanical check.
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

- The blast radius of a deploy is the object set of one bucket, written by a role session that cannot reach anything else and cannot destroy object versions. Agents are configured with only the read-only profile. Outside a deploy window, outside an admin CLI session and with no admin console or CloudShell session signed in where an agent's browser automation can act, the credentials an agent can reach are that profile and the deploy user's key, which can do nothing without MFA; so an agent session can read, not write. This holds only if the admin CLI is used only through login sessions whose cache the owner clears, and admin console work only in a browser profile agents do not drive or with a sign-out right after. During a deploy window the cached deploy role session, during an admin CLI session the cached admin login session, and while an admin console or CloudShell session is signed in the browser session, can be used by any process running as the owner or by an agent's browser automation in that browser; that is controlled only by the owner procedure above (no agent session or browser automation then), and by the short sessions, the cache clears and the sign-outs that end them.
- The deploy depends on the owner being present with an MFA device. Agents cannot write outside a deploy window, an admin CLI session or a signed-in admin console or CloudShell session reachable by their browser automation, under the conditions above.
- The three identities (the admin IAM user, the read-only identity and the deploy user) and their supporting IAM objects (the deploy role `rifas-deploy-role`, customer managed policies, access keys and MFA devices) are the only new AWS resources this work creates.
- Rollback depends on versioning staying Enabled and on no lifecycle rule expiring noncurrent versions within the rollback window. The bucket's lifecycle configuration had not been read when this was written.
- Deleting a key in a versioned bucket adds a delete marker. The previous version stays restorable. The bucket policy grants `s3:GetObject` only, not `s3:GetObjectVersion`, so noncurrent versions are not served to the public.
- Noncurrent versions are kept and stored. With about 336K of previous objects the cost is negligible.
- Public `s3:GetObject` on the whole bucket is load-bearing for serving the site (D-0003). It can be narrowed only after a move to CloudFront with OAC. Until then, the upload rules above are the only control on what becomes public.

## Links

Linked: #1
