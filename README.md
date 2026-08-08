# Cantrill AWS SAA-C03 — Hands-On Labs

A collection of hands-on labs built and debugged in a real AWS account, following
Adrian Cantrill's AWS Certified Solutions Architect – Associate (SAA-C03) course.
Started with database deployment patterns via CloudFormation, migrated from Amazon
Linux 2 (AL2) to Amazon Linux 2023 (AL2023), and has since grown to cover VPC
endpoints, VPC peering, S3 static website hosting, and the CloudFormation
`CreationPolicy`/`cfn-signal` lifecycle pattern.

## Topics

- **`dbs/`** — WordPress + database backend variants (CloudFormation)
- **`efs/`** — Elastic File System labs (CloudFormation)
- **`cloud_formation/`** — S3 static website hosting with a custom-resource content copy (CloudFormation)
- **`cfinit_cfnhub_cfnsignal/`** — `CreationPolicy`/`cfn-signal` demo (CloudFormation)
- **`endpoint-gateway-labs/`** — Private VPC reachable only via Interface Endpoints (CloudFormation)
- **`vpc_peering_with_cloudformation/`** — Three-VPC base template for a manual VPC peering exercise (CloudFormation)

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
| [`efs/efs-with-wordpress/`](./efs/efs-with-wordpress) | Full VPC + Aurora MySQL + EFS + WordPress-on-EC2 stack, with EFS mount targets in every AZ. **Note:** this template is the original course version — it still has the `collectd` and `RequestType is 'Delete'` bugs described below, unlike the AL2023 rewrites elsewhere in this repo |

Each folder has its own README covering what's specific to that variant. This file
covers what's shared across all of them.

## `cloud_formation/` — S3 static website hosting

`cloud_formation/template.yaml` deploys a public S3 bucket configured for static
website hosting (`index.html`/`error.html`), plus a Lambda-backed custom resource
that copies a course's sample "Top 10 Cats" site content from a public S3 bucket
into it on stack create/update, and cleans the objects back out on delete. The
Lambda runs on `python3.13` and reports its own `WebsiteEndPoint` as a stack output.

## `cfinit_cfnhub_cfnsignal/` — CreationPolicy / cfn-signal demo

A small template used to demonstrate how `CreationPolicy` + `cfn-signal` make
CloudFormation wait for an EC2 instance's `UserData` to actually finish (not just
for the instance to start running) before marking the resource `CREATE_COMPLETE`.
See [`cfinit_cfnhub_cfnsignal/README.md`](./cfinit_cfnhub_cfnsignal/README.md) for
the full write-up, including where `cfn-init`/`cfn-hup` fit into the same family
of tools.

## `endpoint-gateway-labs/` — private VPC + Interface Endpoints

`endpoint-gateway-labs/template` builds the same three-tier, IPv6-enabled VPC used
elsewhere in this repo, then launches a single EC2 instance in a private app
subnet with **no Internet Gateway or NAT Gateway at all**. The instance is only
reachable for management through **VPC Interface Endpoints** for `ssm`,
`ec2messages`, and `ssmmessages`, deployed in the web subnets — a hands-on look at
how Interface Endpoints let a fully private instance still be reached over Session
Manager, and the starting point for adding a Gateway Endpoint (e.g. for S3) on top.

## `vpc_peering_with_cloudformation/` — three-VPC peering base

`vpc_peering_with_cloudformation/template` deploys three separate VPCs
(`10.16.0.0/16`, `10.17.0.0/16`, `10.18.0.0/16`), each with one subnet and one
private EC2 instance. VPC A also gets the same `ssm`/`ec2messages`/`ssmmessages`
Interface Endpoints as above, so its instance is reachable via Session Manager;
VPCs B and C are not. **The template intentionally does not create any peering
connections, routes, Internet Gateway, or NAT Gateway** — it's meant as the base
infrastructure for manually creating and testing VPC peering connections and the
route table entries they require, as a hands-on exercise on top of a working
starting point.

## Shared architecture

`dbs/`, `efs/two-instance-al2023`, `efs/efs-with-wordpress`,
`endpoint-gateway-labs/`, and `vpc_peering_with_cloudformation/` all deploy some
version of the same base network:

- Custom VPC across 3 AZs with public web, app, DB, and reserved subnet tiers
- IPv6 support via a custom-resource Lambda that works around a CloudFormation gap
  in IPv6 subnet auto-assignment
- CloudWatch Agent on EC2 instances, driven by a shared SSM Parameter config (where
  a CloudWatch Agent is deployed at all)

`cloud_formation/` and `cfinit_cfnhub_cfnsignal/` are smaller, purpose-built
templates that don't use this shared network — they exist to isolate a single
concept (static site hosting, `CreationPolicy`/`cfn-signal`) rather than to stand
up a full application stack.

## AL2 → AL2023 migration notes (applies to every AL2023 template here)

AL2023 removed or changed several things AL2 templates commonly relied on:

- **`amazon-linux-extras` is gone.** Packages now install directly via `dnf` from
  the base AL2023 repos — no extras topic needed.
- **`cfn-signal`/`cfn-init` are no longer preinstalled.** Requires an explicit
  `dnf install -y aws-cfn-bootstrap` before either binary is available.
- **`yum` → `dnf`** throughout.
- **Lambda runtime bump:** `python3.9` → `python3.12`/`python3.13` (3.9 is
  deprecated on Lambda).

## Bugs found and fixed (dbs/, efs/two-instance-al2023, endpoint-gateway-labs, vpc_peering_with_cloudformation*)

*`vpc_peering_with_cloudformation` doesn't include the IPv6 workaround Lambda at
all, so only the CloudWatch Agent fix is relevant where it applies.

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

> **Not yet applied to `efs/efs-with-wordpress/`.** That template still has both
> bugs in their original form (`collectd` block present, `RequestType is 'Delete'`)
> — it was added as the original course reference version rather than ported
> forward yet.

## Debugging approach

Both failures were diagnosed by connecting to the EC2 instances via **AWS Systems
Manager Session Manager** (no SSH keys or security group changes needed, since the
instance role already includes SSM access) and tailing
`/var/log/cloud-init-output.log` in real time during a deploy with a temporarily
extended `ResourceSignal` timeout — catching the exact failing command before
CloudFormation rolled the stack back and destroyed the evidence. The same
Session Manager approach is what makes the fully-private instances in
`endpoint-gateway-labs/` and `vpc_peering_with_cloudformation/` reachable at all,
since neither has SSH access from the internet.

## Usage

Most template folders include their own `aws cloudformation create-stack` example
with the parameters specific to that variant — see
[`cfinit_cfnhub_cfnsignal/README.md`](./cfinit_cfnhub_cfnsignal/README.md) for a
worked example. For the folders without a dedicated README:

```bash
# cloud_formation/ — S3 static website hosting
aws cloudformation create-stack \
  --stack-name top10cats-site \
  --template-body file://cloud_formation/template.yaml \
  --capabilities CAPABILITY_IAM

# endpoint-gateway-labs/ — private VPC + Interface Endpoints
aws cloudformation create-stack \
  --stack-name endpoint-gateway-lab \
  --template-body file://endpoint-gateway-labs/template \
  --capabilities CAPABILITY_IAM

# vpc_peering_with_cloudformation/ — three-VPC peering base
aws cloudformation create-stack \
  --stack-name vpc-peering-lab \
  --template-body file://vpc_peering_with_cloudformation/template \
  --capabilities CAPABILITY_IAM
```
