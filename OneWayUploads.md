One-way Uploads
===============

Allow file upload to a secure location without credentials.

## Features

- [x] Cloudformation template
- [x] Lambda function for get signed url
- [x] Working html page to upload file to bucket
- [x] Shell script to test CORS
- [x] Shell script to test upload
- [x] Shell script to deploy to AWS
- [x] Allow video and image media formats
- [x] Progress bar during file upload
- [x] S3 date key prefix for objects uploaded
- [x] Limit signed url expiration to a few seconds
- [x] Prevent upload clobbering using Object versioning
- [x] Retain the bucket even if the stack is deleted
- [x] SSE-S3 enforced
- [x] Option for access logging
- [x] Option to block access using CIDR IP addresses
- [x] Option to rename file before upload
- [x] Host HTML application using S3 static website
- [x] Enforce full control of object by bucket owner
- [x] Bucket deletion policy - prevent non-root from deleting it
- [x] Root-only bucket object access policy
- [ ] Drag-and-drop file upload
- [ ] Support other S3 actions (such as multipart upload) using the same Lambda Function

## Design

Since the upload destination (archive bucket) must not be publicly accessible, then a client must have permission to access it privately. An S3 client side JavaScript SDK requires credentials, and can do this using AWS Cognito with an identity provider. While this is a valid solution, it can be hard to configure and manage.

S3 signed URLs is another technique to interact with objects in a private Bucket. Again credentials are required to work with a private S3 Buckets. To work around this, a Lambda Function can be granted permission to generate signed URLs for `s3:PutObject` requests to the Bucket. This will keep credentials on the server-side, while the client browser simply needs to have the address of the Lambda API Gateway Invoke URL.

Now a browser application without credentials can upload files to a private S3 bucket. However, additional security measures need to be taken.

1. Archive (S3 Bucket)
    - Restrict access to a CIDR IP range
    - Block all public access
    - Enable versioning
    - Enable access logs
    - Deletion policy set to Retain
    - Server-side encryption as default
    - CORS Policy for PUT methods only
2. Lambda Function (API Gateway Rest API)
    - Restrict Invoke access to a CIDR IP range
    - Enforce bucket owner full control of object
    - Enforce server-side encryption
    - Restrict `s3:PutObject` to specific Lambda source
    - Generate signed URL and
        - Restrict file name (and prefix)
        - Restrict file type
        - Restrict ACL
        - Restrict server-side encryption
        - Short time-to-live / expiration
3. Website (S3 Bucket)
    - Restrict access to a CIDR IP range
    - Use a HTTP Query Parameter to configure the API Gateway Invoke URL to avoid exposing it publicly in HTML

### Workflow

When a user accesses the Website, Archive or API, they must have an IP address that matches a CIDR IP range. When the IP address is allowed, the website will load. To configure the API address on the client-side, this is either provided as a user supplied field or a query parameter, example:

`https://WEBSITE-BUCKET-NAME.s3.amazonaws.com/index.html?apiId=API-ID`

The API address information will be provided by the administrator, and will not be hard coded in the application. Without the configuration, the Lambda Invoke URL remains unknown.

When the user uploads a file, the client-side application first requests a signed URL from the API, and then attempts a PUT requests against the signed URL. That is to say that the Lambda function is only responsible for generating the URL but not handling the file transfer - S3 handles file transfer. This design shifts the responsibility of scaling upload and upload complexity to AWS S3.

## Setup

1. Configure environment variables for bash scripts
    1. Make a copy of the `example.env` template `cp example.env .env`
    1. Edit `.env` and customize
1. Deploy API to AWS
    ```bash
    ./scripts/deploy-api.sh
    ```
1. (Optional) Deploy website to AWS
    ```bash
    ./scripts/deploy-website.sh
    ```
1. Determine the **API Gateway Invoke URL** from the Stack Outputs
    ```bash
    cat cloudformation/api-outputs.json
    ```
1. Test setup
    ```bash
    # TODO: Update with your API Gateway Invoke URL
    export API_BASE_URL='https://APIGATEWAY-RESTAPI-ID.execute-api.us-east-1.amazonaws.com/live'

    ./scripts/test-cors.sh
    ./scripts/test-upload.sh
    ```
1. Open the web application
    ```bash
    open ./www/index.html
    ```

## Locking and Unlocking the Archive

**WARNING:** Once the policy is applied: full access to archive is irreversible without AWS account root access.

### "Lock" the Archive

This is the proverbial: "lock the door and throw away the key".

**Who can run this script?** Root and regular users and roles.

**What does this script do?** Adds a bucket policy to deny all actions except Put Object actions by regular AWS users and roles. Full access to the bucket is only available to root.

```bash
./scripts/lock-archive.sh
```

### "Unlock" the Archive

**Who can run this script?** Only root.

**What does this script do?** Removes the bucket policy, to allow full access to the archive by all users. Alternatively, login to AWS console and remove the bucket policy.

Option 1:

Use the [AWS support guide](https://aws.amazon.com/premiumsupport/knowledge-center/s3-accidentally-denied-access/) and navigate to the AWS Console to login with AWS root account and remove the bucket policy.

Option 2:

Not recommended.

Setup programmatic AWS root account access key and secret key, then run the following script.

```bash
./scripts/unlock-bucket.sh
```

## End-user Setup

For end-user experience, an HTML standalone project is available in the `./www` directory.

### Option 1

Host web files using S3 by deploying the website:

```bash
./scripts/deploy-website.sh
```

Provide users with an URL to the hosted web files. Include the API ID in the query string.

Template: `${WebsiteUrl}?apiId=${ApiId}`

Example: `https://WEBSITE-BUCKET-NAME.s3.amazonaws.com/index.html?apiId=API-ID`

- The `WebsiteUrl` of the S3 bucket hosting the website can be found in the Website Stack Outputs `cloudformation/website-outputs.json`
- The `ApiId` of the API Gateway Rest API can be found in the API Stack Outputs `cloudformation/api-outputs.json`

### Option 2

The end-user should be sent a zip file with all the contents of the `./www` folder.

Hardcode the API ID, and use the following command to package the zip.

```bash
zip --verbose --recurse-paths one-way-uploads.zip www
```

## Interacting with API

Example: request a signed url for S3 PutObject action.

```bash
curl
  --request POST \
  --header 'Content-Type: application/json' \
  --data '{"filename": "test.jpg", "filetype": "image/png"}' \
    https://APIGATEWAY-RESTAPI-ID.execute-api.us-east-1.amazonaws.com/live/get-signed-url
```

Example: output from signed url route.

```json
{
  "url":"https://A-VERY-LONG-SIGNED-URL",
  "key":"2022-11-03/test.jpg",
  "ttl":30
}
```

Example: upload a file using the new signed url.

```bash
curl \
  --request PUT \
  --header 'Content-Type: image/png' \
  --header 'x-amz-acl: bucket-owner-full-control' \
  --header 'x-amz-server-side-encryption: AES256' \
  --upload-file 'test.jpg' \
  --location \
    https://A-VERY-LONG-SIGNED-URL
```

## Bucket Security

The bucket is [encrypted using SSE-S3](https://aws.amazon.com/blogs/aws/new-amazon-s3-server-side-encryption/) as default, and the lambda function is forced to use AES256 encryption for S3 PutObject actions.

### Security Recommendations

- Use a least privilege approach
- Versioning must be enabled
- Objects must be encrypted
- Access logging must be enabled and configured to write to different S3 bucket
- Block all public access
- Allow required actions only, using wildcards like `s3:*` is unacceptable
- IAM roles used for access and scoped to specific buckets

## Useful Resources

- [API Gateway CORS](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-cors.html)
- [POST Policy versus PUT Policy](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-HTTPPOSTConstructPolicy.html)
- [S3 PutObject Using Signed URL](https://fullstackdojo.medium.com/s3-upload-with-presigned-url-react-and-nodejs-b77f348d54cc)
- Multipart Uploads
    - https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
    - https://www.altostra.com/blog/multipart-uploads-with-s3-presigned-url

## Troubleshooting

Although adding new features can be challenging when using strict security policies, the benefits outweigh the cost. Here are some useful resources and tips to help you get what you need done.

### CORS

To get the app working with CORS + API Gateway + Lambda, here is a check list:
- Set CORS headers on the S3 bucket
- Set CORS headers in the response of the Lambda function
- Set CORS headers in an OPTIONS method for the ApiGateway resource

### CloudTrail

AWS events can be researched for in CloudTrail logs if the payload contains an access key id. Make sure to use the Event History tab to search on the AWS Console CloudTrail service page.

### Deleting the Stack

The S3 bucket deletion policy is set to retain, which attempts to preserve the S3 bucket while the other stack resources are deleted. Please be aware.

### Cross-Account Decrypt

If you are assuming a role defined in one account, to access another account, then it is not possible to access SSE-S3 objects.

Cross-account access needs to be explicitly configured (secure by default).
Since we assume a role to gain access to the to another account, any `s3:GetObject` requests will not have access to the default S3 server-side key (SSE-S3).
The only way you can use `kms:Decrypt` in another account is to:
1. add an S3 bucket policy to give permission to the other account for `S3:GetObject`, and
2. add a KMS policy on the key to give permission to the other account to `kms:Decrypt`

Since the AWS managed s3 key (default key) policy is immutable, then a user from another account assuming a role cannot access the decrypted objects.
When the Lambda function and the S3 bucket are in the same account, they can encrypt and secure the object from a user who assumed a role to gain access to secured account.

#### References:
- [Cross-Account Access Denied Error](https://aws.amazon.com/premiumsupport/knowledge-center/cross-account-access-denied-error-s3/)
- [Using SSE-KMS encryption for cross-account operations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-encryption.html)
- [Changing your Amazon S3 encryption from S3-Managed to AWS KMS](https://aws.amazon.com/blogs/storage/changing-your-amazon-s3-encryption-from-s3-managed-encryption-sse-s3-to-aws-key-management-service-sse-kms/)

### Miscellaneous Tips

- `Content-Type` and other headers must match the original get signed url request.
- The name of the file when uploaded does not determine the key of the S3 Object, only the key defined in the signed url request determines the S3 Object key.
- Must enable CORS on the API Gateway for browser HTTP requests to succeed, otherwise request pre-flight (OPTIONS) will cause the requests to fail.

### Other Resources

- https://aws.amazon.com/premiumsupport/knowledge-center/s3-403-forbidden-error/
- https://aws.amazon.com/premiumsupport/knowledge-center/lambda-delete-cloudformation-stack/
- https://fullstackdojo.medium.com/s3-upload-with-presigned-url-react-and-nodejs-b77f348d54cc
