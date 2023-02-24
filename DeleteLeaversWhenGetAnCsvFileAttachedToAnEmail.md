- Set up an email client to automatically download and save the attachment to a specific folder in AWS S3.

```
import boto3
import email
import os
import re

def save_email_attachment(bucket_name, email_key):
    s3 = boto3.resource('s3')
    bucket = s3.Bucket(bucket_name)
    object = bucket.Object(email_key)

    # Download the email object
    email_object = email.message_from_bytes(object.get()["Body"].read())

    # Find the attachment and save it to S3
    for part in email_object.walk():
        filename = part.get_filename()

        if filename:
            attachment = part.get_payload(decode=True)

            # Save the attachment to S3
            s3_filename = os.path.join('email_attachments', filename)
            bucket.put_object(Key=s3_filename, Body=attachment)

            return s3_filename

```

- Set up an AWS Lambda function that will be triggered by the S3 object created event. This function will extract the user email list from the CSV file and save it to a temporary file.
``` 
import csv
import tempfile

def process_email_attachment(bucket_name, s3_filename):
    s3 = boto3.resource('s3')
    bucket = s3.Bucket(bucket_name)
    object = bucket.Object(s3_filename)

    # Read the attachment CSV file
    csv_content = object.get()["Body"].read().decode('utf-8')
    csv_reader = csv.DictReader(csv_content.splitlines())

    # Create a temporary file to store the list of user emails
    with tempfile.NamedTemporaryFile(mode='w', delete=False) as temp_file:
        for row in csv_reader:
            email_address = row['Email']
            temp_file.write(email_address + '\n')

    return temp_file.name
```

- Use the AWS SDK to delete the users from the AWS account using the temporary file with the list of user emails.
```
import boto3

def delete_users_from_aws(temp_filename):
    iam = boto3.client('iam')

    with open(temp_filename) as f:
        for line in f:
            email_address = line.strip()

            # Find the user by email address
            response = iam.list_users(
                Filter=f"email='{email_address}'"
            )

            # If the user exists, delete them
            if len(response['Users']) > 0:
                user_name = response['Users'][0]['UserName']
                iam.delete_user(UserName=user_name)

    return True

```

- Once the deletion process is complete, send a notification email to aaa@gmail.com to confirm that the process was successful.
```
import boto3
from botocore.exceptions import ClientError

def send_email_notification(subject, body, recipient):
    SENDER = "Your Name <your_email@your_domain.com>"
    RECIPIENT = recipient
    AWS_REGION = "us-west-2"

    # The email body for recipients with non-HTML email clients.
    BODY_TEXT = body

    # The HTML body of the email.
    BODY_HTML = f"<html><body><p>{body}</p></body></html>"

    # The character encoding for the email.
    CHARSET = "UTF-8"

    # Create a new SES resource and specify a region.
    client = boto3.client('ses', region_name=AWS_REGION)

    # Try to send the email.
    try:
        # Provide the contents of the email.
        response = client.send_email(
            Destination={
                'ToAddresses': [
                    RECIPIENT,
                ],
            },
            Message={
                'Body': {
                    'Html': {
                        'Charset': CHARSET,
                        'Data': BODY_HTML,
                    },
                    'Text': {
                        'Charset': CHARSET,
                        'Data': BODY_TEXT,
                    },
                },
                'Subject': {
                    'Charset': CHARSET,
                    'Data': subject,
                },
            },
            Source=SENDER,
        )
    # Display an error if something goes wrong.
    except ClientError as e:
        print(e.response['Error']['Message'])
        return False
    else:
        return True

```
