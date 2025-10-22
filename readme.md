# Prerequisites below:
You’ll need to create iam role and  create OIDC connectivity from AWS to GitHub Actions 
1) AWS Account
2) GitHub Account

Follow instructions below
1) Login to AWS account and go to IAM.
2) click on Identity providers
3) click on Add provider https://token.actions.githubusercontent.com 
4) add the provider URL and click on Get thumbprint. it will add the thumbprint, then add the audience sts.amazonaws.com
5) add optional tags, if required and click on Add provider.
6) now click on assign role and select create a new role/use an existing role if there’s one already.
7) click next.
8) next select the audience and add the GitHub organization name for example, https://github.com/google.com add google as your organization. repository and branch is optional values
9) click next, add required permissions. here I’m adding Administrator Access just for demonstrating purposes. (it’s better to no to use Admin Access to minimize the blast radius of resources) note: always use least privilege access to follow the best practices
10) click next, add the name for role and description and then click on create role trust relationship example is below for reference.


  Trust policy (allow GitHub OIDC for your repo):
  ```json
  {
    "Version":"2012-10-17",
    "Statement":[
      {
        "Effect":"Allow",
        "Principal":{"Federated":"arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},
        "Action":"sts:AssumeRoleWithWebIdentity",
        "Condition":{
          "StringEquals":{
            "token.actions.githubusercontent.com:aud":"sts.amazonaws.com"
          },
          "StringLike":{
            "token.actions.githubusercontent.com:sub":"repo:<YOUR_ORG>/<YOUR_REPO>:*"
          }
        }
      }
    ]
  }
```
Attach a minimal policy (you can scope tighter later):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": [
      "ec2:*",
      "iam:PassRole",
      "ssm:*",
      "cloudwatch:*"
    ], "Resource": "*"}
  ]
}
```
Tip: For production, replace "ec2:*" with a least-privilege set (e.g., ec2:RunInstances, ec2:CreateTags, etc.), and lock the trust policy to the exact branch/environment.


# How it works (high level)

Terraform launches an Amazon Linux 2023 EC2 in a public subnet, adds a security group for HTTP:80, and injects user_data that installs Git/Ansible and runs:

ansible-pull -U <this repo> ansible/site.yml -e web_title="..."

Ansible installs httpd, enables & starts it, and templates index.html with your web_title.

CI/CD validates, plans, applies, then curls the URL to confirm it’s serving the page. The fetched index.html is uploaded as a workflow artifact.

# Customization

Change web_title via Terraform var (already wired into CI).

Replace templates/index.html.j2 with your own content.

Restrict ingress by setting allowed_http_cidr to your office IP.
