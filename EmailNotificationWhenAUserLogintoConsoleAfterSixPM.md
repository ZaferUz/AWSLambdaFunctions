- 

``` 
import boto3
import datetime

def lambda_handler(event, context):
    # Get the current time in UTC
    current_time = datetime.datetime.utcnow()

    # Get the current hour in the user's timezone
    user_timezone = 'Europe/London'  # Change this to your timezone
    user_time = current_time.astimezone(pytz.timezone(user_timezone))
    user_hour = user_time.hour

    # Check if the user logged in after 6pm
    if user_hour >= 18:
        # Get the ARN of the IAM user who logged in
        iam = boto3.client('iam')
        user_arn = event['detail']['userIdentity']['arn']

        # Get the email address of the IAM user
        user = iam.get_user(UserName=user_arn.split(':')[-1])['User']
        email = next(iter([tag['Value'] for tag in user['Tags'] if tag['Key'] == 'Email']), None)

        if email:
            # Send an email notification
            ses = boto3.client('ses')
            ses.send_email(
                Source='your-email@example.com',
                Destination={
                    'ToAddresses': [email],
                },
                Message={
                    'Subject': {
                        'Data': 'Console login after 6pm',
                    },
                    'Body': {
                        'Text': {
                            'Data': f'The IAM user {user_arn} logged in to the AWS Management Console after 6pm {user_timezone}.',
                        },
                    },
                },
            )

```
- This Lambda function uses the AWS CloudTrail service to receive events when an IAM user logs in to the AWS Management Console. The function then checks the current time in the user's timezone and sends an email notification if the login occurred after 6pm. You will need to modify the user_timezone and Source email address to match your own environment.

- To use this function, you will need to configure an AWS CloudTrail trail to log management console sign-in events and create an IAM role for the Lambda function with permissions to read the CloudTrail event logs and send emails using the Amazon SES service. Finally, you can create a new Lambda function in the AWS Management Console and copy the code into the function editor
