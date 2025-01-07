## How to set up auto deployment

Here, we use "Github actions" to automatically deploy new versions of the website whenever there is a new commit.
Since this website is designed to be deployed to an s3 bucket, we will create AWS IAM roles with correct policy settings that will enable the automated deployment job.

### Prerequisites

#### Github settings
* The repo `my-repo-name` must be part of an organization `my-organization-name` (i.e. not a personal repo)
    * Create an organization if you don't have one yet
    * Existing personal repos can be transferred to an organization as long as you are `admin`
        * Go to your repo Settings > General > Danger Zone > Transfer ownership
    * Set Workflow permissions in BOTH the organization AND repo
        * Navigate to organization > Settings > Actions (left column) > General
            * Select "Read and write permissions"
            * Check "Allow GitHub Actions to create and approve pull requests"
        * Do the same for your repo

#### AWS settings
* We assume that user has an IAM account already
* There are several ways to implement Github Actions and we will go with a "role-based" approach
    * Has 1 IAM user that is dedicated to executing Github Actions for the project (`github-actions-deployer`)
    * For each repo, create a new Role (attached to above User) that will execute the Github Action for that repo
* The alternative is a "user-based" approach that will allow 1 IAM user execute Github Actions for multiple repos, but requires sharing of same secrets and is less secure. It is easier for quick prototyping and the how-to is provided for reference.

### Steps before creating the Role

#### Create IAM root account
Skip to next part if you already have a root account.
Root account will not be directly used in your Github Actions, but it is a required first step if you're new to your IAM account. 
1. Create IAM root account (requires credit card, free during trial period). Log in to AWS Console
2. Click on your username in the top right
3. Click "Security credentials"
4. Under "Access keys", click "Create access key"
5. AWS will show you both the `Access Key ID` and `Secret Access Key`. Save them both, because this is the only time AWS will show you the Secret Access Key for your root account. You will not be using the root account for Github Actions.
6. (Optional) Download and install [AWS CLI](https://aws.amazon.com/cli/), then run `aws configure`:
    * Your `Access Key ID`
    * Your `Secret Access Key`
    * Your default region (e.g., `us-east-2`)
    * Your preferred output format (`json` is recommended)

#### Create an s3 bucket
This s3 bucket will be used to host your website `my-bucket-name`

1. Go to AWS Console > S3
2. Click "Create bucket"
3. Choose a bucket name (here, `my-bucket-name`)
4. Choose your region (e.g. `us-east-2`)
    * Note this down. This will be the `AWS_REGION` secret that you will be creating in your repo
5. Uncheck "Block all public access" (since it's a public website)
6. Enable static website hosting:
    * Go to bucket > Properties
    * Scroll to "Static website hosting"
    * Click "Edit"
    * "Enable" it and set `index.html` as your index document
7. Set your bucket policy to the following (Don't forget this step, or else you will get 403 AccessDenited error)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-bucket-name/*"
        }
    ]
}
```
8. Note the URL for your website. It should be something like `http://my-bucket-name.s3-website.us-east-2.amazonaws.com/`

#### Create Policy for s3 deployment
This policy will be attached to the new Role you will be creating for this repo

1. Go to IAM in AWS Console and create a new policy `github-actions-my-repo-name`
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket-name/*",
                "arn:aws:s3:::my-bucket-name"
            ]
        }
    ]
}
```
2. Note `YOUR_POLICY_ARN` and policy name for next step

#### Create a new IAM user (note: not role yet)
First, create one IAM user for managing Github Actions in general
Note that User is different from Role. One user will have multiple roles under it, and your Github Actions will be executed by Roles.
1. If you don't have one already, create a new IAM user for doing the deployment
```sh
aws iam create-user --user-name github-actions-deployer
```
2. Attach the policy you created earlier to this user
```sh
aws iam attach-user-policy --user-name github-actions-deployer --policy-arn YOUR_POLICY_ARN
```

#### Create Identity provider
Create a GitHub OIDC provider to be shared across multiple roles/repos.
1. Go to AWS Console > IAM > Identity providers > Add provider
2. For the provider configuration:
  * Select "OpenID Connect"
  * For Provider URL, enter: https://token.actions.githubusercontent.com
  * For Audience, enter: sts.amazonaws.com
  * Click "Add provider"
3. Note the GitHub OIDC provider to use for later. This is the base OIDC configuration. Later when creating a new role for your repo, you will be asked to select this base OIDC configuration and then make further adjustments (such as the Audience for this provider/repo name).

### Create a new IAM role
This is the role that will be referenced by your Github Actions yaml file.
You will be creating one role per repo.

1. Go to AWS Console > IAM > Roles > Create Role
2. Under "Trusted entity type" select "Web identity"
3. Under "Identity provider" select "token.actions.githubusercontent.com"
4. Under "Audience" select the GitHub OIDC provider that you created previously
5. Set the following:
    * GitHub organization: `my-organization-name`
    * GitHub repository - optional: `my-repo-name`
    * Click "Next"
6. Add permissions
    * Select the name of your policy created in previous step (e.g. `github-actions-my-repo-name`).
    * Click "Next"
7. Name your new role
    * e.g. `my-repo-name-deployer`
    * Click "Create role"
8. Note the Role ARN
    * This will be the `AWS_ROLE_ARN` secret that you will be creating in your repo

#### Set Trust relationships for your role
This is a critical step and it's easy to make typos so pay attention.
Trust relationships specify which entity can assume the above newly created role.
1. Go to AWS console > IAM > Roles > Select your new role `my-repo-name-deployer`
2. Navigate to Trust relationships tab
3. Add trust policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::my-account-id-123:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:my-organization-name/my-repo-name:*"
                }
            }
        }
    ]
}
```
4. Some pointers:
    * The `sts:AssumeRoleWithWebIdentity` is controlled through Trust relationships, and does not need to be added to Policy.
    * If your value contains a wildcard symbol (asterisk *), make sure to use the Condition key "StringLike". Wildcards are not compatible with "StringEquals".
    * If you run into issues, make sure to check that there are no typos in "repo:my-organization-name/my-repo-name:*"

### Create new Github Action
You will need:
* `AWS_S3_BUCKET` set to `my-bucket-name`
* `AWS_REGION` of s3 bucket
* `AWS_ROLE_ARN` of `my-repo-name-deployer`

#### Set your secrets in GitHub repo
1. Go to your repo in Github > Settings > (left column) Secrets and variables > Actions
2. Add the following New repository secrets:
    * Set `AWS_S3_BUCKET` to `my-bucket-name`
    * Set `AWS_REGION` to region of s3 bucket (e.g. `us-east-2`)
    * Set `AWS_ROLE_ARN` associated with the previously created role `my-repo-name-deployer`

#### Create workflow file
1. Create the workflow file in expected folder structure
```sh
mkdir -p .github/workflows/
touch .github/workflows/deploy.yml
```

2. Set contents of YAML file
Use aws-actions/configure-aws-credentials@v2 (Note the v2, not v1)
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Deploy to S3
        run: |
            aws s3 sync . s3://${{ secrets.AWS_S3_BUCKET }} --exclude ".git/*" --exclude ".github/*"
```

### Test it out
1. Go back to the repo you created locally

2. Add, commit and push your new files to Github server. 
```sh
git add -A
git commit -m "Initial commit"
git push -u origin main
```

3. The commit should trigger a Github Action to deploy `index.html` to s3 bucket. Go to Github and click the `Actions` tab to see job status and errors if any

4. In your browser, navigate to the URL you noted previously. For example: http://my-bucket-name.s3-website.us-east-2.amazonaws.com/


### (Not recommended) User-based approach

This isn't the most secure method, but it gets the job done. It is easier because you don't have to setup any Identity providers (OIDC).
Refer to previous section and have these ready:
* s3 bucket `my-bucket-name`
* IAM policy ARN `YOUR_POLICY_ARN`
* The policy is already attached to IAM user `github-actions-deployer`

#### How-to
1. Create access keys (secrets) for IAM user. Note the `AccessKeyId` and `SecretAccessKey`
```sh
# AWS CLI
aws iam create-access-key --user-name github-actions-deployer
```

2. Navigate to Github and add the secrets to the repo
* Go to Settings > Secrets and variables > Actions
* Add two new secrets:
  * Name: `AWS_ACCESS_KEY_ID`, Value: `AccessKeyId` from previous step
  * Name: `AWS_SECRET_ACCESS_KEY`, Value: `SecretAccessKey` from previous step

3. Create a workflow file in your repo: `.github/workflows/deploy.yml`
Use aws-actions/configure-aws-credentials@v1 (Note the v1, not v2)
```yaml
name: Deploy to AWS
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: your-region (e.g. us-east-2)
    
    - name: Deploy to S3
      run: |
        aws s3 sync . s3://${{ secrets.AWS_S3_BUCKET }} --exclude ".git/*" --exclude ".github/*"
```
