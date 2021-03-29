+++
date = "2017-07-30T17:00:00"
draft = false
tags = [AWS,Lambda@Edge,CloudFront,S3]
title = "Enterprise static site hosting in AWS"
math = false
image_preview = "uwa.jpg"
summary = "Building a highly available, secure static site delivery architecture in AWS"
+++

There are a ton of AWS static site guides out there, I know. What makes this one different? This guide is going to approach the problem with a more enterprise view than a lot of other guides. This means focusing on CLI, leveraging IAM, and making it highly available. The one thing I won't talk about which is important from an enterprise standpoint is logging. That deserves a post all to itself.

![Overall static site architecture](img/aws-static-site/architecture.png)

This is what we're building. I'll talk design decisions through the guide but I'll briefly talk some overall requirements that lead to this design. First is high availability, which we accomplish with Route 53 and CloudFront. Second is highly secure, so we heavily use IAM and other policies. A bonus for us on the security front is leveraging abstracted AWS services instead of EC2 or other services we must actively manage. This kind of architecture is also commonly referred to as serverless, but I am not a fan of the moniker. In this guide I am using the domain example.com as the site being delivered. I will also use {} to denote areas where you need to provide input; ARNs or other IDs will need to be filled in.

## Certificate
We're going to start with the easiest thing first, a new certificate we can use in CloudFront. We need to request this certificate in the us-east-1 region, which is the only region supported by CloudFront for certificates. We use the [aws acm request-certificate](https://docs.aws.amazon.com/cli/latest/reference/acm/request-certificate.html) command to accomplish this.

```
aws acm request-certificate --domain-name example.com --subject-alternative-names www.example.com  --validation-method email --region us-east-1
```

This gives us a certificate we can use for our delivery architecture and for a redirect from www.example.com to example.com. You can make this a separate certificate but I can't say I see any reason to. Make sure to copy the ARN as we will need it later!

Be warned, email validation hits you a whole bunch of times. They send a email to every contact in your whois record.

## S3 Storage
Now we're getting into the fun stuff. For this part we want two S3 buckets located in different regions. One bucket is going to be our primary bucket where we deploy our content to and the other bucket will be a secondary bucket that receives replicated changes from the master. This provides us a second origin we can use in the event of a S3 failure in our primary region. Replication requires we enable versioning, so we are also going to enable lifecycle rules to keep our costs down.

We're going to use several commands:
* [aws s3api create-bucket](https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html)
* [aws s3api put-bucket-versioning](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-versioning.html)
* [aws s3api put-bucket-replication](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-replication.html)
* [aws s3api put-bucket-lifecycle-configuration](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-lifecycle-configuration.html)
* [aws iam create-role](https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html)
* [aws iam put-role-policy](https://docs.aws.amazon.com/cli/latest/reference/iam/put-role-policy.html)

Start by creating the buckets and enabling versioning. Versioning is required for replication.
```
aws s3api create-bucket --bucket example.com-primary --acl private --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
aws s3api put-bucket-versioning --bucket example.com-primary --versioning-configuration Status=Enabled
aws s3api create-bucket --bucket example.com-secondary --acl private --region us-east-2 --create-bucket-configuration LocationConstraint=us-east-2
aws s3api put-bucket-versioning --bucket example.com-secondary --versioning-configuration Status=Enabled
```

With our buckets created we can now create a replication role with a policy limiting access to our two buckets. We first create the role and give S3 the permission to assume it. Then we'll create an inline policy for the role that gives S3 permissions so that replication can take place. I am choosing to use an inline policy because I want the 1:1 relationship between the role and policy. This isn't a policy you would reuse or should ever have to change.

We'll need a couple files before we can run the API commands. Create s3-assume-role.json with:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

And we'll also create replication-policy.json:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:Get*",
                "s3:ListBucket"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::example.com-primary",
                "arn:aws:s3:::example.com-primary/*"
            ]
        },
        {
            "Action": [
                "s3:ReplicateObject",
                "s3:ReplicateDelete",
                "s3:ReplicateTags",
                "s3:GetObjectVersionTagging"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:s3:::example.com-secondary/*"
        }
    ]
}
```

We'll create the role and give it an inline policy. Copy the ARN because we need it for the next set of commands!
```
aws iam create-role --role-name Replicate-Example-Primary-To-Secondary --assume-role-policy-document file://s3-assume-role.json
aws iam put-role-policy --role-name Replicate-Example-Primary-To-Secondary --policy-name Allow-Replication --policy-document replication-policy.json
```

All that is left to do is configure our replication and to create lifecycle policies to clean up old objects. Take the ARN of the role we just created and put in in replication.json:
```
{
    "Rules": [
        {
            "Filter": {},
            "ID": "Replicate to Secondary",
            "DeleteMarkerReplication": {
                "Status": "Disabled"
            },
            "Status": "Enabled",
            "Destination": {
                "Bucket": "arn:aws:s3:::example.com-secondary"
            },
            "Priority": 1
        }
    ],
    "Role": "arn:aws:iam::{whatever your ARN is}"
}
```

We also need a lifecycle.json file:
```
{
    "Rules": [
        {
            "NoncurrentVersionExpiration": {
                "NoncurrentDays": 30
            },
            "AbortIncompleteMultipartUpload": {
                "DaysAfterInitiation": 7
            },
            "Expiration": {
                "ExpiredObjectDeleteMarker": true
            },
            "ID": "Limit Versions",
            "NoncurrentVersionTransitions": [
                {
                    "StorageClass": "GLACIER",
                    "NoncurrentDays": 1
                }
            ],
            "Status": "Enabled",
            "Filter": {
                "Prefix": ""
            }
        }
    ]
}
```

The replication policy will replicate all changes from our primary bucket, [except deletions](https://docs.aws.amazon.com/AmazonS3/latest/dev/crr-what-is-isnot-replicated.html), to our secondary bucket. Every time we create a new version of an object the lifecycle policy gets involved. When you update an object you have a current and one or more noncurrent versions. This lifecycle policy transitions any version that is noncurrent for longer than 1 day to glacier class storage which is less than 1/5th the price of standard S3 storage. For small buckets this isn't a huge difference but if you have multiple terabytes worth of data it starts to matter. The lifecycle policy will then delete any noncurrent versions that are older than 30 days which is the minimum amount of time you have to store objects in Glacier class storage for.
```
aws s3api put-bucket-replication --bucket example.com-primary --replication-configuration file://replication.json
aws s3api put-bucket-lifecycle-configuration --bucket example.com-primary --lifecycle-configuration  file://lifecycle.json
aws s3api put-bucket-lifecycle-configuration --bucket example.com-secondary --lifecycle-configuration  file://lifecycle.json
```

Our buckets and the replication between them are now fully configured. It's time to add the next layer and get our Lambda@Edge functions ready.

## Lambda@Edge

Using Lambda we can modify incoming requests and our outgoing responses. Since we are not using the S3 website hosting feature there is nothing to set a default document. We can also use Lambda for any other rewrites or redirects we need. There are also a [variety of headers](https://www.owasp.org/index.php/OWASP_Secure_Headers_Project#tab=Headers) we want added to our response that wouldn't be included without using Lambda@Edge. The first Lambda function we create will operate on the [viewer request](https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html), which is after CloudFront gets the request but before it hits the cache. This increases the executions of our script but lets us apply "corrections" earlier. If we need to do any fancy footwork when the request goes to the origin we've still left the origin request slot open and because we've already cleaned up the incoming requests any function we put there doesn't have to pull double duty. Our second Lambda function will be used on the origin response because we want our modifications to be cached by CloudFront with the item to reduce the number of executions required.

These commands are going to be used in this section:
* [aws lambda create-function](https://docs.aws.amazon.com/cli/latest/reference/lambda/create-function.html)
* [aws iam create-role](https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html)
* [aws iam create-policy](https://docs.aws.amazon.com/cli/latest/reference/iam/create-policy.html)
* [aws iam attach-role-policy](https://docs.aws.amazon.com/cli/latest/reference/iam/attach-role-policy.html)


Just like earlier with our S3 replication we need to create a new role that can be assumed by the Lambda service. Create lambda-assume-role.json with:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "edgelambda.amazonaws.com",
          "lambda.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

We also need a basic policy to allow writing logs to CloudWatch so create lambda-cloudwatch-policy.json:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:*:*:*"
      ]
    }
  ]
}
```

We also need our two Lambda functions. We'll need to take a couple steps here to package something that AWS will accept. Our code will go into an index.js file and then we'll zip that file into FunctionName.zip.

Our first index.js file which should be zipped into RewriteAndRedirect.zip:
```
'use strict';
const { STATUS_CODES } = require('http')
exports.handler = (event, context, callback) => {
    const request = event.Records[0].cf.request;
    return callback(null, request);
};

function redirect (to) {
  return {
    status: '301',
    statusDescription: STATUS_CODES['301'],
    headers: {
      location: [{ key: 'Location', value: to }]
    }
  }
}
```

And our second index.js file that we will zip into SetHeaders.zip, with whatever headers we decide to include:
```
'use strict';
exports.handler = (event, context, callback) => {
    const response = event.Records[0].cf.response;
    const headers = response.headers;
    
    headers['strict-transport-security'] = [{
        key:   'strict-transport-security', 
        value: "max-age=31536000; includeSubdomains; preload"
    }];

    headers['content-security-policy'] = [{
        key:   'Content-Security-Policy', 
        value: "default-src 'none'; img-src 'self'; script-src 'self'; style-src 'self'; object-src 'none'"
    }];

    headers['x-content-type-options'] = [{
        key:   'x-content-type-options',
        value: "nosniff"
    }];
    
    headers['x-frame-options'] = [{
        key:   'x-frame-options',
        value: "deny"
    }];

    headers['x-xss-protection'] = [{
        key:   'x-xss-protection',
        value: "1; mode=block"
    }];


    headers['referrer-policy'] = [{
        key:   'referrer-policy',
        value: "same-origin"
    }];
    
    callback(null, response);
};
```

We'll start by prepping the role we need. Make sure to copy the ARN of the new role and the policy.
```
aws iam create-role --role-name Lambda@Edge --assume-role-policy-document file://lambda-assume-role.json
aws iam create-policy --policy-name Lambda-CloudWatch --policy-document file://lambda-cloudwatch-policy.json
aws iam attach-role-policy --role-name Lambda@Edge --policy-arn {ARN from previous command}
```

Now we can create our two brand new Lambda functions. Like our certificate earlier, these need to be created in the us-east-1 region. If you put them in any other region you will not be able to use them for Lambda@Edge.
```
aws lambda create-function --function-name RewriteAndRedirect --runtime nodejs8.10 --role {ARN for the Lambda@Edge role} --handler index.handler --zip-file file://RewriteAndRedirect.zip --region us-east-1
aws lambda create-function --function-name SetHeaders --runtime nodejs8.10 --role {ARN for the Lambda@Edge role} --handler index.handler --zip-file file://SetHeaders.zip --region us-east-1
```



## CloudFront Distribution

CloudFront is going to accomplish several things for us. First is of course it will be our CDN and cache our site across the world providing for speedy access to our visitors. The second is that we can use the Origin Access Identity feature to force all access to our bucket to go through CloudFront. Recall that we never allowed public access to our buckets when we created them. Finally, Amazon added a feature to create highly available origin groups that allows us to automatically failover on error.

We only need two commands to cover this part:
* [aws cloudfront create-cloud-front-origin-access-identity](https://docs.aws.amazon.com/cli/latest/reference/cloudfront/create-cloud-front-origin-access-identity.html)
* [aws cloudfront create-distribution](https://docs.aws.amazon.com/cli/latest/reference/cloudfront/create-cloud-front-origin-access-identity.html)

Start by creating a new Origin Access Identity and then we'll use that identity to create a new distribution.
```
aws cloudfront create-cloud-front-origin-access-identity --cloud-front-origin-access-identity-config CallerReference=`date +%s`,Comment="Example.com"
```

We need to put together a config file for our distribution. Make sure you have the ARN for the certificate you created earlier, the ID of your Origin Access Identity, and a Unix timestamp. Use these values to create distribution.json:
```
{
    "OriginGroups": {
        "Quantity": 1,
        "Items": [
            {
                "Members": {
                    "Quantity": 2,
                    "Items": [
                        {
                            "OriginId": "S3-example.com-primary"
                        },
                        {
                            "OriginId": "S3-example.com-secondary"
                        }
                    ]
                },
                "FailoverCriteria": {
                    "StatusCodes": {
                        "Quantity": 4,
                        "Items": [
                            500,
                            502,
                            503,
                            504
                        ]
                    }
                },
                "Id": "OriginGroup-1-S3-example.com-primary"
            }
        ]
    },
    "DefaultRootObject": "",
    "CustomErrorResponses": {
        "Quantity": 0
    },
    "WebACLId": "",
    "ViewerCertificate": {
        "SSLSupportMethod": "sni-only",
        "MinimumProtocolVersion": "TLSv1.2_2018",
        "ACMCertificateArn": "arn:aws:acm:us-east-1:{whatever your ARN is}",
        "Certificate": "arn:aws:acm:us-east-1:{whatever your ARN is}",
        "CertificateSource": "acm"
    },
    "Comment": "",
    "PriceClass": "PriceClass_100",
    "Aliases": {
        "Quantity": 0
    },
    "Origins": {
        "Quantity": 2,
        "Items": [
            {
                "DomainName": "example.com-primary.s3.amazonaws.com",
                "CustomHeaders": {
                    "Quantity": 0
                },
                "OriginPath": "",
                "S3OriginConfig": {
                    "OriginAccessIdentity": "origin-access-identity/cloudfront/{Your Origin Access Identity ID}"
                },
                "Id": "S3-example.com-primary"
            },
            {
                "DomainName": "example.com-secondary.s3.amazonaws.com",
                "CustomHeaders": {
                    "Quantity": 0
                },
                "OriginPath": "",
                "S3OriginConfig": {
                    "OriginAccessIdentity": "origin-access-identity/cloudfront/{Your Origin Access Identity ID}"
                },
                "Id": "S3-example.com-secondary"
            }
        ]
    },
    "Enabled": false,
    "HttpVersion": "http2",
    "IsIPV6Enabled": true,
    "CallerReference": "{Unix timestamp}",
    "DefaultCacheBehavior": {
        "TargetOriginId": "S3-example.com-primary",
        "ForwardedValues": {
            "QueryStringCacheKeys": {
                "Quantity": 0
            },
            "QueryString": false,
            "Cookies": {
                "Forward": "none"
            },
            "Headers": {
                "Quantity": 0
            }
        },
        "TrustedSigners": {
            "Quantity": 0,
            "Enabled": false
        },
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": {
            "CachedMethods": {
                "Quantity": 2,
                "Items": [
                    "HEAD",
                    "GET"
                ]
            },
            "Quantity": 2,
            "Items": [
                "HEAD",
                "GET"
            ]
        },
        "FieldLevelEncryptionId": "",
        "Compress": true,
        "LambdaFunctionAssociations": {
            "Items": [
                {
                    "EventType": "origin-response",
                    "IncludeBody": false,
                    "LambdaFunctionARN": "arn:aws:lambda:us-east-1:{your header Lambda ARN here}"
                },
                {
                    "EventType": "viewer-request",
                    "IncludeBody": false,
                    "LambdaFunctionARN": "arn:aws:lambda:us-east-1:{your rewrite/redirect Lambda ARN here}"
                }

            ],
            "Quantity": 2
        },
        "MaxTTL": 31536000,
        "MinTTL": 0,
        "SmoothStreaming": false,
        "DefaultTTL": 86400
    },
    "Restrictions": {
        "GeoRestriction": {
            "Quantity": 0,
            "RestrictionType": "none"
        }
    },
    "Logging": {
        "Prefix": "",
        "IncludeCookies": false,
        "Bucket": "",
        "Enabled": false
    }
}
```

The config covers quite a bit, but I'll discuss the parts important for this guide. We create an origin group that contains our two S3 buckets. Failover rules for the group are based on HTTP responses received from the origin. I include only 500 type errors to cover S3 service failure, but you can include 400 type erors as well. The origins themselves have also been configured to use the Origin Access Identity we created earlier, which is required so that CloudFront can get files from our buckets. When it comes to encyption the sni-only option is used because we're not going to support older systems that don't support SNI which would cost $600/mo extra. Simnilarly, we're going to be agressive with protol and ciphers so the [TLSv1.2_2018](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/secure-connections-supported-viewer-protocols-ciphers.html#secure-connections-supported-ciphers) option is chosen that limits connections to TLS v1.2 and strong ciphers only. The [PriceClass_100](https://aws.amazon.com/cloudfront/pricing/) focuses on North America and Europe and is chosen for cost savings. If you have significant traffic from Asia or Australia you may want to choose a different price class. For the HTTP version we specify http2 to take advantage of all the speed benefits provided. The config also forces a HTTP to HTTPS redirect which is required to enable HSTS Preload. We're also limiting HTTP methods to GET and HEAD because all we're doing is serving static content. Similarly, since we're just serving static content, we're going to enable compression. There are security concerns with compression and TLS but we're not serving anything sensitive. Depending on your security needs you may choose to disable compression instead.

Creating the distribution is easy enough.
```
aws cloudfront create-distribution --distribution-config file://distribution.json
```

## S3 bucket policy

Quick return to S3. We need to use the [aws s3api put-bucket-policy](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-policy.html) to set policies that allow replication and access from CloudFront useing our Origin Access Identity. You'll need the unique IDs from your Origin Access Identity as well as the IAM role we created for replication.

First config we need is primary-bucket-policy.json
```
{
    "Version": "2008-10-17",
    "Id": "CloudFrontAccess",
    "Statement": [
        {
            "Sid": "CloudFrontPolicy",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity {Your Origin Access Identity ID}"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::example.com-primary/*"
        }
    ]
}
```

And of course secondary-bucket-policy.json:
```
{
    "Version": "2008-10-17",
    "Id": "CloudFrontAccessAndReplication",
    "Statement": [
        {
            "Sid": "S3ReplicationPolicy",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::{ID from your replication role ARN}:root"
            },
            "Action": [
                "s3:GetBucketVersioning",
                "s3:PutBucketVersioning",
                "s3:ReplicateObject",
                "s3:ReplicateDelete"
            ],
            "Resource": [
                "arn:aws:s3:::example.com-secondary",
                "arn:aws:s3:::example.com-secondary/*"
            ]
        },
        {
            "Sid": "CloudFrontPolicy",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity {Your Origin Access Identity ID}"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::example.com-secondary/*"
        }
    ]
}
```

So why does the policy for our secondary bucket need to include a policy statement for replication but our primary bucket doesn't? This is due to how replication works, with S3 assuming the role we created when replication is triggered at our primary bucket. The role needs the access we configured for the primary bucket but there is no need to also include that access in the bucket policy.

Go ahead and set the policies we just created.
```
aws s3api put-bucket-policy --bucket example.com-primary --policy file://primary-bucket-policy.json
aws s3api put-bucket-policy --bucket example.com-secondary --policy file://secondary-bucket-policy.json
```
