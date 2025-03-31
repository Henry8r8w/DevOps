
## Basic Structure

Workflow files use YAML syntax and must have a `.yml` or `.yaml` file extension. They must be stored in the `.github/workflows` directory of the repository

```yaml
name: My Workflow                                 # The name displayed in GitHub UI

run-name: Deploy to ${{ inputs.deploy_target }}   # Dynamic name for workflow runs
          by @${{ github.actor }}                 # Uses the username who triggered it

on:                                               # Events that trigger the workflow
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:                              # Manual trigger with custom inputs
    inputs:
      deploy_target:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'

jobs:                                             # Collection of jobs to run
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build project
        run: npm ci && npm run build
```

**Notes:**
- `name` is optional but recommended for easy identification in the GitHub UI
- `run-name` supports dynamic values and is shown in the workflow run list
- Each workflow file should focus on a specific pipeline or purpose
- YAML is sensitive to indentation
## Workflow Triggers
The `on` section defines what events trigger the workflow to run. GitHub Actions supports event ==such as ==

### Single Event Trigger

```yaml
on: push
```

### Multiple Event Triggers

```yaml
on: [push, pull_request, workflow_dispatch]
```

### Detailed Event Configuration

```yaml
on:
  push:
    branches:
      - main
      - 'releases/**'             # Supports glob patterns
    tags:
      - v1.*                      # Match all v1.x tags
    paths:
      - '**.js'                   # Only run when JS files change
    paths-ignore:
      - 'docs/**'                 # Don't run when only docs change
  pull_request:
    types:                        # Specific PR events
      - opened
      - synchronize
      - labeled
  schedule:
    - cron: '0 0 * * *'           # Runs with reoccuring time schedule
```

### Manual Trigger with Inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice                # Dropdown menu in UI
        options:
          - dev
          - staging
          - production
      debug:
        description: 'Enable debug mode'
        required: false
        type: boolean               # Checkbox in UI
      version:
        description: 'Version number'
        required: true
        type: string                # Text input in UI
```
### Repository  Events 

```yaml
on:
  # Issue events
  issues:
    types: [opened, edited, closed, reopened, assigned, labeled]
  issue_comment:
    types: [created, edited, deleted]
  
  # Pull request events
  pull_request:
    types: [opened, closed, synchronize, reopened, assigned, labeled]
  pull_request_review:
    types: [submitted, edited, dismissed]
  pull_request_review_comment:
    types: [created, edited, deleted]
  
  # Repository management events
  fork:
  watch:
    types: [started]  # When someone stars the repository
  
  # Release events
  release:
    types: [published, created, edited, deleted, prereleased, released]
  
  # Status events
  status:  # Commit status changes
  
  # Deployment events
  deployment:
  deployment_status:
  
  # GitHub Pages events
  page_build:
  
  # Custom events
  repository_dispatch:  # Custom webhook event
    types: [deploy-command, my-custom-event]
```

### Activity Types for Events

Many events support specific activity types that give you finer control:

```yaml
on:
  label:
    types:
      - created                     # Only when labels are created
  
  issue_comment:
    types:
      - created
      - edited
```

**Notes:**

- Use `branches-ignore` or `paths-ignore` when you want to exclude patterns
- Glob patterns like `**` (any directory), `*` (any string), and `?` (any single character) are available to use
- For pull requests from forks, some events might have limited permissions for security reasons
- `schedule` uses standard cron syntax in UTC timezone
- When filtering on branches, the pattern is matched against the Git ref name (e.g., `refs/heads/main`)

## Workflow Jobs

Jobs are the core components of a workflow. They define what steps to run and on which environment.
### Basic Job Structure

```yaml
jobs:
  build:                           # Job ID (must be unique within workflow)
    name: Build Application        # Job name displayed in UI (optional)
    runs-on: ubuntu-latest         # Runner environment
    timeout-minutes: 60            # Maximum minutes before GitHub cancels
    steps:                         # Sequence of operations
      - uses: actions/checkout@v4  # Use an pre-existing action
      
      - name: Setup Node.js        # Step name for UI
        uses: actions/setup-node@v4
        with:                      # Parameters for the action
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci                # Run a command
        
      - name: Build
        run: npm run build
```

### Job Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm build

  test:
    needs: build                   # This job runs after build completes
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: [build, test]           # deploy job depends on the success of build and test jobs sucess
    runs-on: ubuntu-latest
    steps:
      - run: npm deploy
```

### Conditional Jobs

```yaml
jobs:
  production-deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy-to-production.sh
  
  staging-deploy:
    if: startsWith(github.ref, 'refs/heads/feature/')
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy-to-staging.sh
```
refs/heads/main is ...?
startsWith(), where does this come from 
### Matrix Strategy

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [14.x, 16.x, 18.x]
        include:                                     # Add specific configurations
          - os: ubuntu-latest
            node-version: 18.x
            experimental: true
        exclude:                                     # Remove specific configurations
          - os: windows-latest
            node-version: 14.x
      fail-fast: false                               # Continue with other matrix jobs if one fails
      max-parallel: 3                                # Limit concurrent jobs
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

### Job Outputs (test this)

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:                            # Define outputs that other jobs can use
      output1: ${{ steps.step1.outputs.test }}
      output2: ${{ steps.step2.outputs.test }}
    steps:
      - id: step1
        run: echo "test=hello" >> $GITHUB_OUTPUT
      - id: step2
        run: echo "test=world" >> $GITHUB_OUTPUT
  
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
      - run: echo ${{ needs.job1.outputs.output1 }} ${{ needs.job1.outputs.output2 }}
```

### Runner Selection (not sure how this works)

```yaml
jobs:
  deploy:
    # Using a GitHub-hosted runner
    runs-on: ubuntu-latest
    
  custom-job:
    # Using a self-hosted runner with specific labels
    runs-on: [self-hosted, linux, x64, gpu]
    
  dynamic-runner:
    # Using a variable to determine the runner
    runs-on: ${{ inputs.runner-type }}
```

**Notes:**
- Job IDs must be unique within a workflow and start with a letter or `-`
- Jobs run in parallel by default; use `needs` to create sequential jobs
- The `matrix` strategy creates multiple job runs with different configurations
- Self-hosted runners use labels to target specific machines
- You can reference outputs from other jobs using the `needs` context
- Each step in a job runs in the same runner environment and shares the filesystem

## Permissions

Modify the default permissions granted to the `GITHUB_TOKEN` to implement the principle of least privilege.

### Workflow-level Permissions

```yaml
name: CI Workflow

# Set permissions for all jobs
permissions:
  contents: read         # Access to repository contents
  pull-requests: write   # Ability to modify pull requests
  issues: write          # Ability to modify issues

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

### Job-level Permissions

```yaml
jobs:
  stale:
    runs-on: ubuntu-latest
    # Set permissions for this specific job
    permissions:
      issues: write
      pull-requests: write
    steps:
      - uses: actions/stale@v9
        
  audit:
    runs-on: ubuntu-latest
    # Different permissions for another job
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        run: ./security-scan.sh
```

### Predefined Permission Sets

```yaml
permissions: read-all     # Grant read permissions to all tokens
# OR
permissions: write-all    # Grant write permissions to all tokens
# OR
permissions: {}           # No permissions 
```

### Available Permission Scopes

- `actions`: Manage GitHub Actions
- `checks`: Manage check runs and suites
- `contents`: Access repository content
- `deployments`: Manage deployments
- `id-token`: Request OIDC tokens
- `issues`: Manage issues
- `discussions`: Access to GitHub Discussions
- `packages`: Access to GitHub Packages
- `pages`: Manage GitHub Pages
- `pull-requests`: Access pull requests
- `repository-projects`: Access repository projects
- `security-events`: Access code scanning and Dependabot alerts
- `statuses`: Access commit 
- `contents`

**Notes:**

- By default, workflows have write permissions to most scopes
- It's a best practice to restrict permissions to only what's needed
- If you specify any permission, all unspecified permissions are set to `none`
- For pull requests from forks, the `GITHUB_TOKEN` has read permissions only by default

## Environment Variables

### Workflow-level Variables

```yaml
name: CI

env:
  SERVER: production
  LOG_LEVEL: info

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Echo environment variables
        run: echo "Server is $SERVER and log level is $LOG_LEVEL"
```

### Job-level Variables

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      DATABASE_URL: mysql://user:password@localhost:3306/test
    steps:
      - name: Run tests
        run: npm test
```

### Step-level Variables

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Set up credentials
        env:
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: echo "Using API token $API_TOKEN"
```

### Setting and Using Output Variables

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Set output variable
        id: set-version
        run: echo "version=$(cat version.txt)" >> $GITHUB_OUTPUT
      
      - name: Use output variable
        run: echo "Version is ${{ steps.set-version.outputs.version }}"
```

**Notes:**

- Environment variables are case-sensitive
- Variables defined at a more specific level override broader ones
- The `GITHUB_ENV` file can be used to set variables that persist between steps
- Default environment variables like `GITHUB_WORKSPACE` and `GITHUB_SHA` are always available
- For security, never hardcode secrets in workflow files; use the secrets context

## Concurrency

The concurrency feature helps you manage how multiple instances of your workflows or jobs run at the same time. This is  useful to prevent race conditions or ensure only one deployment happens at a time.

### What Concurrency Does
Think of a concurrency group as a queue with a single slot:
1. When a workflow run or job starts, it checks if another run with the same concurrency group is already in progress
2. If nothing is running, it proceeds immediately
3. If something is already running, one of two things happens:
    - By default: The new run waits in a "pending" state until the running one finishes
    - With `cancel-in-progress: true`: The currently running workflow is cancelled and the new one starts immediately

### Basic Concurrency Example

```yaml
name: Deployment

# This ensures only one deployment runs at a time
concurrency:
  group: production-deploy
  cancel-in-progress: true  # Cancels any in-progress deployment when a new one is triggered

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh
```

In this example:

- If someone triggers this workflow while a previous run is still deploying, the previous deployment will be cancelled
- This prevents multiple deployments from running simultaneously and potentially causing conflicts

### Branch-Specific Concurrency

```yaml
name: CI

# Creates a separate concurrency group for each branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

In this example:

- Each branch gets its own concurrency group (`workflow-name-refs/heads/branch-name`)
- If multiple commits are pushed to the same branch quickly, only the latest one will run tests
- Different branches can still run tests simultaneously (they have different group names)

### Job-Level Concurrency

You can also set concurrency for individual jobs instead of the entire workflow:

```yaml
jobs:
  # This job can run multiple times simultaneously
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm build
  
  # This job ensures only one deployment runs at a time
  deploy:
    runs-on: ubuntu-latest
    concurrency:
      group: production-deploy
      cancel-in-progress: true
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh
```

In this example:

- Multiple builds can happen simultaneously
- Only one deployment can happen at a time
- This is useful when you want to limit only certain critical jobs

### Conditional Cancellation

You might want different cancellation behavior for different branches:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # Only cancel in-progress runs on development branches
  # For release branches, let the current run finish before starting the new one
  cancel-in-progress: ${{ !contains(github.ref, 'release/') }}
```

In this example:

- For development branches: New workflow runs cancel in-progress runs (faster feedback)
- For release branches: New workflow runs wait for in-progress runs to finish (safer for releases)

### Real-World Example: Deployment Pipeline

```yaml

name: Deploy

on:
  push:
    branches: [main, staging, dev]

# Each environment gets its own concurrency group
jobs:
  deploy:
    runs-on: ubuntu-latest
    
    # Set environment based on branch
    environment:
      ${{ github.ref == 'refs/heads/main' && 'production' || 
          github.ref == 'refs/heads/staging' && 'staging' || 
          'development' }}
    
    # Concurrency based on the environment
    concurrency:
      group: deploy-${{ github.job }}-${{ github.ref }}
      cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}  # Don't cancel production deployments
    
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh

```
## Workflow Defaults

### Default Shell and Working Directory

```yaml
defaults:
  run:
    shell: bash                 # Default shell for run steps
    working-directory: ./scripts  # Default working directory

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: ./run-tests.sh     # Will run in ./scripts directory using bash
```

### Job-specific Defaults

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash
        working-directory: ./app
    steps:
      - uses: actions/checkout@v4
      - run: npm build          # Will run in ./app directory
```

### Available Shell Options

- `bash`: Default for non-Windows platforms
- `pwsh`: PowerShell Core (cross-platform)
- `python`: Python interpreter
- `sh`: Used as fallback on non-Windows if bash is not found
- `cmd`: Default for Windows
- `powershell`: PowerShell Desktop (Windows)

**Notes:**

- More specific defaults override broader ones (job-level overrides workflow-level)
- The working directory must exist on the runner before the shell runs
- Always ensure the chosen shell is available on the runner you're using
- For cross-platform workflows, consider using PowerShell Core (`pwsh`) which works on all platforms

## Reusable Workflows

### Creating a Reusable Workflow

```yaml
# .github/workflows/reusable-workflow.yml
name: Reusable workflow

on:
  workflow_call:                      # Indicates this is a reusable workflow
    inputs:                           # Inputs that can be passed to this workflow
      environment:
        required: true
        type: string
        description: 'The environment to deploy to'
    secrets:                          # Secrets that must be provided
      deploy-key:
        required: true                # This secret must be passed when called
    outputs:                          # Values this workflow exports
      result:
        description: "The result of the deployment"
        value: ${{ jobs.deploy.outputs.outcome }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      outcome: ${{ steps.deploy.outputs.result }}
    steps:
      - id: deploy
        run: echo "result=success" >> $GITHUB_OUTPUT
```

### Calling a Reusable Workflow

```yaml
name: Deploy Application

on:
  push:
    branches: [main]

jobs:
  call-deploy-workflow:
    uses: ./.github/workflows/reusable-workflow.yml   # Path to workflow file
    with:                                           # Inputs to pass
      environment: production
    secrets:                                        # Secrets to pass
      deploy-key: ${{ secrets.DEPLOY_KEY }}
```

### Nested Reusable Workflows

```yaml
jobs:
  setup:
    uses: ./.github/workflows/setup.yml
  
  deploy:
    needs: setup
    uses: ./.github/workflows/deploy.yml
    with:
      config: ${{ needs.setup.outputs.config }}
    secrets: inherit  # Pass all secrets to the called workflow
```

**Notes:**

- Reusable workflows make it easier to avoid duplicated code
- They can be called from multiple workflows, creating modularity
- You can pass inputs, secrets, and receive outputs
- `secrets: inherit` passes all available secrets to the called workflow
- Reusable workflows can be defined in the same repository or in public repositories

## Contexts and Expressions

GitHub Actions provides various contexts to access runtime information and evaluate expressions.

### Common Contexts

```yaml
jobs:
  example-job:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: Show contexts
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Actor: ${{ github.actor }}"
          echo "SHA: ${{ github.sha }}"
          echo "Job ID: ${{ job.id }}"
          echo "Runner OS: ${{ runner.os }}"
```

### Available Contexts

- `github`: Information about the workflow run and event
- `env`: Contains environment variables
- `job`: Information about the current job
- `steps`: Outputs from steps in the job
- `runner`: Information about the runner
- `secrets`: Access to secrets
- `inputs`: Inputs for reusable workflows or workflow dispatch
- `vars`: Repository, organization, or environment variables

### Expression Syntax

```yaml
jobs:
  example:
    if: ${{ github.ref == 'refs/heads/main' }}
    runs-on: ubuntu-latest
    steps:
      - name: Conditional step
        if: ${{ contains(github.event.head_commit.message, 'release') }}
        run: echo "This is a release commit"
```

### Functions in Expressions

```yaml
steps:
  - name: Example of functions
    if: ${{ startsWith(github.ref, 'refs/tags/') && contains(github.ref, 'release') }}
    run: echo "This is a release tag"
  
  - name: Always run even after failure
    if: ${{ always() }}
    run: echo "This step always runs"
```

**Notes:**

- Expressions are enclosed in `${{ }}` syntax
- You can use operators like `==`, `!=`, `&&`, `||` in expressions
- The `if` condition is evaluated before a job or step runs
- Available functions include `contains()`, `startsWith()`, `endsWith()`, `format()`, `join()`
- Special functions like `always()`, `success()`, `failure()` control workflow behavior

## Secrets and Environment Protection

### Using Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy with secret
        env:
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: ./deploy.sh --token "$API_TOKEN"
```

### Environment Protection

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production    # Reference a protected environment
    steps:
      - name: Deploy to production
        run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}  # Environment-specific secret
```

### Environment Configuration

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ steps.deploy.outputs.deployment-url }}  # Dynamic URL for deployment
    steps:
      - id: deploy
        run: |
          echo "deployment-url=https://example.com/deployment/$GITHUB_SHA" >> $GITHUB_OUTPUT
      what is GITHU_SHA
```

**Notes:**

- Secrets are encrypted and only exposed to selected actions
- Secrets are masked in logs if accidentally printed
- Environments can have protection rules like required reviewers
- Environment-specific secrets are only available to workflows using that environment
- Environments can have deployment URLs for quick access to deployed applications

## Common Patterns and Best Practices

### Caching Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Cache Node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      - name: Install dependencies
        run: npm ci
```

### Artifact Management ( what is that)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm run build
      
      - name: Upload build artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-files
          path: dist/
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v3
        with:
          name: build-files
          path: dist/
      
      - name: Deploy
        run: ./deploy.sh
```

### Setup and Teardown Pattern

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Setup testing environment
        run: ./setup-env.sh
        
      - name: Run tests
        run: npm test
        
      - name: Cleanup environment
        if: always()  # Run even if tests fail
        run: ./cleanup-env.sh
```

### Matrix with Custom Variables

```yaml
jobs:
  deploy:
    strategy:
      matrix:
        environment: [dev, staging, prod]
        include:
          - environment: dev
            url: dev.example.com
            debug: true
          - environment: staging
            url: staging.example.com
            debug: true
          - environment: prod
            url: example.com
            debug: false
    
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ matrix.environment }}
        run: |
          echo "Deploying to ${{ matrix.url }}"
          echo "Debug mode: ${{ matrix.debug }}"
```

### Conditional Job Execution

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint
  
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  
  notify-failure:
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: ${{ failure() && github.ref == 'refs/heads/main' }}
    steps:
      - name: Send failure notification
        run: ./notify-team.sh
```
