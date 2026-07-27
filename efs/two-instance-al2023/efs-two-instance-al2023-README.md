# EFS Two-Instance Base (AL2023)

Base VPC template with two plain EC2 instances (`EC2InstanceA` in one AZ,
`EC2InstanceB` in another), intended as the starting point for an EFS mounting
demo — no web server, database, or application software installed. See the
[repo-level README](../../README.md) for shared architecture and migration notes.

## What this deploys

- The same custom VPC/subnet/IPv6 layout used across this repo's other labs
- Two bare EC2 instances in separate AZs (`SubnetWEBA` / `SubnetWEBB`), each with
  an instance role that includes `AmazonElasticFileSystemClientFullAccess` so
  they're ready to mount an EFS file system once one is created and attached
- No EFS file system itself is created by this template — it's deliberately just
  the network + compute base, ready for an `AWS::EFS::FileSystem` and mount
  targets to be added on top

## AL2023 notes specific to this template

- The original template called `/opt/aws/bin/cfn-signal` directly with no
  `aws-cfn-bootstrap` install step at all — this worked on AL2 only because that
  package was preinstalled. Added `dnf install -y aws-cfn-bootstrap` explicitly,
  since AL2023 doesn't ship it.
- `CWAgentConfig` originally logged `/var/log/httpd/access_log` and
  `/var/log/httpd/error_log`, left over from being copied from a WordPress-based
  template — but this template never installs `httpd`. Removed those two log
  paths since they'd never exist; only `/var/log/secure` remains. If a web server
  gets added to these instances later, the right log paths should be added back.
- Same IPv6 workaround Lambda fix and `collectd` removal as the other templates
  in this repo — see the [repo-level README](../../README.md) for details.

## Usage

```bash
aws cloudformation create-stack \
  --stack-name my-efs-demo-base \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_IAM
```

No parameters beyond the AMI (defaulted) are required — this template has no
database or application-specific inputs.
