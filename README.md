# ECS + Amplify CI/CD Pipelines (GitHub Actions)

A set of GitHub Actions workflows that deliver the Project backend to **Amazon ECS** and the Project frontend to **AWS Amplify**, plus a legacy SSM-based EC2 deploy path kept for reference. Each workflow is independently triggered by a branch push and performs end-to-end build, image push, task-definition registration, and service roll-out — no CodePipeline, no Jenkins, just GitHub-hosted runners and the AWS CLI. A reference ECS task definition (`task.json`) accompanies the workflows to document the exact container shape being deployed.

## Highlights

- **Backend ECS workflow** (`backend-p.yaml`) — builds the Django image, pushes to ECR, clones the live task definition, swaps in the new image digest with `jq`, registers a new revision, updates the service, and waits on `aws ecs wait services-stable` for a clean cutover.
- **Frontend Amplify workflow** (`frontend-p.yaml`) — Node 16 build (`npm ci && npm run build`), then triggers an Amplify `start-job --job-type RELEASE` and polls until `SUCCEED`.
- **Legacy SSM/EC2 path** (`backend.yml` / `frontend.yml`) — preserved as a "before" reference: deploys to a single EC2 host via `aws ssm send-command` running docker-compose, git pull, and `manage.py migrate`. Good evidence of the migration from EC2+SSM to Fargate+ECS.
- Environment variables (`.env`) hydrated at build time from **AWS Secrets Manager** (`Project/env/prod`, `Project/env/backend`) — nothing baked into the image.
- Task definition fixture (`task.json`, revision 48) shows the real multi-container shape: `backend` (Gunicorn :8000), `celery-worker`, `celery-beat`, and a `flower` monitoring sidecar on :5555, all sharing an EFS media volume.

## Architecture

```
┌────────────┐   push to ecs-cicd   ┌──────────────────────┐
│  GitHub    │ ───────────────────► │  GitHub Actions      │
│  branch    │                      │  runner (ubuntu-22)  │
└────────────┘                      └──────────┬───────────┘
                                               │
          ┌────────────────────────────────────┼────────────────────────────────────┐
          │ backend-p.yaml                     │ frontend-p.yaml                    │
          ▼                                    ▼                                    │
   aws secretsmanager ──► .env         aws amplify start-job                       │
   docker build & push ──► ECR         (job-type RELEASE)                          │
   aws ecs describe-task-definition                                                │
          │  jq patch image                                                        │
          ▼                                                                        │
   aws ecs register-task-definition ──► new revision                               │
   aws ecs update-service ──► rolling deploy                                       │
   aws ecs wait services-stable ──► verified cutover                               │
```

## Tech stack

- **GitHub Actions** (ubuntu-latest runners)
- **AWS services:** ECS (Fargate), ECR, Secrets Manager, Amplify, SSM Run Command (legacy path), IAM
- **Actions used:** `actions/checkout@v3`, `aws-actions/configure-aws-credentials@v1`, `aws-actions/amazon-ecr-login@v1`, `actions/setup-node@v3`
- **Tooling in the runner:** Docker, `jq`, AWS CLI v2
- **Target application:** Project (Django + Gunicorn backend, React frontend on Amplify)

## Repository layout

```
ECS-PIPELINE-1/
├── README.md
├── .gitignore
├── backend-p.yaml    # primary backend pipeline — ECS deploy via task-def swap
├── backend.yml       # legacy backend pipeline — EC2 deploy via SSM + docker-compose
├── frontend-p.yaml   # primary frontend pipeline — AWS Amplify release job
├── frontend.yml      # legacy frontend copy (identical to backend.yml — kept for audit)
└── task.json         # reference ECS task definition (backend + celery-worker + celery-beat + flower)
```

## How it works

### `backend-p.yaml` — ECS backend deploy
1. Triggered on `push` to `ecs-cicd`.
2. Sets deployment vars for staging (`ECS_CLUSTER=Project-stage`, `ECS_SERVICE=Project-backend-stage`, `TASK_DEFINITION=Project-backend-stage`).
3. Authenticates to AWS (credentials sourced from repo secrets) and logs into ECR.
4. Pulls the environment file from Secrets Manager (`Project/env/prod`) and writes it to `.env` for the Docker build context.
5. `docker build --no-cache` → `docker push <ECR_URI>:latest`.
6. `aws ecs describe-task-definition` → `jq` strips the read-only fields (`taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, `registeredAt`, `registeredBy`) and rewrites the first container's image to the freshly pushed tag.
7. `aws ecs register-task-definition --cli-input-json` returns the new ARN; `aws ecs update-service --task-definition <new>` rolls the service.
8. `aws ecs wait services-stable` blocks until ECS reports the deployment complete.

### `frontend-p.yaml` — Amplify frontend deploy
1. Triggered on `push` to `ecs-cicd`.
2. Authenticates to AWS, fetches `.env` from Secrets Manager.
3. Node 16 build: `npm ci` → `npm run build`.
4. `aws amplify start-job --app-id <your-amplify-app-id> --branch-name ecs-cicd --job-type RELEASE`.
5. Polls `aws amplify get-job` every 30s until the job leaves `PENDING`/`RUNNING`; exits non-zero if the terminal status is anything other than `SUCCEED`.

### `backend.yml` / `frontend.yml` — legacy SSM path
Triggered on pushes to `dev`/`master`. Builds and pushes the image to ECR, then `aws ssm send-command` runs a bash payload on a target EC2 instance that: `git reset --hard`, `docker-compose down`, `docker system prune -af`, `docker pull`, `docker-compose run backend python manage.py migrate`, `docker-compose up -d`. `aws ssm wait command-executed` followed by `get-command-invocation` surfaces stdout / stderr on failure. Useful as a before-and-after artefact next to the ECS workflow.

### `task.json` — reference task definition
A revision-48 export for `Project-backend-stage` (family; Fargate; 1024 CPU / 2048 MB). Contains four containers — `backend`, `celery-worker`, `celery-beat`, `flower` — all wired to an EFS-mounted `media-volume` and to SSM-sourced DB/JWT secrets. It is the exact document the ECS workflow mutates at deploy time.

## Prerequisites

- GitHub repository with secrets configured:
  - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- AWS-side prerequisites:
  - ECR repository `Project/backend`
  - ECS cluster `Project-stage` with a running service `Project-backend-stage`
  - Task definition family `Project-backend-stage` (see `task.json`)
  - Secrets Manager entries `Project/env/prod` and `Project/env/backend`
  - Amplify app (id supplied via `AMPLIFY_APP_ID` env) with a branch named `ecs-cicd`
  - IAM user/role used by the workflow with permissions for ECR, ECS, Secrets Manager, Amplify, SSM
- (Legacy path only) EC2 instance registered with SSM Agent and a cloned repo under `/var/www/html/Project-be`

## Deployment

No manual Terraform here — the workflows are the deployment mechanism. To dry-run locally you can lint the YAML with `actionlint` or run individual AWS CLI commands with a scoped role.

To trigger:

```bash
git checkout ecs-cicd
git commit --allow-empty -m "Trigger ECS deploy"
git push origin ecs-cicd
```

## Teardown

Disable or delete the workflows in `.github/workflows/` (the YAML files shipped here would be moved into that directory in the consuming application repo). ECS and Amplify resources themselves are managed outside this project.

## Notes

- The two legacy files `backend.yml` and `frontend.yml` are identical — `frontend.yml` is a pre-migration copy kept for history; the real frontend pipeline is `frontend-p.yaml`.
- All AWS access keys that previously lived inline in these files have been replaced with `${{ secrets.AWS_ACCESS_KEY_ID }}` / `${{ secrets.AWS_SECRET_ACCESS_KEY }}`. The original EC2 instance ID has been replaced with `<your-ec2-instance-id>` and the ECR URI's account prefix with `<your-account-id>`.
- Deploy strategy is rolling (ECS default); no blue/green is configured. For a future iteration, wire `aws ecs update-service` through a CodeDeploy `DeploymentController = CODE_DEPLOY` for proper blue/green with CloudWatch alarm rollback.
- Both environments currently share one branch (`ecs-cicd`) and one secrets payload; a production-ready version would branch `prod` vs `ecs-cicd` and switch `SECRETS_ID` / `ECS_CLUSTER` accordingly.
