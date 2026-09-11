# Problem 4: Ship It Twice

Two GitHub Actions pipelines — one for the backend HTTP API (EC2), one for the frontend SPA (S3) — in `.github/workflows/backend-ci-cd.yml` and `.github/workflows/frontend-ci-cd.yml`.

## Assumptions I made, and why

- **Each app has its own repo, with this workflow at the repo root**, and app source under `backend/` or `frontend/`.
- **The backend exposes a `/health` endpoint**, used to confirm the deploy actually worked.
- **Deployment reaches EC2 through AWS Systems Manager (SSM), not SSH.** This means no SSH private key needs to be stored as a GitHub secret, and no inbound SSH port needs to be open on the instance — the pipeline just needs an IAM role/user allowed to use SSM (plus, now, `ec2:DescribeInstances` so Ansible's dynamic inventory can enumerate the fleet).
- **The backend fleet is discovered by tag, not hardcoded by instance ID.** Ansible's `amazon.aws.aws_ec2` dynamic inventory (`backend/ansible/inventory/aws_ec2.yml`) targets every running instance tagged `Name=backend-api`. Scaling the fleet — adding or removing capacity — means tagging an instance, not touching the pipeline.

## How the pipelines work

Both follow the same shape: **test on every push/PR → deploy only on push to `main`.**

**Backend (`backend-ci-cd.yml`):**
1. `test` — checkout, install deps, run tests.
2. `deploy` (only on push to `main`) — zip the app, upload it to S3, then run `ansible-playbook backend/ansible/deploy.yml`. Ansible discovers the target fleet at runtime via dynamic inventory (every running instance tagged `Name=backend-api`) and, on each one over SSM (no SSH key, no open port): pulls the new zip, stops the service, replaces the code, restarts it.
3. A final `curl` from the pipeline against `/health` confirms the service came back up.

**Frontend (`frontend-ci-cd.yml`):**
1. `test` — checkout, install deps, run tests, build the production bundle.
2. `deploy` (only on push to `main`) — rebuild the bundle, `aws s3 sync` it to the bucket (`--delete` removes files from old builds), then invalidate the CloudFront cache so visitors get the new version immediately.

**Both workflows also have:**
- `paths:` filters, so a change to one app doesn't trigger the other's pipeline.
- A GitHub `environment: production` on the deploy job — this is where a real repo would attach required reviewers, so a human approves before anything touches production, without slowing down the test feedback loop on every PR.

## What I deliberately left out

- **Actual AWS setup** (IAM roles/policies, the S3 bucket, CloudFront distribution, EC2 instance/tags) — out of scope for "the pipeline," and the task doesn't require a real account. Credentials are read from GitHub secrets (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`) rather than OIDC, to keep the setup to "add two secrets" instead of also standing up an IAM OIDC trust relationship.
- **Automatic rollback.** The health check will fail the deploy job if the backend doesn't come back up, but it doesn't auto-revert to the previous release
- **Rolling/canary deploys.** Ansible pushes to every matching instance at once, same as the SSM version did — no batching (`serial`), no per-instance health gating before moving to the next one, no staging environment. Straightforward to add later (Ansible supports `serial` out of the box) but left out to keep the playbook to what the task actually needs: a tag-driven target list.
- **Security/dependency scanning, notifications.** Both are easy to add as extra steps later, but aren't core to whether the deploy pipeline itself works.
