# todo-app-foundation

Shared, slow-changing AWS resources for the **Todo App**. Deployed with
CloudFormation Git sync. This is the base layer every other repo builds on, and
everything they need is published as a cross-stack Export.

Region: `eu-north-1`.

## How the three repos fit together

| Repo                       | Purpose                                                                                  |
| -------------------------- | ---------------------------------------------------------------------------------------- |
| `todo-app-foundation`      | **This repo.** Base resources needed before anything else can be provisioned.           |
| infrastructure repo        | CloudFormation templates that provision the runtime (VPC, ECS, ALB, CloudFront, RDS, RDS Proxy, ElastiCache, pipeline). Imports this stack's exports. |
| `todo-app-codes`           | Application code. Its GitHub Actions workflow assumes the CI role created here to push images and artifacts. |

## What this stack creates

- **ECR repository** for the application image.
- **Static-assets bucket** (private; served through CloudFront by the infra stack).
- **Pipeline artifact bucket** (versioned, for CodePipeline).
- **GitHub Actions CI role** (OIDC, reusing the account's existing GitHub OIDC provider).
- **Data-layer foundations** — the Todo App stores its data in **Amazon RDS**,
  accessed only through **RDS Proxy**, and uses **Amazon ElastiCache for Redis**
  to cache reads. This stack creates the long-lived pieces those need:
  - `DbMasterSecret` — Secrets Manager secret with the generated RDS master
    username/password (RDS Proxy authenticates with it).
  - `RdsProxyRole` — IAM role RDS Proxy assumes to read that secret.
  - `RedisAuthTokenSecret` — generated AUTH token for the Redis cluster.

  The RDS instance, the RDS Proxy and the ElastiCache Redis cluster themselves
  are created in the infrastructure stack, because they need its VPC, subnets
  and security groups. Keeping the credentials here means they survive any
  rebuild of the infrastructure stack.

## Repo layout

Everything sits at the repository root — no subfolders, so the paths Git sync
resolves are exactly the file names.

```
todo-app-foundation/
├── bootstrap.yaml                  # Run ONCE with the CLI (see step 1)
├── foundation.yaml                 # Deployed by Git sync
├── foundation.deployment.yaml      # Git sync deployment file
└── README.md
```

## Why there is a bootstrap step

Git sync can only deploy a stack once a GitHub connection and a deploy role
already exist. A stack cannot create the very role and connection that Git sync
needs to deploy that stack. So those two resources, and only those two, are
created once by `bootstrap.yaml`. The foundation reuses the account-level
GitHub OIDC provider at `https://token.actions.githubusercontent.com`; it must
already exist in the target account. Every other role in the project
(task role, execution role, pipeline, CodeDeploy, EventBridge, CI, RDS Proxy) is
created automatically inside the templates.

## Step 1: deploy the bootstrap once

```bash
aws cloudformation deploy \
  --stack-name todo-app-bootstrap \
  --template-file bootstrap.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-north-1
```

Then authorize the connection: open **Developer Tools > Settings > Connections**
in the console, select `todo-app-github`, and click **Update pending
connection** to finish the GitHub handshake. It stays `PENDING` until you do
this, and Git sync will not work before it is `AVAILABLE`.

Note the two stack outputs:

- `CfnGitSyncDeployRoleArn` — select this role when configuring Git sync.
- `GitHubConnectionArn` — the connection Git sync links to.

## Step 2: check the deployment file

In `foundation.deployment.yaml`, confirm `GitHubOrg` is your GitHub org or
username and `AppRepoName` is the application repo name, then commit and push.

## Step 3: create the foundation stack from Git

In the CloudFormation console choose **Create stack > With new resources**, then
**Sync from Git**. Use:

- Deployment file: `foundation.deployment.yaml`
- Connection: `todo-app-github`
- Repository: `todo-app-foundation`, branch `main`
- IAM role: the existing `todo-app-cfn-gitsync-role`

CloudFormation opens a pull request in the repo to complete the link. Merge it,
and the stack deploys. From then on, any push that changes the template or the
deployment file re-syncs the stack automatically.

## What this stack exports

| Export name                         | Value                                                    |
| ----------------------------------- | -------------------------------------------------------- |
| `todo-app-ecr-uri`                  | ECR repository URI                                       |
| `todo-app-ecr-arn`                  | ECR repository ARN                                       |
| `todo-app-ecr-name`                 | ECR repository name                                      |
| `todo-app-assets-bucket-name`       | Static-assets bucket name                                |
| `todo-app-assets-bucket-arn`        | Static-assets bucket ARN                                 |
| `todo-app-assets-bucket-domain`     | Static-assets bucket regional domain (CloudFront origin) |
| `todo-app-artifact-bucket-name`     | Artifact bucket name                                     |
| `todo-app-artifact-bucket-arn`      | Artifact bucket ARN                                      |
| `todo-app-ci-role-arn`              | CI role ARN for GitHub Actions                           |
| `todo-app-db-secret-arn`            | RDS master credentials secret (RDS + RDS Proxy auth)     |
| `todo-app-db-name`                  | Initial database name                                    |
| `todo-app-db-username`              | RDS master username                                      |
| `todo-app-rds-proxy-role-arn`       | Role RDS Proxy assumes to read the DB secret             |
| `todo-app-redis-auth-secret-arn`    | ElastiCache Redis AUTH token secret                      |

## Note on the deploy role

`todo-app-cfn-gitsync-role` uses `AdministratorAccess` so the lab deploys
without you chasing missing IAM permissions template by template. For a real
environment, replace that managed policy with one scoped to the services the
stacks actually create (ec2, ecs, ecr, elasticloadbalancing, rds (including
RDS Proxy), elasticache, secretsmanager, s3, cloudfront, codedeploy,
codepipeline, events, logs, application-autoscaling, iam create/pass role), and
keep these Git sync actions in it:

- `cloudformation:CreateChangeSet`, `DeleteChangeSet`, `DescribeChangeSet`,
  `ExecuteChangeSet`, `DescribeStackEvents`, `DescribeStacks`, `GetTemplate`,
  `ListChangeSets`, `ListStacks`, `ValidateTemplate`
- `events:PutRule`, `events:PutTargets` (condition
  `events:ManagedBy = cloudformation.sync.codeconnections.amazonaws.com`),
  `events:DescribeRule`
