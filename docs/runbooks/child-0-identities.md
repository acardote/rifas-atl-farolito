# Runbook: child 0, create the three IAM identities

Issue: #3 (parent #1). Decisions: O2, O6, O8 on #1; D-0002; owner decisions of
2026-11-25 recorded in the PR #6 fix-up (agent config isolation, read-only keeps
`s3:GetObjectVersion`, probe key `child0-probe.txt` confirmed).

**Who runs this:** the owner only. Agents drafted this file and the policies in
`infra/iam/`; agents run none of it and make no AWS calls.

**Public repo rule:** this file and the policies contain no account ID, no ARN
with an account number, no email and no local home path. `<ACCOUNT_ID>` is
substituted at run time, outside the repo. When you post evidence on #3, run it
through the `sanitize` function below first.

## What gets created

| Identity | IAM objects | Policy file | Credentials |
|---|---|---|---|
| 1. Admin (everyday admin, O8) | user `rifas-admin`; AWS managed `AdministratorAccess`; customer managed `rifas-admin-require-mfa` | `infra/iam/admin-require-mfa.json` | console password + virtual MFA. Used in the console and CloudShell. **No access key.** |
| 2. Agent read-only | user `rifas-readonly`; customer managed `rifas-readonly-s3` | `infra/iam/readonly-agent.json` | one access key, kept only in the agents' config files |
| 3. Owner deploy | user `rifas-deploy` with customer managed `rifas-deploy-assume`; role `rifas-deploy-role` with trust policy and customer managed `rifas-deploy-s3` | `infra/iam/deploy-user-assume-role.json`, `infra/iam/deploy-role-trust.json`, `infra/iam/deploy-role-permissions.json` | one access key, **Inactive between deploy windows**, + virtual MFA; S3 access only through an MFA role session (session up to 30 minutes; writes only while the MFA is under 15 minutes old) |

### Why the deploy identity is a user plus a role

The AWS CLI only prompts for an MFA code and caches a short session (in
`~/.aws/cli/cache`) for a profile that assumes a role (`role_arn` +
`mfa_serial` + `duration_seconds`). So the deploy identity is:

- the user `rifas-deploy`, whose long-term key can do one thing only: call
  `sts:AssumeRole` on `rifas-deploy-role`, and only with MFA (`Bool`);
- the role `rifas-deploy-role`, whose trust policy admits only that user with
  MFA (`Bool`), and whose permission policy is exactly #3 Scope item 3. Each
  Allow carries `"Bool": {"aws:MultiFactorAuthPresent": "true"}` and, in
  addition, `"NumericLessThan": {"aws:MultiFactorAuthAge": "900"}`, so AWS
  itself refuses writes more than 15 minutes after the MFA code was entered,
  whatever session length a profile asks for.

**Unverified by the drafter:** that `aws:MultiFactorAuthAge` is present in a
role session assumed with MFA. If it is absent, `NumericLessThan` does not
match and V4 is denied. V4 tests this; if V4 fails only for that reason, see
the note under V4.

### Agent isolation (owner decision 2026-11-25)

Agents get their own AWS config files, which contain **only** `rifas-readonly`:

- `~/.aws-agents/config` and `~/.aws-agents/credentials`;
- every agent session starts with
  `AWS_CONFIG_FILE=~/.aws-agents/config AWS_SHARED_CREDENTIALS_FILE=~/.aws-agents/credentials`;
- the admin and deploy profiles stay in your own `~/.aws`, and the read-only
  key is **not** in `~/.aws`.

What this does: an agent's CLI calls resolve only `rifas-readonly`; it cannot
pick the deploy or admin profile by name, and an accidental `--profile
rifas-deploy` fails because the profile does not exist in its config.

What this does **not** do: it is configuration isolation, not an OS
permission boundary. An agent running as your OS user can still read
`~/.aws/credentials`, `~/.aws/cli/cache` and `~/.aws/login/cache` directly.
So these stay owner procedures with no mechanical check:

- no agent session runs during a deploy window or an admin CLI session;
- caches are cleared at the end of each (see "Clearing cached credentials");
- **browser sessions:** no agent browser automation runs while an admin (or
  root) console or CloudShell session is signed in. Moving admin work to the
  browser does not isolate it by itself: an agent's browser automation (for
  example a Chrome extension that works inside your existing Chrome session)
  can drive a signed-in console or CloudShell tab. So either sign out of the
  console after each admin task, or do admin and root work only in a separate
  browser profile that agents do not drive (and sign out there too when done).

So the claim is: agents cannot write outside a deploy window, an admin CLI
session, or a signed-in admin/root console or CloudShell session. Each of those
windows is closed by the owner procedure above, not by a mechanical check.

The AWS-enforced parts are: the deploy key is Inactive outside deploy windows,
the deploy key alone grants no S3 action, the role requires MFA, and writes
need an MFA age under 15 minutes.

### Admin use

Do admin work in the **console or CloudShell**, signed in as `rifas-admin`
with password + MFA. `aws login` on the laptop is optional; it is used once in
V2 to record whether it carries MFA. If you ever use the admin CLI, clear
`~/.aws/login/cache` at the end of **every** admin CLI session, before any
agent starts. Sign out of the admin console (or close the separate admin
browser profile's session) after **every** admin task, before any agent
browser automation starts; see "Browser sessions" above.

### Long-term access keys

- **Root:** none, ever (O6). Checked in A1 and A6.
- **Admin:** none.
- **Read-only agent identity:** one access key, unavoidable: agents run the CLI
  non-interactively and cannot answer an MFA prompt or a browser login. The key
  can only read this one bucket and list CloudFront distributions.
- **Deploy user:** one access key, unavoidable for a CLI role profile
  (`source_profile` needs credentials) and required by falsifier F3. It grants
  nothing without MFA, and it is **Inactive except during a deploy window**.

Keys are created in the console (the secret is shown once), never printed in a
terminal. Rotation is in "Key rotation" below.

## Prerequisites

- Root MFA is enabled (O6) and you can sign in as root.
- An authenticator app on your phone for two new virtual MFA devices (admin and
  deploy). They are separate entries; one app can hold both.
- AWS CLI v2 on your laptop. For the optional V2 `aws login` check it must be
  recent enough to have `aws login` (`aws login help` prints a help page).
- `python3` 3.9 or later on the laptop (policy comparison and the no-terminal
  wrapper in V6).
- The PR that adds this runbook has been reviewed. Note the commit you will
  run: after merge, the squash commit on `main`; before merge, the reviewed
  PR head SHA. It is `<RUN_COMMIT>` below.
- No agent browser automation runs while you are signed in to the console as
  root or `rifas-admin` (Part A, B2, V1, V2, the end of Part B). Use a browser
  profile that agents do not drive, or sign out before agents start.
- No agent session runs from the start of Part B to the end of Part B (the
  deploy session sits in the CLI cache while it is valid, D-0002).
- Probe key: `child0-probe.txt`, confirmed by the owner on 2026-11-25. Steps
  that write to it are marked PROBE.

Sanitizer for anything you post (account IDs and access key IDs):

```bash
sanitize() { sed -E -e 's/[0-9]{12}/<ACCOUNT_ID>/g' -e 's/A[SK]IA[A-Z0-9]{16}/<KEY_ID>/g'; }
```

---

## Part A: root session (console + CloudShell), eu-west-1

Sign in to the console as root with MFA. Switch the region selector to
**Europe (Ireland) eu-west-1** and open **CloudShell**. CloudShell runs with the
root session's temporary credentials; no access key is created. IAM is global,
so the region only decides where CloudShell runs.

If CloudShell is not offered to the root user, do step A2 in the console as
root: before creating the policy, render `infra/iam/admin-require-mfa.json` by
hand (replace each `<ACCOUNT_ID>` with the 12-digit account ID shown in the
console account menu) and paste the result into the policy JSON editor; create
the user, attach both policies, and do A2's console steps there. Then sign out
of root, open CloudShell signed in as `rifas-admin` with MFA, run A1 in full as
`rifas-admin` (it sets `$R` and `ACCOUNT_ID` and renders the policies; the root
access-key check still applies to the account), skip A2, then run A3 to A6
(keys are made in the console in A5 either way).

### A1. Root state and placeholder rendering

```bash
sanitize() { sed -E -e 's/[0-9]{12}/<ACCOUNT_ID>/g' -e 's/A[SK]IA[A-Z0-9]{16}/<KEY_ID>/g'; }

# Root must have no access keys, and MFA must be on.
aws iam get-account-summary \
  --query 'SummaryMap.{AccountAccessKeysPresent:AccountAccessKeysPresent,AccountMFAEnabled:AccountMFAEnabled}'
# Expected: AccountAccessKeysPresent 0, AccountMFAEnabled 1. If keys are present: STOP, delete them first.

git clone https://github.com/acardote/rifas-atl-farolito.git "$HOME/rifas"
cd "$HOME/rifas"
# After merge:
git checkout <RUN_COMMIT>
# Before merge, instead:  git fetch origin pull/6/head && git checkout FETCH_HEAD
git rev-parse HEAD    # must equal <RUN_COMMIT>; if not, STOP

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
: "${R:?}" "${ACCOUNT_ID:?}"   # set by A1; if unset, run A1 first
aws iam create-user --user-name rifas-admin
aws iam attach-user-policy --user-name rifas-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam create-policy --policy-name rifas-admin-require-mfa \
  --description "Deny all but own MFA self-management without MFA" \
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
: "${R:?}" "${ACCOUNT_ID:?}"   # set by A1; if unset, run A1 first
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
: "${R:?}" "${ACCOUNT_ID:?}"   # set by A1; if unset, run A1 first
aws iam create-user --user-name rifas-deploy
aws iam create-policy --policy-name rifas-deploy-assume \
  --description "Deploy user: assume rifas-deploy-role with MFA only" \
  --policy-document "file://$R/deploy-user-assume-role.json"
aws iam attach-user-policy --user-name rifas-deploy \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/rifas-deploy-assume"
```

IAM changes take a few seconds to propagate. If the next `create-role` fails
with `MalformedPolicyDocument` and "Invalid principal", the new user is not
visible yet: wait 10 to 30 seconds and run the same command again.

```bash
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
asks for 1800 seconds (30 minutes, within D-0002's short-lived session of 15
to 60 minutes), and the policy's `aws:MultiFactorAuthAge` cap denies every S3
action once the MFA code is 15 minutes old. The profile does not ask for 900:
the CLI treats a cached role session with under 15 minutes left as expired, so
a 900-second session would prompt for MFA again in every new `aws` process.

Console step: IAM > Users > `rifas-deploy` > Security credentials >
**Assign MFA device** > Authenticator app, device name e.g.
`rifas-deploy-mfa`. No console password for this user. Copy the device's
identifier (its ARN, shown in the MFA list) for B1; you will paste it into the
local profile only.

```bash
aws iam list-mfa-devices --user-name rifas-deploy --query 'length(MFADevices)'
# Expected: 1
```

### A5. Access keys, created in the console

For each of `rifas-readonly` and `rifas-deploy`: IAM > Users > the user >
Security credentials > **Create access key** > use case "Command Line
Interface (CLI)" > confirm > Create. The secret is shown once.

- Copy the two values one at a time straight into the laptop prompt in B1
  (`aws configure`, which reads them interactively and keeps them out of shell
  history). Do not download the `.csv`; if you did, delete it.
- After each paste, clear the clipboard: `pbcopy < /dev/null` (macOS).
- Do not put the values in a file, a note or an issue.

Then set the deploy key to **Inactive** until Part B starts: IAM > Users >
`rifas-deploy` > Security credentials > Access keys > Actions > Deactivate.
(Part B step B2 activates it; the end of Part B deactivates it again.)

### A6. Policy and role checks (falsifier F8) and acceptance

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

# Live trust policy and session cap of the role:
aws iam get-role --role-name rifas-deploy-role \
  --query 'Role.AssumeRolePolicyDocument' --output json > "$R/live-trust.json"
python3 -c 'import json,sys; a=json.load(open(sys.argv[1])); b=json.load(open(sys.argv[2])); print("trust", "MATCH" if a==b else "DIFFER")' \
  "$R/live-trust.json" "$R/deploy-role-trust.json"
aws iam get-role --role-name rifas-deploy-role --query 'Role.MaxSessionDuration'
# Expected: trust MATCH; MaxSessionDuration 3600.

# F8 specific checks on the live deploy policy:
grep -c 'DeleteObjectVersion' "$R/live-rifas-deploy-s3.json"   # Expected: 0
grep -c 'BoolIfExists'        "$R/live-rifas-deploy-s3.json"   # Expected: 0
actions() { python3 -c 'import json,sys; d=json.load(open(sys.argv[1])); print(sorted(a for s in d["Statement"] for a in ([s["Action"]] if isinstance(s["Action"],str) else s["Action"])))' "$1"; }
actions "$R/live-rifas-deploy-s3.json"
# Expected exactly: ['s3:DeleteObject', 's3:GetObject', 's3:GetObjectVersion', 's3:ListBucket', 's3:ListBucketVersions', 's3:PutObject']
python3 -c 'import json,sys; d=json.load(open(sys.argv[1])); print([s["Effect"]+" "+json.dumps(s["Condition"], sort_keys=True) for s in d["Statement"]])' \
  "$R/live-rifas-deploy-s3.json"
# Expected: two "Allow" entries, each with {"Bool": {"aws:MultiFactorAuthPresent": "true"}, "NumericLessThan": {"aws:MultiFactorAuthAge": "900"}}

# Readonly action list, for the acceptance evidence (13 S3 actions + cloudfront:ListDistributions):
actions "$R/live-rifas-readonly-s3.json"

# Only the expected policies are attached:
aws iam list-attached-user-policies --user-name rifas-admin --query 'AttachedPolicies[].PolicyName'
# Expected: ["AdministratorAccess", "rifas-admin-require-mfa"] (any order)
aws iam list-attached-user-policies --user-name rifas-readonly --query 'AttachedPolicies[].PolicyName'
# Expected: ["rifas-readonly-s3"]
aws iam list-attached-user-policies --user-name rifas-deploy --query 'AttachedPolicies[].PolicyName'
# Expected: ["rifas-deploy-assume"]
aws iam list-attached-role-policies --role-name rifas-deploy-role --query 'AttachedPolicies[].PolicyName'
# Expected: ["rifas-deploy-s3"]
for u in rifas-admin rifas-readonly rifas-deploy; do
  echo "$u inline: $(aws iam list-user-policies --user-name $u --query 'length(PolicyNames)') groups: $(aws iam list-groups-for-user --user-name $u --query 'length(Groups)')"
done
echo "role inline: $(aws iam list-role-policies --role-name rifas-deploy-role --query 'length(PolicyNames)')"
# Expected: all 0.

# F2 (root side): root still has no access key; both MFA users have one device.
aws iam get-account-summary --query 'SummaryMap.AccountAccessKeysPresent'     # Expected: 0
aws iam list-mfa-devices --user-name rifas-admin  --query 'length(MFADevices)' # Expected: 1
aws iam list-mfa-devices --user-name rifas-deploy --query 'length(MFADevices)' # Expected: 1
aws iam list-access-keys --user-name rifas-deploy \
  --query 'AccessKeyMetadata[].Status' --output text                          # Expected: Inactive
```

### A7. End the root session

```bash
rm -rf "$HOME/rifas" "$HOME/child0"
history -c
```

Close CloudShell and **sign out of root** (sign-out, not just closing the
tab). From here on root is used only for
root-only tasks (O6).

---

## Part B: laptop, named profiles (never `default`)

### B0. Shell hygiene and helpers

Make sure no agent session is running. Then, in your own terminal:

```bash
unset AWS_PROFILE AWS_DEFAULT_PROFILE AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN \
      AWS_CONFIG_FILE AWS_SHARED_CREDENTIALS_FILE
aws configure list-profiles   # check no existing profile uses the names below
sanitize() { sed -E -e 's/[0-9]{12}/<ACCOUNT_ID>/g' -e 's/A[SK]IA[A-Z0-9]{16}/<KEY_ID>/g'; }

# Run a command as the agents do: only the agents' config files.
agent_aws() {
  AWS_CONFIG_FILE="$HOME/.aws-agents/config" \
  AWS_SHARED_CREDENTIALS_FILE="$HOME/.aws-agents/credentials" \
  aws "$@"
}

# Run a command with no controlling terminal and stdin from /dev/null.
# The CLI's MFA prompt (Python getpass) reads /dev/tty, not stdin, so a plain
# `< /dev/null` does not stop it; this wrapper removes the terminal itself.
notty() {
  python3 -c '
import os, subprocess, sys
pid = os.fork()
if pid == 0:
    os.setsid()
    r = subprocess.run(sys.argv[1:], stdin=subprocess.DEVNULL)
    os._exit(r.returncode)
_, status = os.waitpid(pid, 0)
sys.exit(os.waitstatus_to_exitcode(status))
' "$@"
}

BUCKET=rifas-atl-farolito
PROBE_KEY=child0-probe.txt    # confirmed by the owner 2026-11-25
PROBE_FILE=$(mktemp)
printf 'child 0 probe, safe to delete\n' > "$PROBE_FILE"
```

### B1. Profiles

Agents' files (read-only key only):

```bash
mkdir -p "$HOME/.aws-agents" && chmod 700 "$HOME/.aws-agents"
agent_aws configure --profile rifas-readonly   # keys of rifas-readonly from A5; region eu-west-1; output json
chmod 600 "$HOME/.aws-agents/config" "$HOME/.aws-agents/credentials"
```

Your own `~/.aws` (deploy, and optionally admin):

```bash
aws configure --profile rifas-deploy-keys      # keys of rifas-deploy from A5; region eu-west-1; output json

ACCOUNT_ID=$(agent_aws sts get-caller-identity --profile rifas-readonly --query Account --output text)
DEPLOY_MFA='<paste the rifas-deploy MFA device ARN copied in A4>'

aws configure set role_arn          "arn:aws:iam::${ACCOUNT_ID}:role/rifas-deploy-role" --profile rifas-deploy
aws configure set source_profile    rifas-deploy-keys   --profile rifas-deploy
aws configure set mfa_serial        "$DEPLOY_MFA"       --profile rifas-deploy
aws configure set duration_seconds  1800                --profile rifas-deploy
aws configure set role_session_name rifas-deploy-owner  --profile rifas-deploy
aws configure set region            eu-west-1           --profile rifas-deploy

aws configure set region eu-west-1 --profile rifas-admin   # only for the optional V2 aws login check
```

Resulting files (shape; yours have real values, the repo never does):

```ini
# ~/.aws-agents/config  (agents: only this profile)
[profile rifas-readonly]
region = eu-west-1
output = json

# ~/.aws-agents/credentials
[rifas-readonly]
aws_access_key_id = <KEY_ID>
aws_secret_access_key = <SECRET>

# ~/.aws/config  (owner)
[profile rifas-deploy-keys]
region = eu-west-1
output = json

[profile rifas-deploy]
role_arn = arn:aws:iam::<ACCOUNT_ID>:role/rifas-deploy-role
source_profile = rifas-deploy-keys
mfa_serial = arn:aws:iam::<ACCOUNT_ID>:mfa/rifas-deploy-mfa
duration_seconds = 1800
role_session_name = rifas-deploy-owner
region = eu-west-1

[profile rifas-admin]
region = eu-west-1
login_session = <written by aws login, only if you run V2>

# ~/.aws/credentials  (owner): only [rifas-deploy-keys]
```

No `[default]` section is created for any of these, and `rifas-readonly` is not
in `~/.aws`. Start every agent session with:

```bash
export AWS_CONFIG_FILE="$HOME/.aws-agents/config"
export AWS_SHARED_CREDENTIALS_FILE="$HOME/.aws-agents/credentials"
export AWS_PROFILE=rifas-readonly
```

Check the isolation (expected: only `rifas-readonly` is listed, and the deploy
profile is unknown):

```bash
agent_aws configure list-profiles
agent_aws sts get-caller-identity --profile rifas-deploy 2>&1 | tail -1
# Expected: an error that the profile "rifas-deploy" could not be found.
```

### B2. Open the deploy window for the probes

In the console as `rifas-admin` (password + MFA): IAM > Users >
`rifas-deploy` > Security credentials > Access keys > Actions > **Activate**.

---

## Verification (one check per falsifier in #3)

Run in order. Post each command and its sanitized result (`| sanitize`) on #3
as typed evidence. "Denied" means the command fails (`AccessDenied`, or for
V6 a non-zero exit) **and** the version count shows no new version.

Helper, used to prove whether a write happened (read-only identity):

```bash
probe_versions() {
  agent_aws s3api list-object-versions --bucket "$BUCKET" --prefix "$PROBE_KEY" --profile rifas-readonly \
    --query '{versions: length(Versions || `[]`), markers: length(DeleteMarkers || `[]`)}'
}
probe_versions   # baseline. Expected: versions 0, markers 0
```

### V1 (F1): no identity resolves to root

```bash
echo "rifas-readonly: $(agent_aws sts get-caller-identity --profile rifas-readonly --query Arn --output text | sanitize)"
echo "rifas-deploy-keys: $(aws sts get-caller-identity --profile rifas-deploy-keys --query Arn --output text | sanitize)"
```

Expected (allowed): `arn:aws:iam::<ACCOUNT_ID>:user/rifas-readonly` and
`arn:aws:iam::<ACCOUNT_ID>:user/rifas-deploy`. For the admin, open CloudShell
signed in as `rifas-admin` and run
`aws sts get-caller-identity --query Arn --output text`; expected
`...:user/rifas-admin`. None ends in `:root`. The deploy role session is
checked in V4.

### V2 (F2): admin has MFA, root has no access key; `aws login` MFA check

F2 itself is checked in A6 (root) and can be re-run in CloudShell as
`rifas-admin`:

```bash
aws iam list-mfa-devices --user-name rifas-admin --query 'length(MFADevices)'   # Expected: 1
aws iam get-account-summary --query 'SummaryMap.AccountAccessKeysPresent'      # Expected: 0
```

`aws login` check (records whether the admin CLI is usable at all). Run it on
the laptop with no agent session running, and clear the cache right after:

```bash
aws login --profile rifas-admin     # browser: sign in as rifas-admin with password + MFA
aws iam get-account-summary --profile rifas-admin --query 'SummaryMap.AccountAccessKeysPresent'
rm -rf ~/.aws/login/cache
```

`get-account-summary` is not on the admin policy's no-MFA exception list, so it
succeeding shows that the `aws login` session carries MFA. **Unverified by
the drafter.** Post the result on #3 before treating V2 as passed. If it is
denied with an explicit deny from `rifas-admin-require-mfa`, the `aws login`
session does not carry the MFA flag: keep using the admin in the console and
CloudShell only, record that on #3, and do not loosen the policy without a new
owner decision.

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

Note the time. Writes are allowed for about 15 minutes after the MFA prompt
(the `aws:MultiFactorAuthAge` cap). The role session lasts up to 30 minutes,
but writes are denied once the MFA is 15 minutes old, and from that point the
CLI also treats the cached session as expired and asks for MFA again. Run V5
promptly after V4.

If the role session is created (first command shows the assumed role) but the
put is `AccessDenied` with "no identity-based policy allows", the likely cause
is that `aws:MultiFactorAuthAge` is absent in the role session. Stop, post that
on #3, and fix it through a PR that removes the `NumericLessThan` condition
(then `create-policy-version` per "Partial rollback" below). Do not edit the
live policy by hand.

### V5 (F5): deploy identity can list object versions

```bash
aws s3api list-object-versions --bucket "$BUCKET" --profile rifas-deploy \
  --query 'length(Versions || `[]`)'
# Expected: ALLOWED, prints a number (no MFA prompt if run within about 15
# minutes of the V4 prompt, while the cached session is reused).
```

### V7 (F7): read-only identity cannot write (run before V6, while the probe object exists)

```bash
agent_aws s3api put-object --bucket "$BUCKET" --key "$PROBE_KEY" --body "$PROBE_FILE" --profile rifas-readonly
# Expected: DENIED (AccessDenied). PROBE.
agent_aws s3api delete-object --bucket "$BUCKET" --key "$PROBE_KEY" --profile rifas-readonly
# Expected: DENIED (AccessDenied). PROBE.
probe_versions   # Expected: still versions 1, markers 0
```

### V6 (F6): after the cache is cleared (or the session expires), a non-interactive put is denied

First, a control that the wrapper really removes the terminal:

```bash
python3 -c 'open("/dev/tty"); print("terminal present")'
# Expected: terminal present
notty python3 -c 'open("/dev/tty"); print("terminal present")' 2>&1 | tail -1
# Expected: OSError ... '/dev/tty' (no terminal inside the wrapper). If it prints
# "terminal present", STOP: the check below would not be valid.
```

Then the check:

```bash
rm -rf ~/.aws/cli/cache ~/.aws/login/cache
notty aws s3api put-object --bucket "$BUCKET" --key "$PROBE_KEY" --body "$PROBE_FILE" \
  --profile rifas-deploy
echo "exit code: $?"
# Expected: DENIED. The prompt text "Enter MFA code for ..." is printed (with
# no terminal, getpass falls back: it may also print a GetPassWarning), then
# the prompt gets end-of-file: an EOFError and/or a non-zero exit code. No
# code can be typed, and nothing is written. PROBE.
probe_versions   # Expected: still versions 1, markers 0
```

The printed prompt text is expected and does not invalidate the check; the
terminal control above is what proves no code could be entered. If the
command instead waits for input, do not type a code: press Ctrl-C, and record
the check as not valid. The check passes on: prompt text, then EOFError or a
non-zero exit code, and an unchanged `probe_versions`.

Expiry variant (optional, same falsifier): instead of clearing the cache, wait
more than 15 minutes after the V4 MFA prompt, then run the same `notty aws
s3api put-object ...` command. Expected: the same result (prompt text, then
EOFError or a non-zero exit) and an unchanged `probe_versions`. (Once the
cached session has under 15 minutes left the CLI tries to assume the role
again, which needs a new MFA code; with no terminal it gets end-of-file. Even
a reused session would be denied, because the MFA is over 15 minutes old.)

### V8 (F8): deploy policy content

Done in A6. To re-run later, open CloudShell as `rifas-admin`, repeat A1's
clone and render steps and the A6 block. Expected: every comparison `MATCH`,
`DeleteObjectVersion` count 0, `BoolIfExists` count 0, the exact six-action
list, and both Allow conditions as listed.

### Probe cleanup (acceptance: `child0-probe.txt` is a delete marker)

```bash
aws s3api delete-object --bucket "$BUCKET" --key "$PROBE_KEY" --profile rifas-deploy
# Prompts for MFA. Expected: ALLOWED, "DeleteMarker": true. PROBE.
probe_versions   # Expected: versions 1, markers 1 (the noncurrent version and the marker stay;
                 # no identity has s3:DeleteObjectVersion)
rm -rf ~/.aws/cli/cache ~/.aws/login/cache
rm -f "$PROBE_FILE"
```

### Close the deploy window

In the console as `rifas-admin`: IAM > Users > `rifas-deploy` > Security
credentials > Access keys > Actions > **Deactivate**. Then:

```bash
aws sts get-caller-identity --profile rifas-deploy-keys 2>&1 | tail -1
# Expected: an InvalidClientTokenId error (the key is Inactive).
```

### Session-window check (before any agent starts again)

Not mechanical; post the answers on #3 as owner-stated:

1. `~/.aws/cli/cache` and `~/.aws/login/cache` do not exist
   (`ls -d ~/.aws/cli/cache ~/.aws/login/cache 2>&1` shows "No such file").
2. The `rifas-deploy` key shows Inactive (step above).
3. No console or CloudShell tab is signed in as root or `rifas-admin` in any
   browser profile an agent can drive: open the AWS console URL in that
   profile; expected: the sign-in page, not the console home.
4. No agent session or agent browser automation ran during Part A or Part B.

Part B is complete. Agent sessions may start again only after the cache clear,
the deactivation, the console sign-out and the check above.

### Evidence checklist for #3

Post, sanitized: A1 summary values; the A6 `MATCH` lines, trust `MATCH`,
`MaxSessionDuration`, grep counts, condition list and both action lists; the
attached-policy, inline and group counts; V1 to V7 commands with result lines
(including the V6 terminal control and the V2 `aws login` result); the final
`probe_versions`; the `InvalidClientTokenId` line after deactivation; the
session-window check answers; and the
`duration_seconds = 1800` line of the `rifas-deploy` profile (and that the
profile name is not `default`).

---

## Deploy window (standard procedure, for child 4 and later)

1. Confirm no agent session and no agent browser automation is running.
2. Console as `rifas-admin`: **Activate** the `rifas-deploy` access key.
3. Run the deploy with `--profile rifas-deploy`: one MFA prompt per deploy
   window. Writes are allowed for about 15 minutes after the MFA. The role
   session lasts up to 30 minutes, but writes are denied once the MFA is 15
   minutes old (and a long `s3 sync` cannot refresh past that without a new
   code). To continue after that, run step 4 and re-authenticate with a new
   MFA code.
4. `rm -rf ~/.aws/cli/cache ~/.aws/login/cache`
5. Console as `rifas-admin`: **Deactivate** the `rifas-deploy` access key.
6. **Sign out** of the admin console (or end the session in the separate admin
   browser profile).
7. Run the "Session-window check" (Part B). Only then start agent sessions or
   agent browser automation.

## Clearing cached credentials

After every deploy window, rollback, probe run, and every admin CLI session:

```bash
rm -rf ~/.aws/cli/cache ~/.aws/login/cache
```

`~/.aws/cli/cache` holds the deploy role session; `~/.aws/login/cache` holds
an `aws login` admin session. Long-term keys in the credentials files are not
affected; the deploy user's key alone grants no S3 access, and it is Inactive
outside deploy windows.

## Key rotation

Applies to `rifas-readonly` and `rifas-deploy` (the only users with keys).
Rotate every 90 days, at once if a key may have been exposed, and when an agent
environment that held the read-only key is retired. Keys are created in the
console as `rifas-admin` (the secret is shown once).

1. Set the user and the profile on their own lines (one pair at a time):

   ```bash
   # read-only:
   U=rifas-readonly
   P=rifas-readonly
   CFG=agent_aws          # the read-only key lives only in the agents' files
   # or deploy:
   # U=rifas-deploy
   # P=rifas-deploy-keys
   # CFG=aws
   : "${U:?}" "${P:?}" "${CFG:?}"   # stops here if any is unset
   ```

2. Console as `rifas-admin`: IAM > Users > `$U` > Security credentials >
   Access keys. Note the old key's ID and age. **Create access key** (CLI use
   case). A user can hold two keys at once.
3. Paste the new pair into the prompt, then clear the clipboard:

   ```bash
   "$CFG" configure --profile "$P"
   pbcopy < /dev/null
   "$CFG" sts get-caller-identity --profile "$P" --query Arn --output text | sanitize   # new key works
   ```

   For `rifas-deploy`, the new key must be Active for this test; deactivate it
   again afterwards unless a deploy window is open.
4. Console: **Deactivate** the old key. After a few days with no breakage,
   **Delete** it. The console shows each key's "last used" time.

## Rollback / teardown

Use this to undo child 0, fully or per identity. The probe object's noncurrent
version and delete marker are left in the bucket (no identity here can remove
versions; removing them is a separate owner decision).

Run identities 2 and 3 in CloudShell as `rifas-admin` (or as root). Run
identity 1 in CloudShell as root (an admin should not delete itself). Set
`ACCOUNT_ID` first, as in A1.

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
: "${ACCOUNT_ID:?}"   # stops here if ACCOUNT_ID is unset
POL="arn:aws:iam::${ACCOUNT_ID}:policy"
```

Identity 3 (deploy):

```bash
aws iam detach-role-policy --role-name rifas-deploy-role --policy-arn "$POL/rifas-deploy-s3"
aws iam delete-role --role-name rifas-deploy-role
aws iam delete-policy --policy-arn "$POL/rifas-deploy-s3"
del_keys rifas-deploy; del_mfa rifas-deploy
aws iam detach-user-policy --user-name rifas-deploy --policy-arn "$POL/rifas-deploy-assume"
aws iam delete-user --user-name rifas-deploy
aws iam delete-policy --policy-arn "$POL/rifas-deploy-assume"
```

Identity 2 (read-only):

```bash
del_keys rifas-readonly
aws iam detach-user-policy --user-name rifas-readonly --policy-arn "$POL/rifas-readonly-s3"
aws iam delete-user --user-name rifas-readonly
aws iam delete-policy --policy-arn "$POL/rifas-readonly-s3"
```

Identity 1 (admin, as root):

```bash
aws iam delete-login-profile --user-name rifas-admin
del_keys rifas-admin; del_mfa rifas-admin
aws iam detach-user-policy --user-name rifas-admin --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam detach-user-policy --user-name rifas-admin --policy-arn "$POL/rifas-admin-require-mfa"
aws iam delete-user --user-name rifas-admin
aws iam delete-policy --policy-arn "$POL/rifas-admin-require-mfa"
```

If a policy was edited after creation, `delete-policy` fails until the
non-default versions are deleted (`aws iam list-policy-versions`, then
`aws iam delete-policy-version` for each non-default one).

Laptop: remove the `rifas-*` sections from `~/.aws/config` and
`~/.aws/credentials` by hand, delete `~/.aws-agents`, then
`rm -rf ~/.aws/cli/cache ~/.aws/login/cache`. Remove the authenticator entries
for the deleted MFA devices.

Partial rollback (a check failed and a policy needs a fix): fix the JSON in a
PR first, render it as in A1, then in CloudShell as `rifas-admin` run
`aws iam create-policy-version --policy-arn ... --policy-document file://... --set-as-default`,
and re-run A6 (comparing against the new version id, not `v1`) and the
affected V checks.
