## Create an IAM role with permissions to access EC2 and S3 services. Grant the following permissions to the IAM role:
```
ec2:DescribeVolumes
ec2:DeleteVolume
s3:PutObject
s3:GetObject
s3:DeleteObject
```
- Create a new Lambda function in the AWS Management Console, choose Python as the runtime, and select the IAM role that was created in step 1.

- Copy the following code into the Lambda function code editor:

```
import boto3
from datetime import datetime, timedelta
import csv

def lambda_handler(event, context):
    
    ec2 = boto3.client('ec2')
    s3 = boto3.client('s3')
    
    # Get current date and time
    current_time = datetime.now()
    
    # Calculate the time 90 days ago
    past_time = current_time - timedelta(days=90)
    
    # Get a list of all EBS volumes in the account
    response = ec2.describe_volumes()
    volumes = response['Volumes']
    
    # Create a CSV file to store the details of unused EBS volumes
    csv_file = csv.writer(open('/tmp/ebs_unused.csv', 'w'))
    csv_file.writerow(['VolumeId', 'CreateTime', 'LastUsed'])
    
    # Loop through each volume and check if it's been used in the last 90 days
    for volume in volumes:
        last_used = volume.get('Attachments')[0].get('AttachTime')
        last_used_time = datetime.strptime(last_used, '%Y-%m-%dT%H:%M:%S.%fZ')
        
        if last_used_time < past_time:
            # Write the unused volume details to the CSV file
            csv_file.writerow([volume['VolumeId'], volume['CreateTime'], last_used])
    
    # Upload the CSV file to S3
    s3.upload_file('/tmp/ebs_unused.csv', 'my-bucket', 'ebs_unused.csv')
    
    # Download the CSV file from S3
    s3.download_file('my-bucket', 'ebs_unused.csv', '/tmp/ebs_unused.csv')
    
    # Create a list to store the EBS volume IDs to delete
    volumes_to_delete = []
    
    # Open the CSV file and loop through each row
    with open('/tmp/ebs_unused.csv', 'r') as csv_file:
        reader = csv.reader(csv_file)
        next(reader) # Skip header row
        for row in reader:
            volumes_to_delete.append(row[0])
    
    # Delete the EBS volumes
    for volume_id in volumes_to_delete:
        ec2.delete_volume(VolumeId=volume_id)
    
    # Update the CSV file with the deletion status
    updated_csv_file = csv.writer(open('/tmp/ebs_unused_updated.csv', 'w'))
    updated_csv_file.writerow(['VolumeId', 'CreateTime', 'LastUsed', 'DeletionStatus'])
    
    with open('/tmp/ebs_unused.csv', 'r') as csv_file:
        reader = csv.reader(csv_file)
        next(reader) # Skip header row
        for row in reader:
            if row[0] in volumes_to_delete:
                deletion_status = 'Deleted'
            else:
                deletion_status = 'Not deleted'
            
            # Write the updated details to the new CSV file
            updated_csv_file.writerow([row[0], row[1], row[2], deletion_status])
    
    # Upload the updated CSV file to S3
    s3.upload_file('/tmp/ebs_unused_updated.csv', 'my-bucket', 'ebs_unused_updated.csv')
    ```
