# Runbook: child 0, create the three IAM identities

Issue: #3 (parent #1). Decisions: O2, O6, O8 on #1; D-0002.

**Who runs this:** the owner only. Agents drafted this file and the policies in
`infra/iam/`; agents run none of it and make no AWS calls.

**Public repo rule:** this file and the policies contain no account ID, no ARN
with an account number, no email and no local home path. `<ACCOUNT_ID>` is
substituted at run time, outside the repo. When you post evidence on #3, run it
through the `sanitize` function below first.

## What gets created

| Identity | IAM objects | Policy file | Credentials |
|---|---|---|---|
| 1. Admin (everyday admin, O8) | user `rifas-admin`; AWS managed `AdministratorAccess`; customer managed `rifas-admin-require-mfa` | `infra/iam/admin-require-mfa.json` | console password + virtual MFA; CLI through `aws login`. **No access key.** |
| 2. Agent read-only | user `rifas-readonly`; customer managed `rifas-readonly-s3` | `infra/iam/readonly-agent.json` | one access key (unavoidable, see below) |
| 3. Owner deploy | user `rifas-deploy` with customer managed `rifas-deploy-assume`; role `rifas-deploy-role` with trust policy and customer managed `rifas-deploy-s3` | `infra/iam/deploy-user-assume-role.json`, `infra/iam/deploy-role-trust.json`, `infra/iam/deploy-role-permissions.json` | one access key + virtual MFA; S3 access only through an MFA role session of 15 minutes |

### Why the deploy identity is a user plus a role

The AWS CLI only prompts for an MFA code and caches a short session (in
`~/.aws/cli/cache`) for a profile that assumes a role (`role_arn` +
`mfa_serial` + `duration_seconds`). So the deploy identity is:

- the user `rifas-deploy`, whose long-term key can do one thing only: call
  `sts:AssumeRole` on `rifas-deploy-role`, and only with MFA (`Bool`);
- the role `rifas-deploy-role`, whose trust policy admits only that user with
  MFA (`Bool`), and whose permission policy is exactly #3 Scope item 3, each
  Allow carrying `"Bool": {"aws:MultiFactorAuthPresent": "true"}`.

The deploy user's key on its own grants no S3 action at all. Its exposure (for
example, read by an agent shell from `~/.aws/credentials`) gives nothing without
the MFA device.

### Long-term access keys

- **Root:** none, ever (O6). Checked in A1 and V2.
- **Admin:** none. CLI use goes through `aws login`, which uses the console
  sign-in (password + MFA) and caches a short session in `~/.aws/login/cache`.
- **Read-only agent identity:** one access key, unavoidable: agents run the CLI
  non-interactively and cannot answer an MFA prompt or a browser login. The key
  can only read this one bucket and list CloudFront distributions.
- **Deploy user:** one access key, unavoidable for a CLI role profile
  (`source_profile` needs credentials) and required by falsifier F3. It grants
  nothing without MFA (see above).

Rotation guidance for both keys is in "Key rotation" below.

## Prerequisites

- Root MFA is enabled (O6) and you can sign in as root.
- An authenticator app on your phone for two new virtual MFA devices (admin and
  deploy). They are separate entries; one app can hold both.
- AWS CLI v2 on your laptop, recent enough to have `aws login`
  (`aws login help` prints a help page; if it does not, update the CLI first).
- `python3` on the laptop (used for policy comparison and nothing else).
- The PR that adds this runbook has been reviewed. Note its reviewed commit
  SHA; it is `<REVIEWED_COMMIT>` below.
- No agent session is running from the moment you start Part B until the end
  of Part B (the deploy session sits in the CLI cache while it is valid,
  D-0002).
- **Probe key `child0-probe.txt`: CONFIRM WITH THE OWNER FIRST.** This is open
  question 1 on #1. Do not run any step marked PROBE until the key name is
  confirmed. If a different key is confirmed, change `PROBE_KEY` in B0.

Sanitizer for anything you post (replaces every 12-digit number):

```bash
sanitize() { sed -E 's/[0-9]{12}/<ACCOUNT_ID>/g'; }
```

---

## Part A: root session (console + CloudShell), eu-west-1

Sign in to the console as root with MFA. Switch the region selector to
**Europe (Ireland) eu-west-1** and open **CloudShell**. CloudShell runs with the
root session's temporary credentials; no access key is created. IAM is global,
so the region only decides where CloudShell runs.

If CloudShell is not offered to the root user, do the same steps in the IAM
console (each CLI step below names the console equivalent in a comment), or do
step A2 in the console, then run A3 to A6 as `rifas-admin` from CloudShell
after signing in as that user with MFA.

### A1. Root state and placeholder rendering

```bash
sanitize() { sed -E 's/[0-9]{12}/<ACCOUNT_ID>/g'; }

# Root must have no access keys, and MFA must be on.
aws iam get-account-summary \
  --query 'SummaryMap.{AccountAccessKeysPresent:AccountAccessKeysPresent,AccountMFAEnabled:AccountMFAEnabled}'
# Expected: AccountAccessKeysPresent 0, AccountMFAEnabled 1. If keys are present: STOP, delete them first.

git clone https://github.com/acardote/rifas-atl-farolito.git "$HOME/rifas"
cd "$HOME/rifas" && git checkout <REVIEWED_COMMIT>

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
R="$HOME/child0"
mkdir -p "$R"
for f in infra/iam/*.json; do
  sed "s/<ACCOUNT_ID>/${ACCOUNT_ID}/g" "$f" > "$R/$(basename "$f")"
done
grep -l '<ACCOUNT_ID>' "$R"/*.json || echo "all placeholders substituted"
for f in "$R"/*.json; do python3 -m json.tool "$f" >/dev/null && echo "valid $f"; done
```

### A2. Identity 1: admin IAM user with MFA

```bash
aws iam create-user --user-name rifas-admin
aws iam attach-user-policy --user-name rifas-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam create-policy --policy-name rifas-admin-require-mfa \
  --description "Deny all but MFA self-management without MFA" \
  --policy-document "file://$R/admin-require-mfa.json"
aws iam attach-user-policy --user-name rifas-admin \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/rifas-admin-require-mfa"
```

Console steps (the password and the MFA QR code are not put in a shell):

1. IAM > Users > `rifas-admin` > Security credentials > Console sign-in >
   **Enable console access**. Custom password, long and unique, stored in your
   password manager. Leave "must create a new password" unticked (you set it).
2. Same tab > Multi-factor authentication (MFA) > **Assign MFA device** >
   Authenticator app. Device name e.g. `rifas-admin-mfa`. Scan the QR code,
   enter two consecutive codes.
3. Do **not** create an access key for `rifas-admin`.

```bash
aws iam list-mfa-devices --user-name rifas-admin --query 'length(MFADevices)'
# Expected: 1
aws iam list-access-keys --user-name rifas-admin --query 'length(AccessKeyMetadata)'
# Expected: 0
```

### A3. Identity 2: agent read-only user

```bash
aws iam create-user --user-name rifas-readonly
aws iam create-policy --policy-name rifas-readonly-s3 \
  --description "Agent read-only for bucket rifas-atl-farolito" \
  --policy-document "file://$R/readonly-agent.json"
aws iam attach-user-policy --user-name rifas-readonly \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/rifas-readonly-s3"
```

No console password and no MFA for this user. Its access key is created in A5.

### A4. Identity 3: deploy user and deploy role

The user must exist before the role, because the trust policy names it.

```bash
aws iam create-user --user-name rifas-deploy
aws iam create-policy --policy-name rifas-deploy-assume \
  --description "Deploy user: assume rifas-deploy-role with MFA only" \
  --policy-document "file://$R/deploy-user-assume-role.json"
aws iam attach-user-policy --user-name rifas-deploy \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/rifas-deploy-assume"

aws iam create-role --role-name rifas-deploy-role \
  --description "Owner deploy role for bucket rifas-atl-farolito, MFA only" \
  --assume-role-policy-document "file://$R/deploy-role-trust.json" \
  --max-session-duration 3600
aws iam create-policy --policy-name rifas-deploy-s3 \
  --description "Deploy object writes on rifas-atl-farolito, no DeleteObjectVersion, MFA only" \
  --policy-document "file://$R/deploy-role-permissions.json"
aws iam attach-role-policy --role-name rifas-deploy-role \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/rifas-deploy-s3"
```

`--max-session-duration 3600` is the IAM minimum (1 hour); the profile in B1
asks for 900 seconds (15 minutes), so the session is between 15 and 60 minutes.

Console step: IAM > Users > `rifas-deploy` > Security credentials >
**Assign MFA device** > Authenticator app, device name e.g.
`rifas-deploy-mfa`. No console password for this user.

```bash
aws iam list-mfa-devices --user-name rifas-deploy --query 'length(MFADevices)'
# Expected: 1
```

### A5. Access keys for the read-only and deploy users

Create each key, copy the two values straight into the laptop prompt in B1
(`aws configure --profile ...`, which reads them interactively and keeps them
out of shell history), then clear the CloudShell screen. Do not save them in a
file, a note or an issue.

```bash
aws iam create-access-key --user-name rifas-readonly \
  --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text
# -> type both values into: aws configure --profile rifas-readonly   (B1, laptop)
clear

aws iam create-access-key --user-name rifas-deploy \
  --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text
# -> type both values into: aws configure --profile rifas-deploy-keys   (B1, laptop)
clear
```

You can run the two `aws configure` key prompts of B1 on the laptop now, in between, and come back.

### A6. Policy content checks (falsifier F8) and acceptance

```bash
cmp_policy() {  # $1 = policy name, $2 = rendered local file
  aws iam get-policy-version \
    --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/$1" --version-id v1 \
    --query PolicyVersion.Document --output json > "$R/live-$1.json"
  python3 -c 'import json,sys; a=json.load(open(sys.argv[1])); b=json.load(open(sys.argv[2])); print(sys.argv[3], "MATCH" if a==b else "DIFFER")' \
    "$R/live-$1.json" "$2" "$1"
}
cmp_policy rifas-admin-require-mfa "$R/admin-require-mfa.json"
cmp_policy rifas-readonly-s3       "$R/readonly-agent.json"
cmp_policy rifas-deploy-assume     "$R/deploy-user-assume-role.json"
cmp_policy rifas-deploy-s3         "$R/deploy-role-permissions.json"
# Expected: four lines ending in MATCH.

# F8 specific checks on the live deploy policy:
grep -c 'DeleteObjectVersion' "$R/live-rifas-deploy-s3.json"   # Expected: 0
grep -c 'BoolIfExists'        "$R/live-rifas-deploy-s3.json"   # Expected: 0
python3 -c 'import json,sys; d=json.load(open(sys.argv[1])); print(sorted(a for s in d["Statement"] for a in ([s["Action"]] if isinstance(s["Action"],str) else s["Action"])))' \
  "$R/live-rifas-deploy-s3.json"
# Expected exactly: ['s3:DeleteObject', 's3:GetObject', 's3:GetObjectVersion', 's3:ListBucket', 's3:ListBucketVersions', 's3:PutObject']

# Readonly action list, for the acceptance evidence:
python3 -c 'import json,sys; d=json.load(open(sys.argv[1])); print(sorted(a for s in d["Statement"] for a in ([s["Action"]] if isinstance(s["Action"],str) else s["Action"])))' \
  "$R/live-rifas-readonly-s3.json"

# Only the expected policies are attached:
aws iam list-attached-user-policies --user-name rifas-readonly --query 'AttachedPolicies[].PolicyName'
# Expected: ["rifas-readonly-s3"]
aws iam list-attached-user-policies --user-name rifas-deploy --query 'AttachedPolicies[].PolicyName'
# Expected: ["rifas-deploy-assume"]
aws iam list-attached-role-policies --role-name rifas-deploy-role --query 'AttachedPolicies[].PolicyName'
# Expected: ["rifas-deploy-s3"]
for u in rifas-admin rifas-readonly rifas-deploy; do
  echo "$u inline: $(aws iam list-user-policies --user-name $u --query 'length(PolicyNames)')"
done
echo "role inline: $(aws iam list-role-policies --role-name rifas-deploy-role --query 'length(PolicyNames)')"
# Expected: all 0. No groups were used: list-groups-for-user should return [] for each user.
```

### A7. End the root session

```bash
rm -rf "$HOME/rifas" "$HOME/child0"
history -c
```

Close CloudShell and **sign out of root**. From here on root is used only for
root-only tasks (O6). Everything below runs on your laptop.

---

## Part B: laptop, named profiles (never `default`)

### B0. Shell hygiene

```bash
unset AWS_PROFILE AWS_DEFAULT_PROFILE AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws configure list-profiles   # check no existing profile uses the names below
sanitize() { sed -E 's/[0-9]{12}/<ACCOUNT_ID>/g'; }
BUCKET=rifas-atl-farolito
PROBE_KEY=child0-probe.txt    # PROBE: confirm with owner first (open question 1 on #1)
PROBE_FILE=$(mktemp)
printf 'child 0 probe, safe to delete\n' > "$PROBE_FILE"
```

### B1. Profiles

Keys (interactive prompts; region `eu-west-1`, output `json`):

```bash
aws configure --profile rifas-readonly      # keys from A5, first pair
aws configure --profile rifas-deploy-keys   # keys from A5, second pair
```

Then the admin profile, and the deploy role profile. The account ID and MFA
serial are read at run time and written only to your local `~/.aws/config`:

```bash
aws configure set region eu-west-1 --profile rifas-admin
aws login --profile rifas-admin     # browser opens: sign in as rifas-admin with password + MFA

ACCOUNT_ID=$(aws sts get-caller-identity --profile rifas-readonly --query Account --output text)
DEPLOY_MFA=$(aws iam list-mfa-devices --user-name rifas-deploy --profile rifas-admin \
  --query 'MFADevices[0].SerialNumber' --output text)
# If that call is denied (see V2), copy the device ARN from the console instead:
# IAM > Users > rifas-deploy > Security credentials > Multi-factor authentication.

aws configure set role_arn          "arn:aws:iam::${ACCOUNT_ID}:role/rifas-deploy-role" --profile rifas-deploy
aws configure set source_profile    rifas-deploy-keys   --profile rifas-deploy
aws configure set mfa_serial        "$DEPLOY_MFA"       --profile rifas-deploy
aws configure set duration_seconds  900                 --profile rifas-deploy
aws configure set role_session_name rifas-deploy-owner  --profile rifas-deploy
aws configure set region            eu-west-1           --profile rifas-deploy
```

Resulting `~/.aws/config` (shape; your file has the real values, the repo never does):

```ini
[profile rifas-admin]
region = eu-west-1
login_session = <written by aws login>

[profile rifas-readonly]
region = eu-west-1
output = json

[profile rifas-deploy-keys]
region = eu-west-1
output = json

[profile rifas-deploy]
role_arn = arn:aws:iam::<ACCOUNT_ID>:role/rifas-deploy-role
source_profile = rifas-deploy-keys
mfa_serial = arn:aws:iam::<ACCOUNT_ID>:mfa/rifas-deploy-mfa
duration_seconds = 900
role_session_name = rifas-deploy-owner
region = eu-west-1
```

The two access keys sit in `~/.aws/credentials` under `[rifas-readonly]` and
`[rifas-deploy-keys]`. There is no `[default]` section for any of these.

Agents use only `--profile rifas-readonly`. You use `rifas-deploy` only inside
a deploy window. `rifas-deploy-keys` is never used directly except for probe V3.

---

## Verification (one check per falsifier in #3)

Run in order. Post each command and its sanitized result (`| sanitize`) on #3
as typed evidence. "Denied" means the command fails with `AccessDenied` (or,
for V6, fails without writing) and the version count check shows no new
version.

Helper, used to prove whether a write happened (read-only identity):

```bash
probe_versions() {
  aws s3api list-object-versions --bucket "$BUCKET" --prefix "$PROBE_KEY" --profile rifas-readonly \
    --query '{versions: length(Versions || `[]`), markers: length(DeleteMarkers || `[]`)}'
}
probe_versions   # baseline. Expected: versions 0, markers 0
```

### V1 (F1): no identity resolves to root

```bash
for p in rifas-admin rifas-readonly rifas-deploy-keys; do
  echo "$p: $(aws sts get-caller-identity --profile $p --query Arn --output text | sanitize)"
done
```

Expected (allowed): `rifas-admin` shows `arn:aws:iam::<ACCOUNT_ID>:user/rifas-admin`,
`rifas-readonly` shows `...:user/rifas-readonly`, `rifas-deploy-keys` shows
`...:user/rifas-deploy`. None ends in `:root`. The deploy role session is
checked in V4.

### V2 (F2): admin has MFA, root has no access key

```bash
aws iam list-mfa-devices --user-name rifas-admin --profile rifas-admin --query 'length(MFADevices)'
# Expected: 1
aws iam get-account-summary --profile rifas-admin --query 'SummaryMap.AccountAccessKeysPresent'
# Expected: 0
```

`get-account-summary` is not on the admin policy's no-MFA allow list, so it
succeeding also shows that the `aws login` session carries MFA. **Not verified
by the drafter:** if it is denied with an explicit deny from
`rifas-admin-require-mfa`, the `aws login` session does not carry the MFA flag.
Then use the admin from the console (and CloudShell) only, post that on #3, and
do not loosen the policy without a new owner decision.

### V3 (F3): deploy user's long-term keys without MFA are denied

```bash
aws s3api put-object --bucket "$BUCKET" --key "$PROBE_KEY" --body "$PROBE_FILE" --profile rifas-deploy-keys
# Expected: DENIED (AccessDenied). PROBE.
aws sts assume-role --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/rifas-deploy-role" \
  --role-session-name no-mfa-test --profile rifas-deploy-keys 2>&1 | sanitize
# Expected: DENIED (AccessDenied), because no MFA was supplied.
probe_versions   # Expected: versions 0, markers 0
```

### V4 (F4): put-object in an MFA session of the deploy profile is allowed

```bash
aws sts get-caller-identity --profile rifas-deploy --query Arn --output text | sanitize
# Prompts for the rifas-deploy MFA code.
# Expected: arn:aws:sts::<ACCOUNT_ID>:assumed-role/rifas-deploy-role/rifas-deploy-owner (not root)
aws s3api put-object --bucket "$BUCKET" --key "$PROBE_KEY" --body "$PROBE_FILE" \
  --content-type 'text/plain; charset=utf-8' --profile rifas-deploy --query VersionId --output text
# Expected: ALLOWED, prints a VersionId. PROBE.
probe_versions   # Expected: versions 1, markers 0
```

Note the time: the session expires 15 minutes after the MFA prompt.

### V5 (F5): deploy identity can list object versions

```bash
aws s3api list-object-versions --bucket "$BUCKET" --profile rifas-deploy \
  --query 'length(Versions || `[]`)'
# Expected: ALLOWED, prints a number (no MFA prompt while the session is cached).
```

### V7 (F7): read-only identity cannot write (run before V6, while the probe object exists)

```bash
aws s3api put-object --bucket "$BUCKET" --key "$PROBE_KEY" --body "$PROBE_FILE" --profile rifas-readonly
# Expected: DENIED (AccessDenied). PROBE.
aws s3api delete-object --bucket "$BUCKET" --key "$PROBE_KEY" --profile rifas-readonly
# Expected: DENIED (AccessDenied). PROBE.
probe_versions   # Expected: still versions 1, markers 0
```

### V6 (F6): after the cache is cleared (or the session expires), a non-interactive put is denied

```bash
rm -rf ~/.aws/cli/cache ~/.aws/login/cache
aws s3api put-object --bucket "$BUCKET" --key "$PROBE_KEY" --body "$PROBE_FILE" \
  --profile rifas-deploy < /dev/null
echo "exit code: $?"
# Expected: DENIED: the command exits non-zero without writing, because no cached
# session exists and the MFA prompt cannot read a code from /dev/null. PROBE.
probe_versions   # Expected: still versions 1, markers 0
```

Expiry variant (optional, same falsifier): instead of clearing, wait more than
15 minutes after the V4 MFA prompt, then run the same `put-object ... < /dev/null`.
Expected: same denial. Clearing `~/.aws/login/cache` also logs out
`rifas-admin`; run `aws login --profile rifas-admin` again when you next need it.

### V8 (F8): deploy policy content

Done in A6 as root. To re-run later as admin, repeat the A6 `cmp_policy` and
`grep` block with `--profile rifas-admin` added to each `aws` call, comparing
against the repo files after placeholder substitution. Expected: `MATCH`,
`DeleteObjectVersion` count 0, `BoolIfExists` count 0, and the exact six-action
list.

### Probe cleanup (acceptance: `child0-probe.txt` is a delete marker)

```bash
aws s3api delete-object --bucket "$BUCKET" --key "$PROBE_KEY" --profile rifas-deploy
# Prompts for MFA. Expected: ALLOWED, "DeleteMarker": true. PROBE.
probe_versions   # Expected: versions 1, markers 1 (the noncurrent version and the marker stay;
                 # no identity has s3:DeleteObjectVersion)
rm -rf ~/.aws/cli/cache ~/.aws/login/cache
rm -f "$PROBE_FILE"
```

Part B is complete. Agent sessions may start again only after this cache clear.

### Evidence checklist for #3

Post, sanitized: A1 summary values; A2/A4 MFA device counts; A6 `MATCH` lines,
grep counts and both action lists; the attached-policy lists; V1 to V7 commands
with result lines; the final `probe_versions`; and the `duration_seconds = 900`
line of the `rifas-deploy` profile (and that the profile name is not `default`).

---

## Clearing cached credentials

After every deploy window, rollback, or probe run:

```bash
rm -rf ~/.aws/cli/cache ~/.aws/login/cache
```

`~/.aws/cli/cache` holds the deploy role session; `~/.aws/login/cache` holds the
`aws login` admin session. Long-term keys in `~/.aws/credentials` are not
affected; the deploy user's key alone grants no S3 access.

## Key rotation

Applies to `rifas-readonly` and `rifas-deploy` (the only users with keys). Run
as `rifas-admin` (CLI or console). Rotate every 90 days, at once if a key may
have been exposed, and when an agent environment that held the read-only key is
retired.

```bash
U=rifas-readonly   # or rifas-deploy; profile P=rifas-readonly or rifas-deploy-keys
aws iam list-access-keys --user-name "$U" --profile rifas-admin \
  --query 'AccessKeyMetadata[].{Id:AccessKeyId,Status:Status,Created:CreateDate}'
aws iam create-access-key --user-name "$U" --profile rifas-admin \
  --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text
aws configure --profile "$P"                                  # enter the new pair
aws sts get-caller-identity --profile "$P" --query Arn --output text | sanitize   # new key works
aws iam update-access-key --user-name "$U" --access-key-id <OLD_KEY_ID> --status Inactive --profile rifas-admin
# After a few days with no breakage:
aws iam delete-access-key --user-name "$U" --access-key-id <OLD_KEY_ID> --profile rifas-admin
```

Check use with `aws iam get-access-key-last-used --access-key-id <KEY_ID> --profile rifas-admin`.
Optional hardening: keep the `rifas-deploy` key `Inactive` between deploy
windows and set it `Active` at the start of each window.

## Rollback / teardown

Use this to undo child 0, fully or per identity. The probe object's noncurrent
version and delete marker are left in the bucket (no identity here can remove
versions; removing them is a separate owner decision).

Run identities 2 and 3 as `rifas-admin` (add `--profile rifas-admin`) or as root
in CloudShell. Run identity 1 as root in CloudShell (an admin should not delete
itself). Set `ACCOUNT_ID` first, as in A1.

Delete a user's access keys and MFA devices before the user:

```bash
del_keys() {
  for k in $(aws iam list-access-keys --user-name "$1" --query 'AccessKeyMetadata[].AccessKeyId' --output text); do
    aws iam delete-access-key --user-name "$1" --access-key-id "$k"
  done
}
del_mfa() {
  for m in $(aws iam list-mfa-devices --user-name "$1" --query 'MFADevices[].SerialNumber' --output text); do
    aws iam deactivate-mfa-device --user-name "$1" --serial-number "$m"
    aws iam delete-virtual-mfa-device --serial-number "$m"
  done
}
P="arn:aws:iam::${ACCOUNT_ID}:policy"
```

Identity 3 (deploy):

```bash
aws iam detach-role-policy --role-name rifas-deploy-role --policy-arn "$P/rifas-deploy-s3"
aws iam delete-role --role-name rifas-deploy-role
aws iam delete-policy --policy-arn "$P/rifas-deploy-s3"
del_keys rifas-deploy; del_mfa rifas-deploy
aws iam detach-user-policy --user-name rifas-deploy --policy-arn "$P/rifas-deploy-assume"
aws iam delete-user --user-name rifas-deploy
aws iam delete-policy --policy-arn "$P/rifas-deploy-assume"
```

Identity 2 (read-only):

```bash
del_keys rifas-readonly
aws iam detach-user-policy --user-name rifas-readonly --policy-arn "$P/rifas-readonly-s3"
aws iam delete-user --user-name rifas-readonly
aws iam delete-policy --policy-arn "$P/rifas-readonly-s3"
```

Identity 1 (admin, as root):

```bash
aws iam delete-login-profile --user-name rifas-admin
del_keys rifas-admin; del_mfa rifas-admin
aws iam detach-user-policy --user-name rifas-admin --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam detach-user-policy --user-name rifas-admin --policy-arn "$P/rifas-admin-require-mfa"
aws iam delete-user --user-name rifas-admin
aws iam delete-policy --policy-arn "$P/rifas-admin-require-mfa"
```

If a policy was edited after creation, `delete-policy` fails until the
non-default versions are deleted (`aws iam list-policy-versions`, then
`aws iam delete-policy-version` for each non-default one).

Laptop: remove the `rifas-*` sections from `~/.aws/config` and
`~/.aws/credentials` by hand, then `rm -rf ~/.aws/cli/cache ~/.aws/login/cache`.
Remove the authenticator entries for the deleted MFA devices.

Partial rollback (a check failed and the policy needs a fix): fix the JSON in a
PR first, then `aws iam create-policy-version --policy-arn ... --policy-document file://... --set-as-default`
from a rendered copy, and re-run A6 (comparing against the new version id, not
`v1`) and the affected V checks.
