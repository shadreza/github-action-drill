# HearingSystem — lr / lr-backup deploy toggle

Setup notes for adding a manual branch toggle to the prod repo's
`.github/workflows/aws.yml`.

Written 2026-07-28. Action SHAs below were the current releases on that date.

## What this gives you

`lr` (new work) and `lr-backup` (legacy) deploy to the **same** EC2 box and the
same Apache docroot (`/var/www/hearing`), so only one of them can occupy the
server at a time. A repository variable named `ACTIVE_BRANCH` decides which one
auto-deploys on push. The other branch still accumulates commits — it just
doesn't deploy.

| Situation | Result |
| --- | --- |
| `ACTIVE_BRANCH=lr`, push to `lr` | deploys |
| `ACTIVE_BRANCH=lr`, push to `lr-backup` | run happens, deploy job **skipped** |
| `ACTIVE_BRANCH=lr-backup`, push to `lr` | run happens, deploy job **skipped** |
| `ACTIVE_BRANCH=lr-backup`, push to `lr-backup` | deploys |
| Manual run (any branch picked in the UI) | deploys that branch, ignores the toggle |
| `ACTIVE_BRANCH` unset or typo'd | gate fails, nothing deploys |

The toggle is deliberately in repo settings rather than in the repo, so
changing it needs admin access rather than push access.

## Why the toggle can't live in the trigger

GitHub evaluates `on: push: branches:` before any workflow context exists, so
`vars` is not available there. Both branches must therefore be listed as
triggers, and the decision happens in a `gate` job. Consequence: the inactive
branch produces a **skipped** run rather than no run at all. That's expected,
not a misconfiguration.

## Prerequisites

Do all three before the first run.

1. **Set the variable.** Settings → Secrets and variables → Actions →
   Variables → New repository variable. Name `ACTIVE_BRANCH`, value `lr`.
   Without it the gate fails closed and nothing deploys.
2. **Create `lr-backup`, and put this workflow file on it.** Push events run
   the workflow from the pushed commit, so an `lr-backup` that still has the
   old ungated file would deploy unconditionally and clobber `lr`.
3. **Put the file on the default branch too**, or the "Run workflow" dropdown
   never appears in the Actions tab.

Existing secrets `EIP` and `SSH_PRIVATE_KEY` are unchanged and still required.

## The file

Full contents of `.github/workflows/aws.yml`:

```yaml
name: HearingSystem CI/CD Workflow

on:
  # NOTE: vars cannot be used in these filters, so both branches always
  # trigger a run. Which one actually deploys is decided by the `gate` job.
  push:
    branches:
      - lr
      - lr-backup
  # Manual run deploys whichever branch you pick in the ref dropdown,
  # ignoring ACTIVE_BRANCH. Used for short client look-backs at lr-backup.
  workflow_dispatch:

# One server, one docroot: never let two deploys interleave.
concurrency:
  group: hearing-deploy
  cancel-in-progress: false

jobs:
  gate:
    runs-on: ubuntu-latest
    outputs:
      deploy: ${{ steps.pick.outputs.deploy }}
    steps:
      - id: pick
        env:
          ACTIVE_BRANCH: ${{ vars.ACTIVE_BRANCH }}
          EVENT: ${{ github.event_name }}
          REF: ${{ github.ref_name }}
        run: |
          case "$REF" in
            lr|lr-backup) ;;
            *)
              echo "::error::$REF is not a deployable branch (expected lr or lr-backup)."
              exit 1
              ;;
          esac

          if [ "$EVENT" = "workflow_dispatch" ]; then
            echo "deploy=true" >> "$GITHUB_OUTPUT"
            echo "::notice::Manual run — deploying $REF (ACTIVE_BRANCH is ${ACTIVE_BRANCH:-unset})."
            exit 0
          fi

          if [ -z "$ACTIVE_BRANCH" ]; then
            echo "::error::Repository variable ACTIVE_BRANCH is not set (expected lr or lr-backup)."
            exit 1
          fi

          if [ "$REF" = "$ACTIVE_BRANCH" ]; then
            echo "deploy=true" >> "$GITHUB_OUTPUT"
            echo "::notice::$REF owns the server — deploying."
          else
            echo "deploy=false" >> "$GITHUB_OUTPUT"
            echo "::notice::Skipping: pushed to $REF but ACTIVE_BRANCH is $ACTIVE_BRANCH."
          fi

  deploy:
    needs: gate
    if: needs.gate.outputs.deploy == 'true'
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Check GitHub Actions IP
        run: curl https://api.ipify.org

      - name: Package app
        run: |
          mkdir git
          rsync -av --exclude='.git' --exclude='git' ./ git/
          tar -czvf app.tar.gz git/

      - name: Upload app.tar.gz to server
        uses: appleboy/scp-action@ff85246acaad7bdce478db94a363cd2bf7c90345 # v1.0.0
        with:
          host: ${{ secrets.EIP }}
          username: ec2-user
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: app.tar.gz
          target: /home/ec2-user/

      - name: Deploy HearingSystem
        uses: appleboy/ssh-action@0ff4204d59e8e51228ff73bce53f80d53301dee2 # v1.2.5
        with:
          host: ${{ secrets.EIP }}
          username: ec2-user
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Abort on the first failing command instead of running to the end
            # and exiting 0 from the final echo. Upstream removed the
            # script_stop input; this is the documented replacement.
            set -euo pipefail

            ROOT_DIR=/home/ec2-user
            DEPLOY_PATH=${ROOT_DIR}/source
            APACHE_PATH=/var/www/hearing

            mkdir -p ${DEPLOY_PATH}

            cd ${ROOT_DIR}
            mv app.tar.gz ${DEPLOY_PATH}/

            cd ${DEPLOY_PATH}
            sudo rm -rf git
            sudo tar xzvf app.tar.gz
            sudo chown -R ec2-user:ec2-user git

            sudo systemctl stop httpd

            sudo rm -rf ${APACHE_PATH}/src
            sudo mv ${DEPLOY_PATH}/git/webroot/css/* ${APACHE_PATH}/webroot/css/
            sudo mv ${DEPLOY_PATH}/git/webroot/js/* ${APACHE_PATH}/webroot/js/
            sudo mv ${DEPLOY_PATH}/git/webroot/img/* ${APACHE_PATH}/webroot/img/

            sudo mv ${DEPLOY_PATH}/git/src ${APACHE_PATH}/
            sudo cp ${DEPLOY_PATH}/git/composer.json ${APACHE_PATH}/composer.json
            sudo cp ${DEPLOY_PATH}/git/index.php ${APACHE_PATH}/index.php

            sudo cp ${DEPLOY_PATH}/git/config/constant.php ${APACHE_PATH}/config/constant.php
            sudo cp ${DEPLOY_PATH}/git/config/routes.php ${APACHE_PATH}/config/routes.php

            sudo rm -rf ${APACHE_PATH}/tmp/cache/models/*
            sudo systemctl start httpd

            echo "DEPLOY DONE"
```

## What changed vs. the original workflow

The deploy script itself is byte-for-byte identical to what the original
author wrote. Everything else:

- `lr-backup` added to `push.branches` — the gate can't skip a run that never
  starts.
- `workflow_dispatch:` added — there was previously no way to deploy manually.
  No inputs needed: GitHub's dispatch dialog already has a branch selector,
  and bare `actions/checkout@v4` checks out the triggering ref.
- `concurrency: hearing-deploy` with `cancel-in-progress: false` — the script
  does `stop httpd` → shuffle files → `start httpd`. Two overlapping runs
  means one deploy stopping Apache while the other is mid-`mv`, leaving a
  corrupted docroot. Queueing is the correct behaviour here.
- `gate` job plus `needs`/`if` on `deploy`. Also rejects a dispatch from any
  branch other than `lr`/`lr-backup`, so nobody can hand-deploy `master` onto
  the hearing box.
- Both `appleboy` actions pinned by commit SHA instead of `@master`. They run
  with `SSH_PRIVATE_KEY` and `sudo` on the box, so "whatever that branch
  points at today" is not a good version constraint.
- `set -euo pipefail` at the top of the ssh script. The `script_stop` input
  that used to do this was **removed upstream**; passing it now is a silent
  no-op. Without this line, a failure partway through exits 0 from the
  trailing `echo` and the run reports green over a half-updated docroot.

`set -u` and `pipefail` are safe here specifically: the script defines
`ROOT_DIR`, `DEPLOY_PATH` and `APACHE_PATH` before use and contains no
pipelines. The only behavioural change is `-e`.

## Rollout order

1. Set `ACTIVE_BRANCH=lr`.
2. Merge the new workflow to `lr`. The first run is then the ordinary path —
   `lr` deploying exactly as it does today — so you're only exercising the
   gate's happy case against a live server.
3. Push a trivial commit to `lr-backup` and confirm the deploy job shows
   **skipped**, with the gate log reading
   `Skipping: pushed to lr-backup but ACTIVE_BRANCH is lr`.

Watch the first run. Two things change independently of the toggle: the
`ssh-action` version moves from drifted `@master` to v1.2.5, and `set -e` goes
live. If any command in that script has been failing quietly, this is the run
where it turns red — **and Apache will be stopped when it does**. Have someone
able to `systemctl start httpd` by hand while you watch.

## Day-to-day operation

Two controls answering different questions:

- **`ACTIVE_BRANCH`** — "which branch owns the server when someone pushes."
  A durable setting.
- **Manual dispatch** — "put this branch on the server right now." One-shot.

For a quick client look-back at legacy, **don't touch the toggle** — just run
the workflow manually against `lr-backup`. The server shows legacy, and
`ACTIVE_BRANCH` stays on `lr`. Flip the toggle only when the look-back will
last long enough that an incidental push to `lr` would otherwise clobber what
the client is looking at.

Flipping the toggle by itself deploys nothing. It only changes what the *next*
push does. Flip, then dispatch, if you want it live immediately.

## Limits of the toggle

Worth being explicit, because it looks stronger than it is:

1. **Manual dispatch bypasses it entirely.** Anyone with write access can
   deploy `lr-backup` while the toggle says `lr`. That's the intended hotfix
   escape hatch, but it makes the toggle a default, not an enforcement
   boundary.
2. **The gate lives in a file on the branch it gates.** Someone with push
   access to `lr` can delete the `if:` on `lr` and deploy freely. If the
   toggle needs to be authoritative, add branch protection with required
   review on both branches plus `CODEOWNERS` covering `.github/`.
3. **A skipped job counts as passing for required status checks.** Never make
   `deploy` a required check — the inactive branch would report green without
   deploying. Gate on `gate` instead, or don't require it.

## Known hazards when swapping lr ↔ lr-backup

These are properties of the existing deploy script, not of the toggle, but the
toggle makes them reachable because the two codebases have diverged.

- **Stale assets accumulate.** `src` is replaced cleanly (`rm -rf` then `mv`),
  but css/js/img use `mv .../* dest/`, which **merges**. A file present in
  `lr` but not `lr-backup` survives the swap, so a client "checking previous
  work" can see legacy PHP loading a newer JS bundle. Fixing this means
  `rsync --delete` instead of `mv`.
- **`composer install` never runs.** `composer.json` is copied, but `vendor/`
  on the box stays whatever the last install produced. If the two branches
  need different dependency versions, legacy code runs against newer libs.
- **Persistent state doesn't roll back.** If forward migrations have run for
  `lr`, `lr-backup` may not boot against that schema — and the failure will
  look like a toggle problem when it isn't. If the gap includes schema
  changes, a separate long-lived environment for `lr-backup` is more honest
  than time-sharing one box, and it removes the need for the toggle.

## Open item

With `set -e` active, a mid-script failure leaves Apache stopped — loud, but
down. The fix is `trap 'sudo systemctl start httpd' ERR` near the top of the
script, so a failure still goes red but the site comes back. That's a change
to the original author's script, so it's their call. Worth raising, since
`set -e` makes it more relevant than it was before.

Also unpinned: `actions/checkout@v4`. First-party and low risk, but the same
argument applies if you want everything on SHAs.
