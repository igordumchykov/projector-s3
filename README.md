# projector-s3

# Cloudformation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Creates S3 bucket with object lock enabled and bucket for logs

Resources:
  S3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Sub "my-bucket-${AWS::AccountId}-${AWS::Region}"
      ObjectLockEnabled: true
      LoggingConfiguration:
        DestinationBucketName: !Ref S3LogsBucket
        LogFilePrefix: "logs/"
      ObjectLockConfiguration:
        ObjectLockEnabled: Enabled
        Rule:
          DefaultRetention:
            Days: 1
            Mode: GOVERNANCE
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
  S3LogsBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: LogDeliveryWrite
      OwnershipControls:
        Rules:
          - ObjectOwnership: ObjectWriter
```

# Run

1. Create a stack
```shell
aws cloudformation create-stack --stack-name RestrictedBucketStack --template-body file:///<path-to-file>/template.yml
```

2. Check buckets

```shell
 aws s3 ls
```

Output:
```
2023-10-05 16:22:38 my-bucket-781744813347-eu-central-1
2023-10-05 16:22:15 restrictedbucketstack-s3logsbucket-1a5o0g0ej4q8k
```

3. Add object:

```shell
 aws s3 cp ./test-file.txt s3://my-bucket-781744813347-eu-central-1/test-file.txt
```

4. Check logs:
```shell
aws s3 ls s3://restrictedbucketstack-s3logsbucket-1a5o0g0ej4q8k/logs/
```

Output:
```
2023-10-05 16:23:08        530 2023-10-05-14-23-07-618AD37704BE5040
2023-10-05 16:31:24        690 2023-10-05-14-31-23-5C028A65E5355163
```

You'll also find that you can apparently overwrite an object with a new upload, but again you can't actually do that, because uploading an object with the same key in a versioned bucket (and enabling versioning is mandatory for object lock to work) doesn't overwrite the object -- it just creates a newer version of the object, leaving older versions intact.

When the top (newest, current) version of an object is a delete marker, the object disappears from the console and isn't included in ListObjects requests sent to the bucket via the API, but does appear in ListObjectVersions API requests. The "show/hide" setting is only applicable to your personal console view, it doesn't change actual bucket behavior.

The timestamps on object versions can't be altered, so locking an object version not only prevents deletion of the object contents, it also preserves a record of when that object was originally created. "Overwriting" an object creates a new version with a new timestamp, and the timestamps on the versions prove what content existed in the bucket at any given point in time.
