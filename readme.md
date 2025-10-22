Prerequisites below:
You’ll need to create that role once
Create an IAM role your workflow can assume via OIDC, and store its ARN as a repo secret AWS_DEPLOY_ROLE_ARN.

Trust policy (allow GitHub OIDC for your repo):
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
Attach a minimal policy (you can scope tighter later):
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
Tip: For production, replace "ec2:*" with a least-privilege set (e.g., ec2:RunInstances, ec2:CreateTags, etc.), and lock the trust policy to the exact branch/environment.


How it works (high level)

Terraform launches an Amazon Linux 2023 EC2 in a public subnet, adds a security group for HTTP:80, and injects user_data that installs Git/Ansible and runs:

ansible-pull -U <this repo> ansible/site.yml -e web_title="..."


Ansible installs httpd, enables & starts it, and templates index.html with your web_title.

CI/CD validates, plans, applies, then curls the URL to confirm it’s serving the page. The fetched index.html is uploaded as a workflow artifact.

Customization

Change web_title via Terraform var (already wired into CI).

Replace templates/index.html.j2 with your own content.

Restrict ingress by setting allowed_http_cidr to your office IP.