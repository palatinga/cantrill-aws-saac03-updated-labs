# A4L UserData / CreationPolicy Demo

A CloudFormation template used to illustrate how EC2 instance provisioning can be synchronized with in-instance configuration using **`CreationPolicy`**, **`cfn-signal`**, and `UserData`. This is a common pattern from Adrian Cantrill's AWS courses for teaching how CloudFormation "waits" for an instance to actually finish configuring itself before marking the stack resource as `CREATE_COMPLETE`.

## What this template does

1. Launches a `t3.micro` EC2 instance running the latest Amazon Linux 2023 AMI (resolved dynamically via an SSM public parameter).
2. Opens a Security Group for SSH (22) and HTTP (80) from anywhere.
3. Creates an empty S3 bucket (included in the course demo, not otherwise wired into the instance).
4. Uses `UserData` to install and start `httpd`, deploy a simple HTML page containing a customizable `Message` parameter, and simulate a long-running configuration step (`sleep 300`).
5. Signals CloudFormation once configuration is complete, using `cfn-signal`.
6. Uses a `CreationPolicy` on the instance so the stack **waits** for that signal (up to 15 minutes) before considering the `Instance` resource successfully created.

## Why this matters: the problem being solved

By default, CloudFormation considers an `AWS::EC2::Instance` resource `CREATE_COMPLETE` the moment the EC2 API confirms the instance is **running** — not when the software inside it has finished installing and configuring itself.

That's a problem for real-world stacks:

- If a dependent resource (e.g., a Load Balancer target group check, or a second instance that depends on this one) is created immediately afterward, it may find the web server not yet listening on port 80.
- Automation or monitoring might assume "stack complete" means "service is live," which isn't true yet.

`UserData` alone runs asynchronously — CloudFormation fires it off and moves on. There's no built-in feedback loop telling CloudFormation "the actual application-level setup is done."

## How the pattern works

### 1. `CreationPolicy`

```yaml
CreationPolicy:
  ResourceSignal:
    Timeout: PT15M
```

This tells CloudFormation: *don't mark this resource as `CREATE_COMPLETE` until it receives a success signal, and don't wait longer than 15 minutes for it.* If no signal (or a failure signal) arrives within the timeout, the resource — and by extension the stack — rolls back / fails.

### 2. `cfn-signal`

At the very end of `UserData`, after `httpd` is installed and the page is deployed:

```bash
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource Instance --region ${AWS::Region}
```

- `-e $?` passes the exit code of the *previous* command as the signal's success/failure status. `0` = success, anything else = failure.
- `--stack`, `--resource`, and `--region` tell CloudFormation exactly which resource's `CreationPolicy` this signal satisfies.

This is what closes the loop: the instance itself is reporting back to CloudFormation that its configuration work is genuinely finished, not just that it booted.

### 3. Why `sleep 300` is in there

This simulates a slow configuration step (patching, downloading dependencies, compiling something, etc.). Without a `CreationPolicy`/`cfn-signal`, CloudFormation wouldn't know or care — it'd already say `CREATE_COMPLETE`. With them in place, you can watch the stack legitimately sit in `CREATE_IN_PROGRESS` for the full 5 minutes, then flip to `CREATE_COMPLETE` right as the signal arrives — a visible demonstration of the wait behavior.

### Related concepts not used in this template (but part of the same family)

- **`WaitCondition` / `WaitConditionHandle`** — an older, more manual mechanism for the same goal. Instead of a `CreationPolicy` directly on a resource, you create a separate `AWS::CloudFormation::WaitConditionHandle`, generate a pre-signed URL, and have the instance `curl` a signal payload to that URL. More flexible (can be used outside of resource creation, e.g., mid-stack checkpoints) but more manual than `CreationPolicy` + `cfn-signal`.
- **`cfn-init`** — a helper script (also in the `aws-cfn-bootstrap` package) that reads a `AWS::CloudFormation::Init` metadata block on the resource and performs declarative configuration: installing packages, writing files, creating users, running commands — instead of a monolithic `UserData` script. Typically paired with `cfn-signal` to report success once `cfn-init` finishes.
- **`cfn-hup`** — a daemon (also part of `aws-cfn-bootstrap`) that runs continuously on the instance, polls for changes to the stack's metadata, and re-triggers `cfn-init` when it detects an update. This is what enables **`UpdateReplacePolicy`/`Update`**-time reconfiguration of running instances without replacing them — useful when you update a stack's `AWS::CloudFormation::Init` metadata and want existing instances to pick up the change.

This template only uses `CreationPolicy` + `cfn-signal` for simplicity, but in a `cfn-init`/`cfn-hup` version, the `UserData` would instead call `cfn-init` (which reads a `Metadata` block to do the `httpd` install/config), and `cfn-hup` would be installed and configured to watch for future metadata changes.

## Parameters

| Parameter     | Description                                   | Default                                                              |
|---------------|------------------------------------------------|-----------------------------------------------------------------------|
| `LatestAmiId` | Amazon Linux AMI resolved via SSM public param | `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64` |
| `Message`     | Text displayed on the deployed HTML page        | `Cats are the best`                                                    |

## Resources created

| Logical ID              | Type                        | Purpose                                      |
|--------------------------|-----------------------------|-----------------------------------------------|
| `InstanceSecurityGroup`  | `AWS::EC2::SecurityGroup`   | Allows inbound SSH (22) and HTTP (80)         |
| `Bucket`                 | `AWS::S3::Bucket`           | Empty demo bucket (not used by the instance)  |
| `Instance`                | `AWS::EC2::Instance`        | Web server instance with `CreationPolicy`      |

## Deploying

```bash
aws cloudformation create-stack \
  --stack-name a4l-userdata-test \
  --template-body file://template.yaml \
  --parameters ParameterKey=Message,ParameterValue="Dogs are the best"
```

Watch the stack events — the `Instance` resource will remain `CREATE_IN_PROGRESS` for roughly 5 minutes (the `sleep 300`) before receiving its signal and completing.

To observe a **timeout/failure**, either:
- Break the `cfn-signal` call (wrong `--stack`/`--resource`/`--region`), or
- Extend `sleep` beyond the `CreationPolicy` `Timeout` (15 minutes).

In either case, the stack will roll back once the timeout is reached.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name a4l-userdata-test
```

> Note: the `Bucket` resource has no `DeletionPolicy`, so CloudFormation will attempt to delete it on stack deletion — this will **fail if the bucket is not empty**, since CloudFormation does not empty buckets automatically.

## Security note

The security group opens SSH (22) and HTTP (80) to `0.0.0.0/0` (the entire internet). This is fine for a short-lived training/demo stack but should be scoped down (e.g., to your IP, or removed entirely if SSH access isn't needed) for anything longer-lived.
