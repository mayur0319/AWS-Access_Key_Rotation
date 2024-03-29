# Automation of IAM keys rotation at scale with AWS Organizations

# Use Case
To protect AWS account from compromised IAM access keys, rotating them regularly as a security best practice in IAM.
For AWS Organizations, this solution identifies all AWS account IDs within your organization and applies to them. The AWS Lambda function will run in “provisioning-p01” account will assume “provisioning-admin” IAM role to locally run the rotation functions across multiple accounts.
· New IAM access keys are generated when existing access keys are older than 80 days.  
· The user receives the new access keys in password-protected zip file through email notifications using the SES service.
· The previous access keys are deactivated at 90 days old, and then deleted at 100 days old.
· An email notification will be sent to the user as per the applicable scenario.
Lambda functions and Amazon CloudWatch automatically perform these actions. Access keys shared via email can be used to retrieve the new access key pair and use them in your code or applications. The rotation, deletion, and deactivation periods can be customized.
Target technology stack
· Amazon Event Bridge· IAM· AWS Lambda· AWS Organizations  · Amazon CloudWatch· SES· SNS

# Prerequisites
· AWS Organizations configured and set up.
· To run full functionality each IAM user should have tag: Email Id with proper email id.
· To exclude the user from key rotation activity, apply a tag Exempt with value True.
· Each account must have “provisioning-admin” IAM role.
· The sender’s Email Id must be configured into SES.
· To receive Lamba execution failure notifications SNS should be configured.
· Before running the functionality, the below environment variable must be configured with the given value.
o limit_days = 90
o notice_period = 10
o max_users =200o SENDER_EMAIL = <Pass the mail Id, which will be used to send the email notification to the users.>
o Clean_up = True or False
o exclude_acct = <Pass the list of accounts that wants to be excluded from key rotation activity>
· To run initial clean up only, set Clean_up = True

# Configuration details:
· Lambda function name: iam-access-key-rotator
· EventBridge (CloudWatch Events name): iam-access_key_rotator-rule
· EventBridge Schedule: Invokes every day at 13:00 UTC.
· SES configuration sets: iam-access-key-rotator-configset
· SNS topic name: iam-access-key-rotator-topic
· IAM role: iam-access-key-rotator-role

#Architecture:  


# Workflow:
1. An Event Bridge event initiates Lambda function every 24 hours at 9 UTC.
2. The “sn-iam-access-key-rotator” Lambda function will assume “provisioning-admin” IAM role to access the AWS Organizations management account and list down of all AWS account IDs.
3. The Lambda function will assume “provisioning-admin” IAM role to access the target account. For each user, it will process the following information:· The username· The email ID that will be used by SES to notify user.· The details of the access keys
4. If any of the IAM access keys has exceeded the 80-days threshold, the Lambda function determine which rotation action to perform based on the following rules:
· If the user has only one key and it’s nearing its expiry, then a new key pair will be generated and shared with the user.
· If the user has two keys and older key is inactive, then the older key will be deleted and a new key will be generated.
· If the user has two keys and younger key is inactive, then the younger key will be deleted and new key will be generated.
· If the user has two active keys, and one of them is nearing its expiry, the oldest key will be deactivated first (with 10 days’ notice), and the user will be informed to use the other key.
5. If any of the IAM access keys was not used from last 90 days, then the access keys will be deleted, and user get notified about deletion of keys.
6. Whenever a rotation action is performed, the Lambda function sends an email notification to the user using SES. If a new key pair is generated, it will be shared in a password-protected zip file and the password to access the file will be sent in a separate email to the user.

(Note: This Lambda function can also exclude specific AWS accounts. The list of AWS account IDs that are passed through the parameter “exclude_acct“ are excluded from the rotation action.)

The solution will also perform an initial cleanup (First time only)· Any existing inactive key that is more than 100 days old will be deleted. No notification will be sent to the user for this clean-up activity.· Any existing active keys more than limit days, will be deactivated for user which doesn’t have email id tag with value.· After successful initial cleanup, consolidate data will be sent to the AWS Support team.
