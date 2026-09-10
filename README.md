# lask-aws

A [Lask](../lask) module that wraps the AWS CLI so Lask tasks can drive S3,
CloudFront, Cognito user provisioning, Secrets Manager, SSM Parameter Store,
and STS as typed, composable functions — plus a `cli()` escape hatch for
everything else. Design rationale lives in [doc/design.md](doc/design.md).

## Quick start

Dependency names must be lower_id identifiers (no `-`), so pick e.g. `aws`:

```text
lask deps add aws --git https://github.com/lask-task-runner/lask-aws --rev v0.1.0
```

```lask
import * as aws from "aws"

deploy(--bucket: String, --dist_id: String): String = do {
  aws.s3_sync(source = "web/dist", destination = "s3://#{bucket}", delete = true)
  inv = aws.cloudfront_create_invalidation(distribution_id = dist_id, paths = ["/index.html"])
  inv.id
}
```

```text
lask run deploy --bucket my-bucket --dist-id E123ABC
```

## Naming

Function and keyword-parameter names mirror the aws-cli service, subcommand,
and option they wrap, in snake_case: `cognito_idp_admin_create_user(--user_pool_id
= ...)` wraps `aws cognito-idp admin-create-user --user-pool-id ...`. Lask's
CLI reconstructs the original kebab-case spelling, so the two stay
recognizably the same command:

```text
lask run cognito-idp-admin-create-user --user-pool-id X --username Y
```

The one exception is credentials — see below.

## API

Every function below also takes these common keyword parameters (omitted
from the table for brevity):

| Parameter | Default | Meaning |
|---|---|---|
| `--region` | `""` | Forwarded as aws-cli's `--region` when non-empty |
| `--profile` | `""` | Forwarded as aws-cli's `--profile` when non-empty |
| `--endpoint_url` | `""` | Forwarded as aws-cli's `--endpoint-url` when non-empty (LocalStack, a S3-compatible store, a VPC endpoint, ...) |
| `--access_key_id` | `""` | Sets `AWS_ACCESS_KEY_ID` for this one command, when non-empty |
| `--secret_access_key` | `""` | Sets `AWS_SECRET_ACCESS_KEY` for this one command, when non-empty (masked) |
| `--session_token` | `""` | Sets `AWS_SESSION_TOKEN` for this one command, when non-empty (masked) |
| `--env` | `#docker("amazon/aws-cli:2.36.41")` | Lask execution environment |

| Function | Returns | Notes |
|---|---|---|
| `version()` | `String` | `aws --version` |
| `sts_get_caller_identity()` | `StsIdentity` | `{user_id, account, arn}` |
| `s3_sync(source, destination, --delete, --dryrun, --acl, --exclude, --include)` | `String` | |
| `s3_cp(source, destination, --recursive, --dryrun, --acl, --exclude, --include)` | `String` | |
| `s3_rm(target, --recursive, --dryrun, --exclude, --include)` | `String` | |
| `s3_ls(--path, --recursive)` | `Array<String>` | omit `--path` to list buckets |
| `cloudfront_create_invalidation(distribution_id, paths)` | `CfInvalidation` | `{id, status}` |
| `cognito_idp_admin_create_user(user_pool_id, username, --user_attributes, --message_action, --temporary_password, --desired_delivery_mediums)` | `String` | raw aws-cli output |
| `cognito_idp_admin_set_user_password(user_pool_id, username, password, --permanent)` | `String` | `--permanent` defaults to `true` |
| `cognito_idp_admin_delete_user(user_pool_id, username)` | `String` | |
| `secretsmanager_get_secret_value(secret_id, --version_id, --version_stage)` | `String` | `SecretString` only; result auto-masked |
| `ssm_get_parameter(name, --with_decryption)` | `String` | `Value` only; result auto-masked; `--with_decryption` defaults to `true` |
| `cli(...args)` | `CommandResult` | escape hatch; never fails on non-zero exit |

Types (named imports): `StsIdentity`, `CfInvalidation`.

## Credentials

Every credential parameter defaults to `""` and, left alone, adds no
override — ambient credentials already available to `--env` (an instance
role, a mounted AWS profile, `~/.aws/credentials` on `#local`) apply
unchanged, same as any plain `aws` invocation.

Pass `--access_key_id`/`--secret_access_key`/`--session_token` only when a
call needs its own, different credentials — typically because `--env` is a
`#docker(...)` container and the credentials are ephemeral (e.g. assumed-role
output from CI):

```lask
deploy(
  --region: String = get_env("AWS_DEFAULT_REGION"),
  --access_key_id: String = get_env("AWS_ACCESS_KEY_ID"),
  --secret_access_key!!: String = get_env("AWS_SECRET_ACCESS_KEY")
) = do {
  aws.s3_sync(
    source = "web/dist", destination = "s3://my-bucket", delete = true,
    region = region, access_key_id = access_key_id, secret_access_key = secret_access_key
  )
}
```

`--secret_access_key` and `--session_token` are declared `!!` (spec 6.10),
so whatever value ends up bound to them — whether passed explicitly or
read from `get_env` in a caller's own default — is masked out of the
command execution log automatically.

**Why this module accepts credentials at all**, unlike
[lask-terraform](../lask-terraform)'s stricter "no secrets through
arguments" stance: a `#docker(...)` environment gets no ambient host
environment variables at all (spec 10.6) — only an explicit per-command
override reaches the container — and because this module builds the final
`$[env] aws ...` command line itself, that override has to be threaded
through the module's own parameters for a caller to reach it. See
[doc/design.md §5](doc/design.md#5-credential-handling) for the full
reasoning.

## Secret results

`secretsmanager_get_secret_value` and `ssm_get_parameter` mark their return
value secret (spec 6.10) before returning it, regardless of how the caller
binds it — so the fetched value is masked from the command execution log
even if you write `token = aws.ssm_get_parameter(name = "/app/api-token")`
without `!!`. This does not extend to values returned by any other
function, or to `cli()`'s output — mark those `!!` yourself if they turn
out to carry something sensitive.

## Security

- **`cli()` never fails on a non-zero exit** — it returns the full
  `CommandResult` so callers can interpret an unwrapped subcommand's exit
  code as data. Check `.code` yourself; a silently-ignored non-zero exit
  here will not surface as a Lask failure.
- Lask relays child output to stderr logs in real time and logs every
  command line it runs (masking registered secrets only); avoid passing
  values you don't want visible in logs through parameters this module
  does not already mask.
- `cognito_idp_admin_set_user_password`'s `password` and
  `cognito_idp_admin_create_user`'s `temporary_password` are `!!`-marked
  here, but generating a suitably strong value is the caller's
  responsibility.

## Environments

By default, every function runs in `#docker("amazon/aws-cli:2.36.41")`.

`--env` is forwarded as-is, so you can override with `#local` (aws-cli must
be installed on the host) or another `#docker(...)` image (it must contain
the `aws` binary).

## Development

- Static check: `lask check --module main.lask`
- Self-test (needs Docker only, no host aws-cli: `lask run --module
  test/selftest.lask all`). Runs [LocalStack](https://github.com/localstack/localstack)
  in a plain `docker run` container and exercises S3/STS/Secrets
  Manager/SSM through this module's own functions in their normal
  containerized `--env`, which reaches LocalStack's published port via
  `host.docker.internal` — verified working on Docker Desktop and Rancher
  Desktop; a Linux Docker Engine host without that hostname pre-wired needs
  `endpoint` in `test/selftest.lask` pointed at the bridge gateway instead.
  CloudFront and Cognito are LocalStack Pro-only and are not covered by the
  automated selftest, see [doc/design.md §10](doc/design.md#10-testing-plan).
- Example tasks: `lask check --module example/main.lask`
