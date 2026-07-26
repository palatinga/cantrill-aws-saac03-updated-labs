# WordPress + RDS for MySQL (AL2023)

WordPress on a single EC2 instance, with Amazon RDS for MySQL as a managed DB
backend. See the [repo-level README](../../README.md) for shared architecture,
migration notes, and both fixed bugs — this file only covers what's specific to
this variant.

## What's different about this variant

- The DB is a managed `AWS::RDS::DBInstance` rather than a second EC2 instance —
  AWS handles patching, backups, and (optionally) Multi-AZ failover.
- `wp-config.php` points at `${DB.Endpoint.Address}` instead of a private DNS name.
- Includes a `DBVersion` parameter and an `update_wp_ip.sh` script that rewrites
  WordPress's stored site URL to the instance's public IP after first boot (needed
  because the DB isn't populated with a real URL until WordPress's install wizard
  runs, so this can't happen before the `cfn-signal`).
- `MultiAZ` parameter lets you toggle RDS Multi-AZ (`"True"`/`"False"`) — leave it
  `"False"` for lab/sandbox use to avoid doubling RDS cost.

## MySQL version note

`DBVersion` defaults to `8.4.8`. MySQL 8.0 reaches end of standard support on
**July 31, 2026** — after that it moves to paid RDS Extended Support. 8.4 is the
current supported major version as a replacement. Since this template creates a
brand-new database rather than upgrading an existing one, the main 8.0 → 8.4
compatibility concern (the `mysql_native_password` auth plugin being disabled by
default in 8.4) doesn't apply here.

Before deploying, confirm the exact minor version is available in your region:

```bash
aws rds describe-db-engine-versions --engine mysql --engine-version 8.4.8
```

## Usage

```bash
aws cloudformation create-stack \
  --stack-name my-wordpress-rds-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=DBPassword,ParameterValue=<your-password> \
               ParameterKey=DBRootPassword,ParameterValue=<your-password> \
  --capabilities CAPABILITY_IAM
```

> `DBPassword` and `DBRootPassword` default to placeholder values in the template
> for lab use. Override them for anything beyond a throwaway sandbox.
