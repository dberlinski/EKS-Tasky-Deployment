# S3 public bucket creation via AWS CLI

This needs to be completed in multiple steps given that the “public block” feature is enabled in all new buckets by default.

1. Create a new bucket.
```
aws s3api create-bucket --bucket <bucketname> --region <aws-region> --create-bucket-configuration LocationConstraint="<aws-region>" --object-ownership BucketOwnerPreferred
```

2. Delete bucket "public block".
```
aws s3api delete-public-access-block --bucket <bucketname>
```

3. Create public access policy document.
```sh
cat >> s3-policy-bucketname.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::bucketname/*"
            ]
        }
    ]
}
EOF
```

4. Apply policy to bucket.
```sh
aws s3api put-bucket-policy --bucket bucketname --policy file://s3-policy-bucketname.json
```



