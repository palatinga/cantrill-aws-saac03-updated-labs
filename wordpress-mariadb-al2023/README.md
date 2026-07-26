# WordPress + MariaDB on EC2 (AL2023)

WordPress on one EC2 instance, MariaDB 10.5 running on a second, separate EC2
instance as the DB backend. See the [repo-level README](../README.md) for shared
architecture, migration notes, and both fixed bugs — this file only covers what's
specific to this variant.

## What's different about this variant

- MariaDB runs as a self-managed service (`mariadb105-server`) on its own EC2
  instance rather than a managed RDS instance — you own patching, backups, and
  failover here.
- `WordpressEC2`'s `wp-config.php` points at `${MariaDBEC2.PrivateDnsName}` rather
  than an RDS endpoint.
- No `DBVersion` parameter — MariaDB's version is fixed by the package name
  (`mariadb105`) rather than being selectable like an RDS engine version.

## Usage

```bash
aws cloudformation create-stack \
  --stack-name my-wordpress-mariadb-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=DBPassword,ParameterValue=<your-password> \
               ParameterKey=DBRootPassword,ParameterValue=<your-password> \
  --capabilities CAPABILITY_IAM
```

> `DBPassword` and `DBRootPassword` default to placeholder values in the template
> for lab use. Override them for anything beyond a throwaway sandbox.
