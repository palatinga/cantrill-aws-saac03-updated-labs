# AWS CloudFormation: Amazon Linux 2023 WordPress + MariaDB Stack

A CloudFormation template originally built against Amazon Linux 2 (AL2), migrated to
Amazon Linux 2023 (AL2023) and debugged end-to-end through failed deployments in a
real AWS account.

## What this deploys

- Custom VPC with public web, app, DB, and reserved subnet tiers across 3 AZs, plus
  IPv6 support
- An EC2 instance running WordPress (Apache + PHP 8.1)
- A separate EC2 instance running MariaDB 10.5 as the WordPress database backend
- CloudWatch Agent on both instances, driven by a shared SSM Parameter config
- A Lambda-backed custom resource that works around a CloudFormation IPv6 subnet
  auto-assignment gap

## Migration notes: AL2 → AL2023

AL2023 removed several things AL2 templates commonly relied on, and this repo
documents the concrete fixes:

- **`amazon-linux-extras` is gone.** Packages (`httpd`, `mariadb105`,
  `mariadb105-server`, `php8.1`, etc.) now install directly via `dnf` from the base
  AL2023 repos — no extras topic needed.
- **`cfn-signal`/`cfn-init` are no longer preinstalled.** AL2023 requires an explicit
  `dnf install -y aws-cfn-bootstrap` before either binary is available.
- **`yum` → `dnf`** throughout.
- **Lambda runtime bump:** `python3.9` → `python3.12` (3.9 is deprecated on Lambda).

## Bugs found and fixed during deployment

Two separate failures surfaced while adapting this template, both silent under
`set -e` — CloudFormation only reported a signal timeout, with no indication of
*why* the script died.

### 1. CloudWatch Agent config validation failure (root cause of stack rollback)

The shared `CWAgentConfig` SSM parameter still included a `collectd` block under
`metrics_collected`. The CloudWatch Agent's config validator checks for
`/usr/share/collectd/types.db` whenever that block is present. The original AL2
template created that file as a workaround (`mkdir -p /usr/share/collectd/ && touch
.../types.db`); this step was dropped during the AL2023 rewrite, and `collectd` was
never installed on AL2023 in the first place. Result: `amazon-cloudwatch-agent-ctl`
exited non-zero, `set -e` killed the UserData script immediately, and `cfn-signal`
never ran — leaving both `WordpressEC2` and `MariaDBEC2` stuck until the 15-minute
`ResourceSignal` timeout, followed by full stack rollback.

**Fix:** removed the `collectd` block from `CWAgentConfig` entirely, since nothing
in this stack uses it.

### 2. IPv6 workaround Lambda: broken `cfnresponse.send()` on stack deletion

The custom resource Lambda that enables IPv6 auto-assignment on the web subnets had
two issues in its `Delete` handling:

- `cfnresponse.send()` was called without a `responseData` argument on delete,
  which can prevent a valid signal from reaching CloudFormation.
- The `RequestType` comparison used `is` instead of `==` — an identity check, not
  an equality check, which is unreliable for string comparison in Python.

Together these could leave the custom resource without a completed signal back to
CloudFormation on stack deletion, causing the stack to hang in
`DELETE_IN_PROGRESS`/get stuck until timeout.

**Fix:** corrected the comparison to `==`, always pass an explicit `responseData`
dict, and wrapped the handler body in `try/except` so any failure (e.g. the subnet
already being gone) still sends a `SUCCESS`/`FAILED` signal rather than leaving the
custom resource hanging.

## Debugging approach

Both failures were diagnosed by connecting to the EC2 instances via **AWS Systems
Manager Session Manager** (no SSH keys or security group changes needed, since the
instance role already includes SSM access) and tailing
`/var/log/cloud-init-output.log` in real time during a deploy with a temporarily
extended `ResourceSignal` timeout — catching the exact failing command before
CloudFormation rolled the stack back and destroyed the evidence.

## Usage

```bash
aws cloudformation create-stack \
  --stack-name my-wordpress-stack \
  --template-body file://a4l-vpc-wordpress-al2023.yaml \
  --parameters ParameterKey=DBPassword,ParameterValue=<your-password> \
               ParameterKey=DBRootPassword,ParameterValue=<your-password> \
  --capabilities CAPABILITY_IAM
```

> Note: `DBPassword` and `DBRootPassword` default to placeholder values in the
> template for lab use. Override them for anything beyond a throwaway sandbox.
