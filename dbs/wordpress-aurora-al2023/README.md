# WordPress + Aurora MySQL Cluster (AL2023)

WordPress on a single EC2 instance, with an Amazon Aurora MySQL cluster (writer +
reader) as the DB backend. See the [repo-level README](../../README.md) for shared
architecture, migration notes, and both fixed bugs — this file only covers what's
specific to this variant.

## What's different about this variant

- The DB is an `AWS::RDS::DBCluster` (Aurora) with two `AWS::RDS::DBInstance`
  members (`AuroraInstance1`/`AuroraInstance2`) rather than a single RDS instance
  or a self-managed EC2 database — this gives a writer + reader topology out of
  the box.
- `wp-config.php` points at `${DBCluster.Endpoint.Address}` (the cluster's writer
  endpoint) rather than a single-instance endpoint.
- Supports restoring from an existing snapshot via the `DatabaseRestoreSnapshot`
  parameter — leave it blank for a fresh DB, or supply a snapshot identifier/ARN
  to migrate from an existing instance instead of creating a new one.
- `WordpressEC2` explicitly `DependsOn` both Aurora instances and the cluster, so
  the web server won't start configuring itself against a DB that isn't ready yet.
- The IMDS token-fetching pattern in `update_wp_ip.sh` uses IMDSv2
  (`X-aws-ec2-metadata-token`) rather than the older IMDSv1-style plain GET used
  in the other two templates in this repo.

## Aurora version note

`DBVersion` defaults to `8.0.mysql_aurora.3.10.4`. The 3.10 line is Aurora MySQL's
current **long-term support (LTS)** release, compatible with MySQL 8.0.42.
`AutoMinorVersionUpgrade` is deliberately set to `false` on both instances — AWS
advises against enabling it on LTS release lines, since an automatic minor upgrade
could move the cluster off the LTS track onto a non-LTS version.

## Usage

```bash
aws cloudformation create-stack \
  --stack-name my-wordpress-aurora-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=DBPassword,ParameterValue=<your-password> \
               ParameterKey=DBRootPassword,ParameterValue=<your-password> \
               ParameterKey=DatabaseRestoreSnapshot,ParameterValue="" \
  --capabilities CAPABILITY_IAM
```

> `DBPassword` and `DBRootPassword` default to placeholder values in the template
> for lab use. Override them for anything beyond a throwaway sandbox.
