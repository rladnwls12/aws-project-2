## S3 IAM Policy

대충 S3 예시
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "1",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::demo-bucket-346151107821-ap-northeast-2-an"
        },
        {
            "Sid": "2",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::demo-bucket-346151107821-ap-northeast-2an/*",
            "Condition": {
                "StringEquals": {
                    "s3:ExistingObjectTag/Owner": "${aws:username}"
                }
            }
        }
    ]
}
```
