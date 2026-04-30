# ecs-amplify-github-actions-cicd

GitHub Actions workflows that deliver a Django backend to Amazon ECS Fargate and a React frontend to AWS Amplify. A reference ECS task definition (`task.json`) accompanies the workflows to document the container shape being deployed. Each workflow runs end-to-end on GitHub-hosted runners using the AWS CLI, with no CodePipeline or external orchestrator.

A legacy SSM-based EC2 deploy path is preserved for reference, showing the prior deployment pattern that the ECS workflow replaced.

## Workflows

### `backend-p.yaml` — ECS Backend Deploy

1. Triggered on push to `ecs-cicd`.
2. Authenticates to AWS, logs into ECR, pulls the environment file from Secrets Manager and writes it to `.env` for the Docker build context.
3. `docker build --no-cache` then `docker push <ECR_URI>:latest`.
4. `aws ecs describe-task-definition` retrieves the live task definition. `jq` strips read-only fields (`taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, `registeredAt`, `registeredBy`) and rewrites the first container's image to the freshly pushed tag.
5. `aws ecs register-task-definition --cli-input-json` returns the new ARN. `aws ecs update-service --task-definition <new>` rolls the service.
6. `aws ecs wait services-stable` blocks until ECS reports the deployment complete.

### `frontend-p.yaml` — Amplify Frontend Deploy

1. Triggered on push to `ecs-cicd`.
2. Authenticates to AWS, fetches `.env` from Secrets Manager.
3. Node build: `npm ci` then `npm run build`.
4. `aws amplify start-job --app-id <amplify-app-id> --branch-name ecs-cicd --job-type RELEASE`.
5. Polls `aws amplify get-job` every 30 seconds until the job leaves `PENDING` or `RUNNING`. Exits non-zero if the terminal status is anything other than `SUCCEED`.

### `backend.yml` — Legacy SSM/EC2 Path (kept for reference)

Triggered on push to `dev` or `master`. Builds and pushes the image to ECR, then `aws ssm send-command` runs a bash payload on a target EC2 instance that performs `git reset --hard`, `docker-compose down`, `docker system prune -af`, `docker pull`, `docker-compose run backend python manage.py migrate`, `docker-compose up -d`. `aws ssm wait command-executed` and `get-command-invocation` surface stdout/stderr on failure.

## Reference Task Definition

`task.json` is a revision-48 export for the backend service (Fargate, 1024 CPU / 2048 MB). It contains four containers: `backend` (Gunicorn on port 8000), `celery-worker`, `celery-beat`, and a `flower` monitoring sidecar on port 5555. All four share an EFS-mounted `media-volume` and resolve database and JWT secrets from SSM Parameter Store. This is the exact document the ECS workflow mutates at deploy time.

## Stack

GitHub Actions · AWS ECS Fargate · ECR · Secrets Manager · Amplify · SSM Run Command (legacy path) · IAM · Docker · `jq` · AWS CLI v2

-> When consumed by an application repository, the YAML files belong in `.github/workflows/`.

## Prerequisites

GitHub repository secrets:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

AWS-side prerequisites:
- ECR repository for the backend image
- ECS cluster with a running service for the backend
- Task definition family matching `task.json`
- Secrets Manager entries for the environment files (`<app>/env/prod`, `<app>/env/backend`)
- Amplify app with a branch named `ecs-cicd`
- IAM user or role used by the workflow with permissions for ECR, ECS, Secrets Manager, Amplify, and SSM
- (Legacy path only) An EC2 instance registered with SSM Agent and a cloned repo at the expected path

## Triggering a Deploy

```bash
git checkout ecs-cicd
git commit --allow-empty -m "Trigger ECS deploy"
git push origin ecs-cicd
```

## Notes

- **Image tag is `:latest`.** The backend workflow pushes and references `:latest` on every deploy. This makes rollback by tag impossible and removes the audit trail of what was deployed when. Replace with `:${{ github.sha }}` or a semantic version tag and update the task definition patch step accordingly.
- **One branch, two environments.** Both pipelines trigger on `ecs-cicd`. A production-ready version should split branches (`prod` vs `staging`) and select the corresponding `SECRETS_ID`, `ECS_CLUSTER`, and Amplify branch.
- **Action versions are dated.** `actions/checkout@v3` and `aws-actions/configure-aws-credentials@v1` should move to v4. `aws-actions/amazon-ecr-login@v1` should move to v2.
- **Node 16** reached end of life in September 2023. Update `actions/setup-node` to Node 20 LTS.
- **Deploy strategy is rolling.** ECS default. For blue/green deploys with CloudWatch alarm rollback, switch the service to `DeploymentController = CODE_DEPLOY` and route updates through CodeDeploy.
- **AWS access keys.** Static IAM user keys work but are not recommended for production. Switch to OIDC federation via `aws-actions/configure-aws-credentials` with `role-to-assume` for short-lived credentials.
- **The `:latest` image and the rolling deploy together mean a failed task can leave the service partially upgraded.** `aws ecs wait services-stable` will fail noisily, but rollback requires manually rolling back the task definition revision in the AWS console.
