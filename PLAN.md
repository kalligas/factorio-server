# Factorio server on AWS: build plan

This file records the plan for building the Factorio server, the decisions behind it, and progress so far. Claude updates this file as each step is completed or the plan changes.

## Goal

Two players share one Factorio world hosted on AWS. Either player can start the server alone, play alone or together, and stop the server, without needing the other player. The server costs money only while someone is playing. Terraform defines all AWS infrastructure.

The project is also a learning exercise. The repository owner writes the code and runs the commands. Claude explains each concept, links the relevant documentation, and reviews each diff before it is committed. Each pull request is small enough for a person to review and understand on its own.

## Progress

| Step | Topic | Status |
| --- | --- | --- |
| 0 | Repository and git identity | done |
| 1 | Personal AWS account | not started |
| PR 1 | Remote state bucket | not started |
| PR 2 | Terraform skeleton and budget alarm | not started |
| PR 3 | Network | not started |
| PR 4 | First EC2 instance with SSM access | not started |
| PR 5 | Data volume and first-boot setup | not started |
| PR 6 | S3 bucket for scripts and backups | not started |
| PR 7 | Factorio running (first playable milestone) | not started |
| PR 8 | Player CLI | not started |
| PR 9 | Save backups | not started |
| PR 10 | Idle auto-stop | not started |
| PR 11 | Player IAM users | not started |
| PR 12 | Stop guard | not started |
| Later | Space Age upgrade | not started |

## Decisions

Each decision lists the choice and the reason for the choice.

- **One shared world.** The server hosts a single save file. Either player can play that world alone or together with the other player.
- **Start and stop with an AWS CLI script.** Each player has an IAM user that can only start, stop, and inspect the server instance. The reason is that the script is simple and needs no extra service such as a bot or a web endpoint.
- **Base game first, Space Age later.** The server starts with the Factorio 2.0 base game. A Terraform variable named `space_age` turns on the Space Age expansion later. Turning on Space Age for an existing base-game save works, but the change cannot be undone for that save.
- **Region `eu-central-1` (Frankfurt).** Both players are in Greece. Frankfurt is chosen over `eu-south-1` (Milan) because Milan is an opt-in region that must be turned on manually, and latency from Greece to the two regions is similar.
- **Personal accounts only.** The GitHub repository is `kalligas/factorio-server` on the personal account `kalligas`. The AWS account is a new personal account accessed through the CLI profile `personal`. The work accounts (`michalis-kalligas-wikifarmer` on GitHub, `wikifarmer` on AWS) are never used for this project.

## Target architecture

- **Compute:** one EC2 instance (x86_64, default type `c7a.large` with 2 vCPUs and 4 GiB of memory) running Amazon Linux 2023. The instance is stopped, not terminated, when nobody plays, because AWS does not bill compute time for a stopped instance.
- **Game:** the `factoriotools/factorio` Docker image, run by a systemd unit. The unit stops the container with the SIGINT signal, because Factorio saves the world when it receives SIGINT.
- **World storage:** a separate EBS volume (a network disk) holds the save files. Replacing the instance does not delete this volume.
- **Backups and scripts:** a versioned S3 bucket holds save backups and the server-side scripts. The instance copies the scripts from S3 on every boot, because EC2 user data (the first-boot setup script) runs only on the first boot.
- **Idle auto-stop:** a timer on the instance checks the number of online players every minute and powers off the instance after 15 minutes with no players.
- **Network access:** the only open inbound port is UDP 34197, the Factorio game port. Shell access uses AWS Systems Manager (SSM) Session Manager, so no SSH port is open.
- **Player permissions:** one IAM user per player, limited to starting, stopping, and inspecting the server instance.
- **Terraform state:** Terraform state is stored in an S3 bucket with lockfile locking (`use_lockfile = true`), which requires Terraform 1.10 or newer.
- **Estimated cost:** about $7 per month for about 40 hours of play.

## Workflow for each pull request

1. Claude explains the concept and the AWS resources involved and links the Terraform provider documentation.
2. The repository owner creates a branch, writes the files, and runs `terraform fmt`, `terraform validate`, and `terraform plan`.
3. Claude reviews `git diff` and the plan output.
4. The repository owner runs `terraform apply`, tests the result, commits, pushes the branch, opens the pull request on github.com, and merges it.

Git authenticates to GitHub over SSH only, with no API token. The remote `git@github.com:kalligas/factorio-server.git` uses the `github.com` entry in `~/.ssh/config`, which selects the personal key `~/.ssh/id_personal`. SSH can push and pull but cannot create repositories or pull requests, so pull requests are opened on github.com from the link that `git push` prints.

## Steps

### Step 0: Repository and git identity (direct commit to `main`)

- **Concept:** `~/.gitconfig` contains `includeIf "gitdir:~/projects/"`, which applies the work identity to every repository under `~/projects/`. Repository-local config is read after global config, so a repository-local identity overrides the work identity.
- **Tasks:** run `git init -b main`, `git config user.name kalligas`, and `git config user.email mkalligas1997@gmail.com`. Add `.gitignore` (ignore `.terraform/`, `*.tfstate`, `*.tfstate.*`, `terraform.tfvars`, `*.tfplan`; keep `.terraform.lock.hcl`), `README.md`, and this file. Create the empty private repository `kalligas/factorio-server` on github.com, then run `git remote add origin git@github.com:kalligas/factorio-server.git` and `git push -u origin main`.
- **Check:** `git log --format='%an <%ae>'` prints `kalligas <mkalligas1997@gmail.com>`. `git remote -v` prints `git@github.com:kalligas/factorio-server.git`.

### Step 1: Personal AWS account (no code)

- **Concept:** the root user, IAM identities, multi-factor authentication (MFA), and named AWS CLI profiles.
- **Tasks:** create the account, turn on MFA for the root user, create an admin IAM user or IAM Identity Center user for daily use, and configure the profile `personal` in `~/.aws/config`.
- **Check:** `aws sts get-caller-identity --profile personal` prints the new account ID.

### PR 1: Remote state bucket (`bootstrap/`)

- **Concept:** Terraform state records which real resources Terraform manages. Storing state in S3 keeps state safe and shareable. The state bucket itself is created by a small separate stack with local state, because a backend bucket must exist before Terraform can use it.
- **Files:** `bootstrap/main.tf`, containing an S3 bucket with versioning, encryption, and a public access block.
- **Check:** the bucket exists, and the local `terraform.tfstate` file is not tracked by git.

### PR 2: Terraform skeleton and budget alarm (`terraform/`)

- **Concept:** `terraform` and `provider` blocks, the S3 backend with `use_lockfile`, input variables and `terraform.tfvars`, and `default_tags`.
- **Files:** `versions.tf`, `backend.tf`, `providers.tf`, `variables.tf`, `terraform.tfvars.example`, and `budget.tf` (an `aws_budgets_budget` that emails alerts at 80% and 100% of $15 per month).
- **Check:** the state file appears in the state bucket, and a budget notification email address is confirmed.

### PR 3: Network

- **Concept:** a VPC (a private network), a subnet, an internet gateway, a route table, and a security group (a firewall), and the reason a public game server needs each one.
- **Files:** `network.tf`.
- **Check:** a second `terraform plan` after apply shows no changes, and the security group allows only UDP 34197 inbound.

### PR 4: First EC2 instance with SSM access

- **Concept:** AMI data sources, IAM roles and instance profiles, Session Manager compared with SSH, `lifecycle { ignore_changes = [ami] }`, and `instance_initiated_shutdown_behavior = "stop"`.
- **Files:** `compute.tf`, `iam.tf` (an instance role with the `AmazonSSMManagedInstanceCore` policy), and `outputs.tf`.
- **Check:** `aws ssm start-session --target <instance-id>` opens a shell. Stop the instance afterwards to avoid charges.

### PR 5: Data volume and first-boot setup

- **Concept:** keeping data separate from the instance, `prevent_destroy`, user data and cloud-init, and `templatefile()`.
- **Files:** an EBS volume and volume attachment in `compute.tf`, and `templates/user_data.sh.tftpl` (installs Docker, formats the volume only when the volume is empty, and mounts the volume at `/opt/factorio`).
- **Check:** in a Session Manager shell, `df -h /opt/factorio` shows the volume and `docker info` succeeds.

### PR 6: S3 bucket for scripts and backups

- **Concept:** delivering server scripts through S3 so that changes take effect on the next boot, `aws_s3_object` with `etag`, and lifecycle rules that delete old object versions.
- **Files:** `storage.tf`, a boot-time sync unit added to the user data, and the first file in `server/`.
- **Check:** after a reboot, the instance has the latest files from `server/`.

### PR 7: Factorio running (first playable milestone)

- **Concept:** systemd units for containers, SSM Parameter Store with the SecureString type for the game password, and the `factoriotools/factorio` image options (`DLC_SPACE_AGE`, save loading, admin list).
- **Files:** `ssm.tf` (game password, server name, admin list, `space_age`), `server/factorio.service`, and `server/prestart.sh` (reads the parameters, writes `server-settings.json`, and pulls the image).
- **Check:** the Factorio client connects to `<public-ip>:34197` with the password, and both players can join.

### PR 8: Player CLI (`bin/factorio`)

- **Concept:** shell scripting with the AWS CLI, finding an instance by tag, and `aws ec2 wait`.
- **Files:** `bin/factorio` with the subcommands `start`, `stop`, `status`, and `ip`.
- **Check:** `bin/factorio start` prints the server address. After `bin/factorio stop` and another `bin/factorio start`, the world progress is still present.

### PR 9: Save backups

- **Concept:** systemd timers, `ExecStopPost`, and unit ordering that makes the shutdown backup run before the network stops.
- **Files:** `server/backup.sh`, `server/backup.service`, `server/backup.timer`, and S3 write permission on the instance role.
- **Check:** a copy of the save appears in S3 after the server stops.

### PR 10: Idle auto-stop

- **Concept:** RCON (Factorio's remote console protocol), writing state to SSM Parameter Store, and power-off logic with a grace period after boot.
- **Files:** `server/idle-check.sh`, `server/idle-check.service`, `server/idle-check.timer`, and the `/factorio/players_online` parameter.
- **Check:** after the server starts and nobody joins, the instance reaches the `stopped` state when the grace period ends.

### PR 11: Player IAM users

- **Concept:** least-privilege IAM policies, resource-level permissions and conditions, and creating access keys outside Terraform, because Terraform would otherwise store the secret key in state.
- **Files:** an IAM group, a policy, and users in `iam.tf`.
- **Check:** the second player's profile can run `start` and `status`, and `aws ec2 terminate-instances` is denied for that profile.

### PR 12: Stop guard

- **Concept:** reading the shared online-player count from the CLI.
- **Files:** `bin/factorio stop` refuses to stop the server while players are online unless the `--force` flag is given.
- **Check:** while a player is connected, `bin/factorio stop` refuses and `bin/factorio stop --force` stops the server.

### Later: Space Age upgrade

- **Tasks:** set `space_age = true`, and optionally `instance_type = "m7a.large"` (8 GiB of memory), in `terraform.tfvars`. Run `terraform apply` and start the server.

## End-state checks

- `terraform validate` passes in both stacks, and `shellcheck` passes on `server/*.sh` and `bin/factorio`.
- Either player alone can run `bin/factorio start`, join the shared world, and run `bin/factorio stop`.
- The world survives a stop and start, and a backup copy exists in S3.
- The instance stops itself after 15 minutes with no players.
- A player's IAM credentials allow only start, stop, and status.

## Change log

- 2026-10-27: Plan created.
- 2026-10-27: GitHub access switched from a `gh` token to SSH only; pull requests are opened on github.com. Step 0 done.
