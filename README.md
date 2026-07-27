# Cantrill AWS SAA-C03 — Hands-On Labs

A collection of hands-on labs built and debugged in a real AWS account, following
Adrian Cantrill's AWS Certified Solutions Architect – Associate (SAA-C03) course.
Organized by topic as the course progresses, starting with database deployment
patterns via CloudFormation, based on Animals4Life (A4L) course material and
migrated from Amazon Linux 2 (AL2) to Amazon Linux 2023 (AL2023).

## Topics

- **`dbs/`** — WordPress + database backend variants (CloudFormation)
- **`efs/`** — Elastic File System labs (CloudFormation)

## Templates in `dbs/`

| Folder | What it deploys |
|---|---|
| [`dbs/wordpress-mariadb-al2023/`](./dbs/wordpress-mariadb-al2023) | WordPress on EC2 + a separate EC2 instance running MariaDB as the DB backend |
| [`dbs/wordpress-rds-al2023/`](./dbs/wordpress-rds-al2023) | WordPress on EC2 + Amazon RDS for MySQL as the managed DB backend |
| [`dbs/wordpress-aurora-al2023/`](./dbs/wordpress-aurora-al2023) | WordPress on EC2 + an Amazon Aurora MySQL cluster (writer + reader) as the DB backend |

## Templates in `efs/`

| Folder | What it deploys |
|---|---|
| [`efs/two-instance-al2023/`](./efs/two-instance-al2023) | Base VPC + two plain EC2 instances in separate AZs, ready to mount an EFS file system (no EFS resource included yet) |

Each folder has its own README covering what's specific to that variant. This file
covers what's shared across all of them.

## Shared architecture

All templates in this repo deploy the same base network:

- Custom VPC across 3 AZs with public web, app, DB, and reserved subnet tiers
- IPv6 support via a custom-resource Lambda that works around a CloudFormation gap
  in IPv6 subnet auto-assignment
- CloudWatch Agent on EC2 instances, driven by a shared SSM Parameter config

## AL2 → AL2023 migration notes (applies to every template here)

AL2023 removed or changed several things AL2 templates commonly relied on:

- **`amazon-linux-extras` is gone.** Packages now install directly via `dnf` from
  the base AL2023 repos — no extras topic needed.
- **`cfn-signal`/`cfn-init` are no longer preinstalled.** Requires an explicit
  `dnf install -y aws-cfn-bootstrap` before either binary is available.
- **`yum` → `dnf`** throughout.
- **Lambda runtime bump:** `python3.9` → `python3.12` (3.9 is deprecated on Lambda).

## Bugs found and fixed in every template here

### CloudWatch Agent config validation failure (silent UserData failure)

The shared `CWAgentConfig` SSM parameter originally included a `collectd` block
under `metrics_collected`. The CloudWatch Agent's config validator checks for
`/usr/share/collectd/types.db` whenever that block is present. The AL2 template
worked around this with `mkdir -p /usr/share/collectd/ && touch .../types.db`;
that workaround was dropped in the AL2023 rewrite along with `collectd` itself
(never installed on AL2023), which meant `amazon-cloudwatch-agent-ctl` exited
non-zero, `set -e` killed the UserData script immediately, and `cfn-signal` never
ran. Both EC2 instances would sit at "Initializing" until the `ResourceSignal`
timeout, followed by a full stack rollback.

**Fix:** removed the `collectd` block from `CWAgentConfig` entirely, since nothing
here uses it.

### IPv6 workaround Lambda: broken `cfnresponse.send()` on stack deletion

The custom resource Lambda that enables IPv6 auto-assignment on the web subnets
had two issues in its `Delete` handling: `cfnresponse.send()` was called without a
`responseData` argument, and the `RequestType` comparison used `is` instead of
`==` (an unreliable identity check for strings). Together these could leave the
custom resource without a completed signal back to CloudFormation on stack
deletion, causing the stack to hang in `DELETE_IN_PROGRESS` until timeout.

**Fix:** corrected the comparison to `==`, always pass an explicit `responseData`
dict, and wrapped the handler body in `try/except` so any failure still sends a
`SUCCESS`/`FAILED` signal rather than leaving the custom resource hanging.

## Debugging approach

Both failures were diagnosed by connecting to the EC2 instances via **AWS Systems
Manager Session Manager** (no SSH keys or security group changes needed, since the
instance role already includes SSM access) and tailing
`/var/log/cloud-init-output.log` in real time during a deploy with a temporarily
extended `ResourceSignal` timeout — catching the exact failing command before
CloudFormation rolled the stack back and destroyed the evidence.

## Usage

Each template folder includes its own `aws cloudformation create-stack` example
with the parameters specific to that variant. 
