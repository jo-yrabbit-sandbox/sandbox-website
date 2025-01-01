## How to create a static website
* Simple website that shows PGP public key, gives users option to download the public key file
* Hosted on AWS s3 server
* Deployment automated using Github Actions

### Set up repo

1. Set up a new git repo
```sh
mkdir 
cd sandbox-website
git init
```
2. Create a `.gitignore` file:
```sh
echo "*.DS_Store" > .gitignore
echo "*.env" >> .gitignore
```

3. Create the new repo on Github
* Go to github.com
* Click the '+' icon in the top right
* Select "New repository"
* Name it "sandbox-website"
* Don't initialize the repo with README. Click "Create repository"
```sh
git remote add origin https://github.com/YOUR-USERNAME/sandbox-website.git
git branch -M main
git add .gitignore
git commit -m "Initial commit of .gitignore"
git push -u origin main
```


### Create website content

It can be anything.
1. Create `index.html` with webpage content. It can be anything. Below is an example which shows your PGP public key as text
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My PGP Key</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        .key-container {
            background-color: white;
            padding: 20px;
            border-radius: 5px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .pgp-key {
            background-color: #f8f8f8;
            padding: 15px;
            border: 1px solid #ddd;
            border-radius: 3px;
            font-family: monospace;
            white-space: pre-wrap;
            word-wrap: break-word;
        }
        .download-button {
            display: inline-block;
            background-color: #4CAF50;
            color: white;
            padding: 10px 20px;
            text-decoration: none;
            border-radius: 3px;
            margin-top: 20px;
        }
        .download-button:hover {
            background-color: #45a049;
        }
    </style>
</head>
<body>
    <div class="key-container">
        <h1>My PGP Public Key</h1>
        <p>You can use this public key to send me encrypted messages or verify signatures.</p>
        
        <div class="pgp-key">
-----BEGIN PGP PUBLIC KEY BLOCK-----
[Your PGP public key goes here]
-----END PGP PUBLIC KEY BLOCK-----
        </div>

        <a href="pubkey.asc" class="download-button">Download Public Key</a>
    </div>
</body>
</html>
```

2. If you don't have your PGP public key handy, create a new PGP key pair, then copy the public key into the placeholder in `index.html`
```sh
gpg --full-generate-key
gpg --armor --export your.email@example.com > pubkey.asc
cat pubkey.asc  # Copy this into the placeholder inside `index.html`
```

3. Save both `index.html` and `pubkey.asc` in top-level folder of your repo


### Set up deployment

This isn't the most secure method, but it gets the job done.

#### Generate your secrets
1. Create IAM root account (requires credit card, free during trial period). Log in to AWS Console
2. Click on your username in the top right
3. Click "Security credentials"
4. Under "Access keys", click "Create access key"
5. AWS will show you both the `Access Key ID` and `Secret Access Key`. Save them both, because this is the only time AWS will show you the Secret Access Key!

#### Setup AWS CLI
1. Download and install AWS CLI
2. Run `aws configure` and enter:
* Your Access Key ID
* Your Secret Access Key
* Your default region (e.g., us-east-2)
* Your preferred output format (json is recommended)

#### Create an s3 bucket for hosting your website
1. Go to S3 in AWS Console
2. Click "Create bucket"
3. Choose a bucket name (here, sandbox-website-bucket)
4. Choose your region (e.g. us-east-2)
5. Uncheck "Block all public access" (since it's a public website)
6. Enable static website hosting:
* Go to bucket > Properties
* Scroll to "Static website hosting"
* Click "Edit"
* "Enable" it and set `index.html` as your index document
7. Set your bucket policy to the following (don't forget, or else you will get 403 AccessDenited error)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::sandbox-website-bucket/*"
        }
    ]
}
```
8. Note the URL for your website. It should be something like `http://<bucket-name>.s3-website.<your-region>.amazonaws.com/`

#### Create an IAM policy for doing the deployment
1. Go to IAM in AWS Console and create a new policy
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
                "arn:aws:s3:::sandbox-website-bucket/*",
                "arn:aws:s3:::sandbox-website-bucket"
            ]
        }
    ]
}
```

2. Note the policy ARN for next step

3. Create a new IAM user for doing the deployment, then attach the policy to this user
```sh
aws iam create-user --user-name github-actions-deployer
aws iam attach-user-policy --user-name github-actions-deployer --policy-arn YOUR_POLICY_ARN
```

4. Create access keys (secrets) for this user. Note the `AccessKeyId` and `SecretAccessKey`
```sh
aws iam create-access-key --user-name github-actions-deployer
```

5. Navigate to Github and add the secrets to the repo
* Go to Settings > Secrets and variables > Actions
* Add two new secrets:
  * Name: AWS_ACCESS_KEY_ID
  * Value: `AccessKeyId` from previous step
  and
  * Name: AWS_SECRET_ACCESS_KEY
  * Value: `SecretAccessKey` from previous step

6. Create a workflow file in your repo: `.github/workflows/deploy.yml`
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
        aws s3 sync . s3://sandbox-website-bucket --exclude ".git/*" --exclude ".github/*"
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

4. In your browser, navigate to the URL you noted previously. For this example: [http://sandbox-website-bucket.s3-website.us-east-2.amazonaws.com](http://sandbox-website-bucket.s3-website.us-east-2.amazonaws.com)

![pgp-website](pgp-website.png)
