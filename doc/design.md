# lask-aws Design Document

`lask-aws` is a Lask module that wraps the AWS CLI (`aws`) so that Lask tasks
(CI/CD pipelines, operational automation) can drive the handful of AWS
operations a typical deploy pipeline actually needs — S3 sync, CloudFront
invalidation, Cognito user provisioning, Secrets Manager/SSM reads, an
identity check — as typed, composable functions, plus an escape hatch for
everything else.

This document defines the goals, public API, internal structure, and the
design decisions that follow from the Lask language specification
(`lask/doc/spec.md`; section numbers below refer to it). It follows the same
shape as [lask-terraform/doc/design.md](../../lask-terraform/doc/design.md),
which set the precedent this module reuses throughout.

## 1. Goals

- Wrap the AWS operations a Lask-driven deploy pipeline demonstrably needs
  today (`example/04-webapp` in the `lask` repository hand-rolls exactly
  these calls): S3 object sync/copy/removal, CloudFront cache invalidation,
  Cognito user-pool user provisioning, and reading a secret at deploy time.
- Function and keyword-parameter names mirror the aws-cli
  service/subcommand/option they wrap, in snake_case, so a reader who
  already knows aws-cli recognizes the call without consulting this
  module's own docs (§4).
- Preserve aws-cli's exit-code semantics through Lask's error contract
  (spec 6.9, 8.10), the same way lask-terraform does: a failed command
  becomes an `Error` whose `code` is the process's exit code and whose
  `message` is its stderr.
- Work in any Lask execution environment (`#local`, `#docker(...)`) via a
  pass-through `--env` keyword parameter, defaulting to the official
  `amazon/aws-cli` image.
- Provide a generic escape hatch (`cli`) for any aws-cli subcommand this
  module does not wrap, since the AWS CLI's surface (dozens of services,
  hundreds of subcommands) is far larger than any one module should try to
  model.

## 2. Non-Goals

- No modeling of the full JSON response shape of every wrapped operation.
  Only the fields a caller demonstrably polls on are typed (`StsIdentity`,
  `CfInvalidation`); everything else returns the raw string aws-cli printed,
  same as lask-terraform's `show_json` returning `Any` by design.
- No coverage of every AWS service. This module wraps S3, CloudFront,
  Cognito Identity Provider, STS, Secrets Manager, and SSM Parameter Store —
  the services `example/04-webapp` and comparable pipelines need — plus
  `cli()` for the rest. Additional services are added as real usage shows
  which ones matter (§10).
- No credential provisioning or rotation. This module only forwards
  credentials it is explicitly given, or lets ambient credentials of the
  target `--env` apply; it never reads `~/.aws/credentials`, assumes a role,
  or performs SSO login on the caller's behalf.
- No retry/backoff logic beyond what aws-cli itself does. A throttled or
  transiently-failing call surfaces as an ordinary non-zero-exit `Error`;
  wrap a wrapped call in the caller's own retry loop if needed.

## 3. Distribution and Consumption

The module is published as a git tree dependency (spec ch. 5, 11.5) with the
public API at `main.lask` (the bare-name entry-point convention), identical
to lask-terraform.

Consumer setup:

```text
lask deps add aws --git https://github.com/lask-task-runner/lask-aws --rev v0.1.0
```

```lask
import * as aws from "aws"        // functions
import { StsIdentity } from "aws" // types need named imports (spec ch. 5)

deploy(): String = do {
  aws.s3_sync(source = "web/dist", destination = "s3://my-bucket", delete = true)
  aws.cloudfront_create_invalidation(distribution_id = "E123", paths = ["/index.html"]).id
}
```

Design notes carried over from lask-terraform (spec 5, "Re-export"):

- **Namespace import is the recommended style**, for the same reason as
  lask-terraform: several names (`version`, `cli`) are generic enough to
  collide with a consumer's own declarations under named import, and types
  can only be brought in by named import regardless.
- **All public functions live directly in `main.lask`, not re-exported.**
  Binding `s3_sync = impl.s3_sync` would turn `s3_sync` into a function
  *value*, and calls through function values cannot use keyword arguments
  (spec 6.1, 7.5) — every `--delete`/`--exclude` would silently fall back to
  its default. Internal helpers live in `lib/` and are imported by
  `main.lask`; imports are not public symbols, so they do not leak.

## 4. Public API

### 4.1 Naming convention

`<service>_<subcommand>` in snake_case, mirroring the aws-cli invocation it
wraps: `cognito_idp_admin_create_user` wraps
`aws cognito-idp admin-create-user`. Keyword parameters follow the same
rule: aws-cli's `--user-pool-id` becomes `--user_pool_id`. The CLI's own
kebab-case mapping (spec 11.2) then reconstructs the original spelling
(`lask run cognito-idp-admin-create-user --user-pool-id ...`), so a reader
moving between aws-cli docs and `lask run --help` output sees the same
names throughout.

The one deliberate exception is credentials (§5): aws-cli itself has no
`--access-key-id`/`--secret-access-key`/`--session-token` flags (by AWS's
own design — credentials are never accepted as CLI arguments), so this
module's equivalent parameters are named after the `AWS_*` environment
variables they set instead, which is the name an AWS user already
associates with a credential value.

### 4.2 Common keyword parameters

Every wrapped function takes these keyword parameters (defaults shown):

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `--region` | `String` | `""` | Forwarded as aws-cli's own `--region` when non-empty. |
| `--profile` | `String` | `""` | Forwarded as aws-cli's own `--profile` when non-empty. |
| `--endpoint_url` | `String` | `""` | Forwarded as aws-cli's own `--endpoint-url` when non-empty — the hook a non-AWS-hosted endpoint (LocalStack, an S3-compatible store) needs; also what the selftest (§10) uses to reach LocalStack. |
| `--access_key_id` | `String` | `""` | Sets `AWS_ACCESS_KEY_ID` for this command only, when non-empty (§5). |
| `--secret_access_key` | `String` (`!!`) | `""` | Sets `AWS_SECRET_ACCESS_KEY` for this command only, when non-empty (§5). |
| `--session_token` | `String` (`!!`) | `""` | Sets `AWS_SESSION_TOKEN` for this command only, when non-empty (§5). |
| `--env` | `Environment` | `#docker("amazon/aws-cli:2.36.41")` | Execution environment for the command (spec ch. 10). |

`cli()` (the escape hatch) takes only the credential and `--env` parameters,
since global options like `--region`/`--profile` are passed as ordinary
elements of its variadic `args`.

Making `env` a keyword parameter (never positional) matters for the same
two reasons lask-terraform's design document gives: CLI-invocability (spec
11.2) and correct enumeration by `lask envs <fn> --check` (spec 11.4).

### 4.3 Types

```lask
type StsIdentity   = Record<user_id: String, account: String, arn: String>
type CfInvalidation = Record<id: String, status: String>
```

Both are deliberately partial views of much larger aws-cli JSON responses —
only the fields a caller demonstrably needs (§2).

### 4.4 Functions

Common parameters (`--region`, `--profile`, `--access_key_id`,
`--secret_access_key`, `--session_token`, `--env`) are elided below.

```lask
version(): String
// `aws --version`; whichever of stdout/stderr is non-empty (the two major
// CLI versions disagree on which stream this goes to).

sts_get_caller_identity(): StsIdentity
// `sts get-caller-identity --output json`, decoded.

s3_sync(source: String, destination: String,
        --delete: Bool = false, --dryrun: Bool = false, --acl: String = "",
        --exclude: Array<String> = [], --include: Array<String> = []): String
// `s3 sync <source> <destination> [--delete] [--dryrun] [--acl <v>]
//         [--exclude <p>]... [--include <p>]...`

s3_cp(source: String, destination: String,
      --recursive: Bool = false, --dryrun: Bool = false, --acl: String = "",
      --exclude: Array<String> = [], --include: Array<String> = []): String
// `s3 cp <source> <destination> [--recursive] [--dryrun] [--acl <v>]
//        [--exclude <p>]... [--include <p>]...`

s3_rm(target: String,
      --recursive: Bool = false, --dryrun: Bool = false,
      --exclude: Array<String> = [], --include: Array<String> = []): String
// `s3 rm <target> [--recursive] [--dryrun] [--exclude <p>]... [--include <p>]...`

s3_ls(--path: String = "", --recursive: Bool = false): Array<String>
// `s3 ls [<path>] [--recursive]`; stdout split into non-empty lines.
// Omitting --path lists buckets, matching aws-cli's own `s3 ls`.

cloudfront_create_invalidation(distribution_id: String, paths: Array<String>): CfInvalidation
// `cloudfront create-invalidation --distribution-id <id> --paths <p>... --output json`, decoded.

cognito_idp_admin_create_user(
  user_pool_id: String, username: String,
  --user_attributes: Map<String> = {}, --message_action: String = "",
  --temporary_password!!: String = "", --desired_delivery_mediums: Array<String> = []
): String
// `cognito-idp admin-create-user --user-pool-id <id> --username <name>
//    [--user-attributes Name=k,Value=v...] [--message-action <v>]
//    [--temporary-password <v>] [--desired-delivery-mediums <v>...]`
// Returns aws-cli's raw output (JSON by default) describing the created user.

cognito_idp_admin_set_user_password(
  user_pool_id: String, username: String, password!!: String, --permanent: Bool = true
): String
// `cognito-idp admin-set-user-password --user-pool-id <id> --username <name>
//    --password <pw> [--permanent]`

cognito_idp_admin_delete_user(user_pool_id: String, username: String): String
// `cognito-idp admin-delete-user --user-pool-id <id> --username <name>`

secretsmanager_get_secret_value(
  secret_id: String, --version_id: String = "", --version_stage: String = ""
): String
// `secretsmanager get-secret-value --secret-id <id> [--version-id <v>]
//    [--version-stage <v>] --output json`; returns SecretString only,
// marked secret (spec 6.10) regardless of how the caller binds it (§6).

ssm_get_parameter(name: String, --with_decryption: Bool = true): String
// `ssm get-parameter --name <name> [--with-decryption] --output json`;
// returns Value only, marked secret regardless of the caller's binding (§6).

cli(...args: Array<String>): CommandResult
// `aws <args...>`, quoted and joined as given; returns the full
// CommandResult without failing on a non-zero exit (§7).
```

## 5. Credential Handling

lask-terraform's design takes the position that a module should never accept
secrets as arguments, and should instead rely on the target environment's
ambient variables. `lask-aws` starts from the same position — every
credential parameter defaults to `""` and adds no override, so ambient
credentials already present in `--env` (an EC2/ECS instance role, a mounted
AWS profile, `~/.aws/credentials` on `#local`) apply completely unchanged —
but cannot stop there, for a reason specific to how Lask environments work:

- For `#local`, ambient host environment variables are already the base set
  (spec 10.6), so an `AWS_ACCESS_KEY_ID` exported before `lask run` reaches
  aws-cli with zero module code, exactly like lask-terraform's `TF_VAR_*`.
- For `#docker(...)`, they are not. Lask's environment-variable construction
  rules (spec 10.6) build a `docker` environment's variable set from the
  image's own defaults plus *explicit* overrides — there is no mechanism to
  forward the host's environment into a container implicitly. The only
  sanctioned way to set a variable for one containerized command is the
  shell-assignment prefix already used throughout this codebase (e.g.
  `example/04-webapp/main.lask`: `AWS_ACCESS_KEY_ID="#{key}" $[env] aws ...`)
  — and because this module, not the caller, builds the final `$[env] aws
  ...` command line, the prefix has to be assembled *inside* the module for
  a caller to reach it at all.

So `lask-aws` accepts three optional credential parameters
(`--access_key_id`, `--secret_access_key!!`, `--session_token!!`) on every
wrapped function, and threads them through `lib/creds.lask`'s
`credential_env` into a leading `AWS_ACCESS_KEY_ID='...' ...` assignment on
the command it runs — never written to a file, never appended to `--vars`
or any other data value, only ever the same shell-prefix idiom already
established as safe (`!!`-masked) elsewhere in this codebase (spec 6.10's
own worked example is `--secret_key!!: String = get_env("AWS_SECRET_ACCESS_KEY")`
used exactly this way).

This is a deliberate, narrow, and documented deviation from lask-terraform's
stricter stance — not a return to `example/04-webapp`'s original pattern of
duplicating the same three lines of credential wiring at every call site
(the problem this module exists partly to fix). The difference is *where*
the wiring lives: once, in `lib/creds.lask`, instead of once per call site.

## 6. Secret Masking of Read Results

`secretsmanager_get_secret_value` and `ssm_get_parameter` bind their result
through a `!!`-marked statement (`v!! = as_string(...)`) before returning it,
regardless of whether the *caller* marks anything. Per spec 6.10, masking is
registered by value, not by name or source, so once the module registers the
fetched secret, every later occurrence of that exact string in the command
execution log is masked — including the case where a careless caller passes
it straight into another wrapped function's arguments (e.g. forwards a
fetched secret into `cognito_idp_admin_set_user_password`'s `password`).
This does not require the `password` parameter itself to be `!!`-marked for
protection to apply; it is already registered by value at the point it was
read.

## 7. Error Mapping

| Situation | Behavior |
|---|---|
| Command exits 0 | Success; stdout (or typed record) returned. |
| Any non-zero exit, wrapped function | `fail(error(code, stderr))`, via the built-in failure of `$` (spec 6.6) inside `run_aws`. Uncaught, the exit code becomes the `lask` process exit code (spec 11.3) — CI semantics preserved for free, same as lask-terraform. |
| Any exit code, `cli()` | Never a failure: returns the full `CommandResult` (`code`, `stdout`, `stderr`) via `run_aws_raw` (`$*`), since an unwrapped subcommand's exit-code conventions (some are meaningful data, e.g. `s3api head-object` returning non-zero for "not found") are not known to this module. |
| `secretsmanager_get_secret_value` with no `SecretString` on the entry (a binary secret) | `fail(error(3, "secretsmanager get-secret-value: no SecretString for <id>"))` after a `has_key` check, mirroring lask-terraform's `output_value` diagnosable-error pattern. |
| Environment resolution / Docker daemon failures | Left to the runtime (`E-IO-ENV-RESOLVE` etc., spec 10.4, ch. 14); the module adds nothing. |

Unlike lask-terraform, no operation wrapped here has a Terraform-style
"detailed exit code" convention (`plan -detailed-exitcode`, `fmt -check`) —
every wrapped aws-cli invocation is 0-success/non-zero-failure, so `$`
inside `run_aws` (§8) is sufficient everywhere except `cli()`.

## 8. Internal Structure

```text
lask-aws/
  LICENSE
  README.md                 # usage, credential-handling notes, security notes
  main.lask                 # entire public API (types + functions)
  lib/
    args.lask                # pure aws-cli argument-string builders
    creds.lask                # credential env-var-prefix builder (effectful only via what it returns; the string itself is pure)
    exec.lask                 # `run_aws`/`run_aws_raw`: the one place `$[env] aws ...` is written
  example/
    main.lask                # consumer-shaped tasks mirroring lask/example/04-webapp's aws usage
  test/
    selftest.lask             # `lask run --module test/selftest.lask all` (LocalStack-backed subset; see doc/design.md §10)
```

`lib/args.lask` (internal, pure):

```lask
quote(s: String): String                                  // shell-quote one value
flag(cond: Bool, name: String): String                    // "" or "--name"
opt(name: String, value: String): String                  // "" or "--name 'value'"
repeated_opt(name: String, values: Array<String>): String // "--name 'a' --name 'b'"
multi_opt(name: String, values: Array<String>): String    // "--name 'a' 'b'"
global_opts(region: String, profile: String, endpoint_url: String): String // "--region 'x' --profile 'y' --endpoint-url 'z'"
join_args(parts: Array<String>): String                   // join(filter(_, != ""), " ")
count(xs: Array<String>): Number                           // reduce-based (no array length builtin)
contains(s: String, needle: String): Bool                  // used by test/selftest.lask only
```

`lib/creds.lask` (internal, pure):

```lask
env_assign(name: String, value: String): String    // "" or "NAME='value'"
credential_env(access_key_id: String, secret_access_key: String, session_token: String): String
```

`lib/exec.lask` (internal, effectful — the only file that executes commands):

```lask
run_aws(tail: String, creds: String, env: Environment): String        // $[env], fails on non-zero exit
run_aws_raw(tail: String, creds: String, env: Environment): CommandResult // $*[env], never fails
```

Constraint note, carried over from lask-terraform: helpers stay
**monomorphic** (user code cannot declare type variables, spec 4.4), so the
`as_map`/`as_string` cast helpers in `main.lask` (§4.3's decode step) exist
as small typed functions rather than a single generic `cast_to<T>` — the
same reasoning as lask-terraform's `decode_outputs`/`output_entry`.

## 9. Security and Observability Notes

- **Never pass long-lived credentials through `--vars`-style data files** —
  this module has no such mechanism to begin with; credentials only ever
  flow through the `!!`-masked shell-prefix path of §5.
- `secretsmanager_get_secret_value` and `ssm_get_parameter` mask their
  result unconditionally (§6); every other wrapped function returns
  aws-cli's ordinary (unmasked) output, so a caller must still mark a
  result `!!` at the binding site if it turns out to carry a secret this
  module did not already know to protect (e.g. a value returned via
  `cli()`).
- Cognito's `admin-create-user`/`admin-set-user-password` accept a
  `temporary_password`/`password` value the caller supplies; both
  parameters are `!!`-marked here, but the responsibility for generating a
  suitably strong value is the caller's.
- Everything else is inherited: command start/exit/output events give full
  audit visibility of every aws-cli invocation with zero module code, same
  as lask-terraform.

## 10. Testing Plan

- **Static gate**: `lask check --module main.lask` and `lask check` on
  `example/` — the cheapest CI step, and the only one that needs no AWS
  account or credentials at all.
- **Selftest**: `test/selftest.lask` starts
  [LocalStack](https://github.com/localstack/localstack) (community
  edition) with a plain `docker run` in the default `#local` environment,
  then exercises S3, STS, Secrets Manager, and SSM through this module's
  own functions in their normal `#docker("amazon/aws-cli:2.36.41")`
  environment — the same containerized path a real consumer uses, not a
  special test-only one — reaching LocalStack's host-published port via
  `host.docker.internal` (`--endpoint_url`). Run with `lask run --module
  test/selftest.lask all`; verified passing end-to-end (Rancher Desktop,
  macOS). CloudFront and Cognito Identity Provider are LocalStack Pro-only
  services, so `cloudfront_create_invalidation` and the `cognito_idp_*`
  functions are **not** exercised by the automated selftest; they are
  covered by `lask check` (static) and by their real use in
  `lask/example/04-webapp` against a real AWS account.
- A typed plan/response-diff style test (comparing before/after `s3_ls`
  output, `sts_get_caller_identity` returning a LocalStack-shaped identity)
  is the practical substitute for lask-terraform's plan-change assertions,
  since there is no equivalent "would this change anything" dry-run
  primitive shared across all of S3/STS/Secrets Manager/SSM.

## 11. Future Work

- Cognito/CloudFront selftest coverage once a LocalStack Pro license (or an
  equivalent free emulator) is available in CI.
- Additional services, added when a real consumer needs them rather than
  speculatively: ECR (`get-login-password` for a private registry a
  `#docker(...)` recipe needs to pull from), DynamoDB, Lambda
  (`update-function-code`), CloudWatch Logs (`tail`).
- A typed `Record` for `cognito_idp_admin_create_user`'s response, once a
  consumer needs to read fields off it rather than just confirming success.
- `--output` pass-through for the functions that currently force
  `--output json` internally, if a consumer wants `text`/`table` instead —
  deferred because it would require re-deriving each function's parsing
  logic per format.
