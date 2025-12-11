# Dev Checkpoints Cron (SSH)

This demo models a cloud cutover scenario: a primary dev workspace is checkpointed every 2 minutes to a Git SSH remote, and a secondary restore workspace (`dev-restore`) simulates bringing that state up on another cloud with minimal data loss.

## What it does
- **Dev workspace (primary)**: Checkout of `octocat/Hello-World` (small public repo).
- **Checkpoint agent**: Mirrors the dev workspace (including `.git`) to `/home/owner/dev-mirror` via `cs mutagen`, then every 2 minutes:
  - Fetches workspace commits (including unpushed work).
  - Creates a synthetic checkpoint commit if the worktree is dirty.
  - Pushes:
    - Workspace tip to `refs/heads/<branch>`.
    - Checkpoint tip to `refs/checkpoints/<sandbox>/<branch>` and timestamped refs (retains latest 90).
- **SSH remote**: Served by the `ssh-remote` workload on port `2222`, storing the bare repo at `/home/owner/checkpoints/remote.git`.
- **Restore helper**: `~/restore_checkpoint.sh` clones from the SSH remote and checks out the latest checkpoint ref as `restored`.
- **Dev-restore workspace (cutover simulation)**: A clean workspace with the same checkout and restore script, used to validate that the checkpoint can be recovered on a “secondary cloud” without touching the primary.

## Setup
- Keep the sandbox name short (<=20 chars). Example:
  ```bash
  cs sandbox create dev-ckpt-ssh --from def:/home/owner/dev-checkpoints-cron/dev-checkpt-ssh-demo.yaml --wait --if-exists skip
  ```
- Wait until `dev`, `dev-restore`, `checkpoint-agent`, and `ssh-remote` show **Ready**.

## Test the checkpoint flow
1) In **dev**, make an uncommitted change (e.g., append to `TEST_CHECKPOINT.md`).
2) Wait ~2 minutes. In **checkpoint-agent**, inspect the log:
   ```bash
   tail -n 80 /home/owner/checkpoints/logs/checkpoint.log
   ```
   You should see the cron run and pushes to `ssh://owner@ssh-remote:2222/home/owner/checkpoints/remote.git`.
3) From **ssh-remote**, list refs:
   ```bash
   git --git-dir=/home/owner/checkpoints/remote.git for-each-ref --format='%(refname)'
   ```
   Expect `refs/heads/main` and `refs/checkpoints/<sandbox>/main`.

## Simulate cutover / restore
1) In **dev**, make changes; wait for a checkpoint run.
2) In **dev-restore**, run:
   ```bash
   bash ~/restore_checkpoint.sh
   ```
   - Clones from the SSH remote.
   - Fetches `refs/checkpoints/<sandbox>/main` and checks out `restored`.
3) If Git warns about dubious ownership:
   ```bash
   HOME=/home/owner git config --global --add safe.directory /home/owner/app
   ```

## How it’s wired
- Cron: `*/2 * * * *` in `checkpoint-agent` via `/home/owner/checkpoint_cron.sh`.
- `bootstrap.sh`: ensures mutagen session, shadow repo, and remote.
- `checkpoint_agent.py`: performs fetch, optional checkpoint commit, and pushes to the SSH remote.

## Credentials and host
- `ssh-remote` provides the SSH daemon and bare repo for the demo. In production, host the Git SSH remote on the backup cloud or a neutral location (not the primary).
- Store SSH keys/known_hosts in Crafting secrets (not inline env vars). The demo YAML uses `${secret:...}` placeholders that must be hydrated before create:
  - `${secret:CHECKPOINT_REMOTE_KEY}` (private key)
  - `${secret:CHECKPOINT_REMOTE_PUB}` (public key)
  - `${secret:CHECKPOINT_KNOWN_HOSTS}` (known_hosts entry pinning the Git SSH host)
  - `${secret:SSH_REMOTE_HOST_KEY}` / `${secret:SSH_REMOTE_HOST_KEY_PUB}` (server host keypair for the SSH remote)
  - `${secret:SSH_REMOTE_AUTH_KEYS}` (authorized_keys for the SSH remote)
  Set `CHECKPOINT_REMOTE` to your Git SSH URL and ensure the agent can push there once these secrets are injected.

### Preparing the secrets (command-by-command)
Generate these once (on a safe machine), then copy the file contents into Crafting secrets with the corresponding names:

1) Client keypair (used by dev/dev-restore/checkpoint-agent to push/pull):
```bash
ssh-keygen -t ed25519 -f checkpoint_remote_ed25519 -N ""
# secrets:
#   CHECKPOINT_REMOTE_KEY  <- cat checkpoint_remote_ed25519
#   CHECKPOINT_REMOTE_PUB  <- cat checkpoint_remote_ed25519.pub
```

2) Server host keypair (used by the SSH remote you control; for the demo ssh-remote workload):
```bash
ssh-keygen -t ed25519 -f ssh_host_checkpoint_ed25519 -N ""
# secrets:
#   SSH_REMOTE_HOST_KEY     <- cat ssh_host_checkpoint_ed25519
#   SSH_REMOTE_HOST_KEY_PUB <- cat ssh_host_checkpoint_ed25519.pub
```

3) Authorized keys for the SSH remote (allow the client key above):
```bash
# secrets:
#   SSH_REMOTE_AUTH_KEYS <- cat checkpoint_remote_ed25519.pub
```

4) Pin the SSH host key in known_hosts (replace HOST and PORT with your Git SSH host/port; for the demo ssh-remote:2222):
```bash
ssh-keyscan -p 2222 ssh-remote.example.com > checkpoint_known_hosts
# secrets:
#   CHECKPOINT_KNOWN_HOSTS <- cat checkpoint_known_hosts
```

5) Set the remote URL to match your host:
```bash
export CHECKPOINT_REMOTE=ssh://owner@ssh-remote.example.com:2222/home/owner/checkpoints/remote.git
```
