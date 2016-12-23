---
title: Rekognize your serverless photo album
date: 2016-12-12 13:57:50
foreword: Add facial recognition to your online photo album using AWS Rekognition, Lambda, Cognito, DynamoDB, S3, and Route53.
tags: aws, rekognition, photo, album, lambda, cognito, facebook, s3, route53, dynamodb, serverless
---
In this blog post I explain how I used AWS to build a serverless photo album with facial recognition.  This is not a step-by-step tutorial but rather an overview of the architecture, setup, and code.

I started my online photo album, finpics.com, back in 1999.  I originally built finpics.com using php and mysql.
I also had a custom Java Applet to upload the images.  Pretty cool, right.

Code: [https://github.com/rgfindl/finpics](https://github.com/rgfindl/finpics)
Website: [http://finpics.com](http://finpics.com)

<img src="/images/finpics-03.png" width="100%" style="border: solid 1px #eee; padding: 5px;"/>

### Serverless

Serverless architectures refer to applications that significantly depend on third-party services (knows as Backend as a Service or "BaaS") or on custom code that's run in ephemeral containers (Function as a Service or "FaaS"), the best known vendor host of which currently is AWS Lambda. [http://martinfowler.com/articles/serverless.html](http://martinfowler.com/articles/serverless.html)

### finpics.com

finpics.com gets almost no traffic.  Good thing too because there are a lot of embarrassing picture of myself, family, and friends. A serverless architectures is more cost effective because I'm not paying for a server to run that gets very little traffic. There is also very little maintenance with serverless apps.

Once AWS Rekognition was released I knew that I wanted to use it to improve the searchability of finpics.com.  It is great to click on someones face and see more pictures of them.

### AWS resources

|AWS Resource|Usage|
|--------|-----|
|Route53|DNS for finpics.com, points to static S3 bucket.|
|S3|Hosts static website and images.|
|Cognito|Authorization. Lets the web client assume an IAM Role to make calls to AWS resources.|
|DynamoDB|Store image metadata.|
|Lambda|Serverless compute.|
|Rekognition|Facial recognition.|

## Architecture
### Web Requests
<img src="/images/finpics-01.jpg" width="100%" style="border: solid 1px #eee; padding: 5px;"/>
**Static Web Site**
Fetch the html, js, css, and image assets directly from S3.

**Get Unauth Creds**
Make a call to Cognito to get AWS credentials to use for all the calls to AWS.  The user assumes the unauthenticated IAM role that you define in Cognito.

**Fetch Pictures**
The pictures are structured into picture sets.  A call is made to DynamoDB to get all the picture sets, and the featured pic for each set.  Then another call is made to DynamoDB to get the pictures for each set.

**Search Faces**
Search Rekognition given a face id.  When you click on a persons face the results are pictures with that face sorted by highest probability.

### Upload Images
Images are uploaded directly to S3.  There is an S3 event that triggers a Lambda function to perform the following tasks on each new image:
* Create a thumbnail
* Index the faces with Rekognition
* Store metadata in DynamoDB

I have a [script](https://github.com/rgfindl/finpics/blob/master/util/upload-pics.js) that uploads a new picture set.
<img src="/images/finpics-02.jpg" width="100%" style="border: solid 1px #eee; padding: 5px;"/>

## DynamoDB Tables
### pics table
* primaykey (Primary Key)
* sortKey (Sort Key)
* data (Rekognition IndexFaces response)

The pics table stores all the picture sets, with featured image, which is used by the [index](http://finpics.com) page.

**Picsets**
* primaykey: '/' (Primary Key)
* sortkey: '014_newportboston' (Sort Key)
* pic: 'Newport_pic_3.jpg'

The pics table also stores all the pictures associated with a picture set, which is used by each [picture set](http://finpics.com/#!/picset?path=280_pic_set_282) page.
**Pics**
* primaykey: '014_newportboston' (Primary Key)
* sortkey: 'Newport_pic_3.jpg' (Sort Key)
* ... Rekognition IndexFaces response

### pics_by_image_id table
* image_id (Primary Key)
* data (Rekognition IndexFaces response)
* image_path

The pics_by_image_id table stores the same facial recognition data as the pics table but is index by the AWS Rekognition image_id.  When we search Rekognition for facial matches we use the image_id's in the response to fetch the picture information via this table.

## Setup & Code Samples
### AWS S3
I'm using 3 buckets for finpics.com.
* **finpics.com** which serves the static web content.
* **finpics-pics** which serves the original (large) images.
* **finpics-thumbs** which serves the thumbnails.

I used different buckets for 2 reasons:
1. AWS Rekognition fails when your bucket name has a period in it.  Awesome!
2. For every new picture added to **finpics-pics** a Lambda function is triggered to create the thumbnail.  I didn't want to create a circular loop.

For each bucket I have **Web Hosting** enabled.

I want all assets in these 3 buckets to be publicly available.  Under **Permissions** I have the following **bucket policy** for each.  Make sure to change the Resource name to match your bucket name.
```
{
	"Version": "2008-10-17",
	"Statement": [
		{
			"Sid": "PublicReadForGetBucketObjects",
			"Effect": "Allow",
			"Principal": {
				"AWS": "*"
			},
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::finpics-pics/*"
		}
	]
}
```

Now all my web assets are publicly available and served via S3.  Cheap, serverless, and performant.

### AWS Cognito
Cognito allows the client-side JavaScript code permission to access AWS resources like DynamoDD & Lambda functions.

I created a Cognito **Federated Identity** called `finpics`.  When creating this identity I **Enabled access to unauthenticated identities**.  I also created new Unauthenticated and Authenticated AWS IAM Roles.  I'll explain the permissions needed within Unauthenticated role later.  I'm currently not using the Authenticated role.

<img src="/images/finpics-04.png" style="border: solid 1px #eee; padding: 5px;"/>

In the client-side JavaScript code I assume the Unathenticated role like this.  All subsequent calls to AWS will use this IAM role.
```
// Initialize the Amazon Cognito credentials provider
AWS.config.region = 'us-east-1'; // Region
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
    IdentityPoolId: 'your-id'
});
```

### IAM Roles
#### Cognito Unathenticated Role
Cognito will automatically create the Unathenticated role and setup the **Trust Relationship** with the Cognito federated identity.

Here are the permissions:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "mobileanalytics:PutEvents",
                "cognito-sync:*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "Stmt1481636027000",
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:GetRecords",
                "dynamodb:Query"
            ],
            "Resource": [
                "arn:aws:dynamodb:us-east-1:132093761664:table/pics"
            ]
        },
        {
            "Sid": "Stmt1481853728000",
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction"
            ],
            "Resource": [
                "arn:aws:lambda:us-east-1:132093761664:function:finpics-dev-search"
            ]
        }
    ]
}```

1. The first is permission is sync the user with Cognito (Cognito stuff).
2. The second permission is to get picture data and metadata from DynamoDB.
3. The third permission is to invoke our Lambda function.

#### Lambda Role
The Lambda role has permissions to interact with the S3 buckets, DynamoDB tables, and the Rekognition collection (which I will create next).  Lambda also has permission to CloudWatch logs.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1481315917000",
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::finpics-pics/*",
                "arn:aws:s3:::finpics-thumbs/*"
            ]
        },
        {
            "Sid": "Stmt1481636027000",
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:GetRecords",
                "dynamodb:Query",
                "dynamodb:BatchGetItem",
                "dynamodb:PutItem",
                "dynamodb:UpdateItem"
            ],
            "Resource": [
                "arn:aws:dynamodb:us-east-1:132093761664:table/pics",
                "arn:aws:dynamodb:us-east-1:132093761664:table/pics_by_image_id"
            ]
        },
        {
            "Sid": "Stmt1481823471000",
            "Effect": "Allow",
            "Action": [
                "rekognition:CompareFaces",
                "rekognition:ListFaces",
                "rekognition:SearchFaces",
                "rekognition:SearchFacesByImage",
                "rekognition:IndexFaces"
            ],
            "Resource": [
                "arn:aws:rekognition:us-east-1:132093761664:collection/finpics/person/*"
            ]
        },
        {
            "Sid": "Stmt1482175623000",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:us-east-1:132093761664:log-group:/aws/lambda/*:*:*"
            ]
        }
    ]
}
```

### Rekognition
Amazon Rekognition is a service that makes it easy to add image analysis to your applications. With Rekognition, you can detect objects, scenes, and faces in images. You can also search and compare faces. Rekognition’s API enables you to quickly add sophisticated deep learning-based visual search and image classification to your applications.

To start indexing faces we first need to create a Rekognition collection.
```
var AWS = require('aws-sdk');
var rekognition = new AWS.Rekognition({apiVersion: '2016-06-27'});

var params = {
    CollectionId: 'finpics' /* required */
};
rekognition.createCollection(params, function(err, data) {
    if (err) console.log(err, err.stack); // an error occurred
    else     console.log(data);           // successful response
});
```

When an image is added to S3 our Lambda function is triggered.  The Lambda function indexes the image which adds all the faces within that image to the Rekognition collection.

```
var params = {
    CollectionId: 'finpics', /* required */
    Image: { /* required */
        S3Object: {
            Bucket: 'finpics-pics',
            Name: key
        }
    }
};
rekognition.indexFaces(params, function(err, data) {
    if (err)  winston.error(err);
    callback(err, data);
});
```

When a user clicks on a face we search the Rekognition collection to find matches ordered by match probability.
```
var AWS = require('aws-sdk');
var rekognition = new AWS.Rekognition({apiVersion: '2016-06-27'});

var params = {
    CollectionId: 'finpics', /* required */
    FaceId: faceid
};
rekognition.searchFaces(params, function (err, data) {
    if (err)  winston.error(err);
    callback(err, data);
});
```

### Lambda Functions
There are two Lambda functions:
1. Search - search the Rekognition facial collection given a `faceid`.
2. Process New Image - Triggered for each new image added to S3.
 * Creates a thumbnail
 * Indexes the image within the Rekognition collection
 * Adds image and metadata to DynamoDB

[Lambda functions](https://github.com/rgfindl/finpics/blob/master/lambda/index.js)

#### Search
1. Search the Rekognition collection.
2. Bulk fetch the images from DynamoDB.
3. Normalize the DynamoDB results.
```
search: function(event, context, callback) {
    //
    // Search AWS Rekognition given the faceid.
    //
    var params = {
      CollectionId: 'finpics', /* required */
      FaceId: event.faceid
    };
    rekognition.searchFaces(params, function (err, data) {
      if (err) {
        var response = {
          statusCode: 500,
          err: err,
          params: params
        };
        callback(null, response);
      } else {
        //
        // For each face match.  Fetch the image information from DynamoDB.
        //
        var keys = [];
        var imageids = [];
        _.forEach(data.FaceMatches, function (FaceMatch) {
          if (!_.includes(imageids, FaceMatch.Face.ImageId)) {
            keys.push({"image_id": {"S": FaceMatch.Face.ImageId}});
            imageids.push(FaceMatch.Face.ImageId);
          }
        });
        var params = {
          "RequestItems": {
            "pics_by_image_id": {
              "Keys": _.slice(keys, 0, 100)
            }
          }
        };
        dynamodb.batchGetItem(params, function (err, results) {
          if (err) {
            var response = {
              statusCode: 500,
              err: err,
              params: params
            };
            callback(null, response);
          } else {
            //
            // Normalize the images we get back from the DynamoDB bulk get request.
            //
            var output = [];
            var imageids = [];
            _.forEach(data.FaceMatches, function (FaceMatch) {
              if (!_.includes(imageids, FaceMatch.Face.ImageId)) {
                var raw_item = _.find(results.Responses.pics_by_image_id, {image_id: {S: FaceMatch.Face.ImageId}});
                if (!_.isNil(raw_item)) {
                  var item = {
                    image_id: raw_item.image_id.S,
                    image_path: raw_item.image_path.S
                  };
                  var faces = [];
                  _.forEach(raw_item.data.M.FaceRecords.L, function (FaceRecord) {
                    faces.push({
                      Face: {
                        Confidence: FaceRecord.M.Face.M.Confidence.N,
                        ImageId: FaceRecord.M.Face.M.ImageId.S,
                        BoundingBox: {
                          Top: FaceRecord.M.Face.M.BoundingBox.M.Top.N,
                          Height: FaceRecord.M.Face.M.BoundingBox.M.Height.N,
                          Width: FaceRecord.M.Face.M.BoundingBox.M.Width.N,
                          Left: FaceRecord.M.Face.M.BoundingBox.M.Left.N
                        },
                        FaceId: FaceRecord.M.Face.M.FaceId.S,
                      }
                    });
                  });
                  item.data = {
                    FaceRecords: faces
                  };
                  output.push(item);
                }
                imageids.push(FaceMatch.Face.ImageId);
              }
            });
            var response = {
              statusCode: 200,
              output: output
            };
            callback(null, response);
          }
        });
      }
    });
}
```

#### Process New Image
1. Download image from S3 (finpics-pics)
2. Create thumbnail
3. Upload thumbnail to S3 (finpics-thumbs)
4. Add feature picture for album, if needed
5. Index image using Rekognition
6. Add image and metadata to DynamoDB

```
  s3: function(event, context, callback) {

    winston.info("Reading options from event:\n", util.inspect(event, {depth: 5}));
    var srcBucket = event.Records[0].s3.bucket.name;
    // Object key may have spaces or unicode non-ASCII characters.
    var srcKey    =
        decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, " "));
    var dstBucket = S3_THUMBS_BUCKET;
    var dstKey    = srcKey;

    // Sanity check: validate that source and destination are different buckets.
    if (srcBucket == dstBucket) {
      callback("Source and destination buckets are the same.");
      return;
    }

    // Infer the image type.
    var typeMatch = srcKey.match(/\.([^.]*)$/);
    if (!typeMatch) {
      callback("Could not determine the image type.");
      return;
    }
    var imageType = _.toLower(typeMatch[1]);
    if (imageType != "jpg" && imageType != "jpeg" && imageType != "png") {
      callback('Unsupported image type: ${imageType}');
      return;
    }

    // Download the image from S3, transform, and upload to a different S3 bucket.
    async.waterfall([
          function download(next) {
            // Download the image from S3 into a buffer.
            s3.getObject({
                  Bucket: srcBucket,
                  Key: srcKey
                },
                next);
          },
          function transform(response, next) {
            gm(response.Body).size(function(err, size) {
              // Infer the scaling factor to avoid stretching the image unnaturally.
              var scalingFactor = Math.min(
                  MAX_WIDTH / size.width,
                  MAX_HEIGHT / size.height
              );
              var width  = scalingFactor * size.width;
              var height = scalingFactor * size.height;

              // Transform the image buffer in memory.
              this.resize(width, height).autoOrient()
                  .toBuffer(imageType, function(err, buffer) {
                    if (err) {
                      next(err);
                    } else {
                      next(null, response.ContentType, buffer);
                    }
                  });
            });
          },
          function upload(contentType, data, next) {
            // Stream the transformed image to a different S3 bucket.
            s3.putObject({
                  Bucket: dstBucket,
                  Key: dstKey,
                  Body: data,
                  ContentType: contentType,
                  StorageClass: 'REDUCED_REDUNDANCY'
                },
                next);
          },
          function add_feature_pic(response, next) {
            var image_parts = _.drop(_.split(srcKey, '/'));
            var params = {
              TableName: 'pics',
              Key: {
                primarykey: '/',
                sortkey: _.head(image_parts)
              },
              UpdateExpression: "set pic = :pic",
              ConditionExpression: "attribute_not_exists(pic)",
              ExpressionAttributeValues:{
                ':pic': _.last(image_parts)
              }
            };
            winston.info(JSON.stringify(params));
            docClient.update(params, function(err, results) {
              if (err && _.isEqual(err.code, 'ConditionalCheckFailedException')) next(null, null);
              else next(err, results);
            });
          },
          function rekognize(response, next) {
            var params = {
              CollectionId: COLLECTION_ID, /* required */
              Image: { /* required */
                S3Object: {
                  Bucket: srcBucket,
                  Name: srcKey
                }
              }
            };
            winston.info('Index faces');
            winston.info(JSON.stringify(params));
            rekognition.indexFaces(params, next);
          },
          function add_pics(data, next) {
            var image_parts = _.drop(_.split(srcKey, '/'));
            var item = {
              primarykey: _.head(image_parts),
              sortkey: _.nth(image_parts, 1),
              data: data
            };
            var params = {
              TableName: 'pics',
              Item: item
            };
            winston.info('Put DynamoDB');
            winston.info(JSON.stringify(params));
            docClient.put(params, function(err, respose) {
              if (err)  winston.error(err);
              next(err, data);
            });
          },
          function add_pics_by_image_id(data, next) {
            if (!_.isNil(data) && !_.isNil(data.FaceRecords) && !_.isEmpty(data.FaceRecords) &&
                !_.isNil(data.FaceRecords[0].Face) && !_.isNil(data.FaceRecords[0].Face.ImageId)) {
              var image_parts = _.drop(_.split(srcKey, '/'));
              var item = {
                image_id: data.FaceRecords[0].Face.ImageId,
                data: data,
                image_path: _.join(image_parts, '/')
              };
              var params = {
                TableName: 'pics_by_image_id',
                Item: item
              };
              winston.info('Put DynamoDB');
              winston.info(JSON.stringify(params));
              docClient.put(params, next);
            } else next(null, null);
          }
        ], function (err) {
          if (err) {
            winston.error(
                'Unable to resize ' + srcBucket + '/' + srcKey +
                ' and upload to ' + dstBucket + '/' + dstKey +
                ' due to an error: ' + err
            );
          } else {
            winston.info(
                'Successfully resized ' + srcBucket + '/' + srcKey +
                ' and uploaded to ' + dstBucket + '/' + dstKey
            );
          }

          callback(null, "message");
        }
    );
  }
```

Thanks for reading my blog post.  Please let me know if you have any questions.

Here is the [source code](https://github.com/rgfindl/finpics).
Here is the [demo (finpics.com)](http://finpics.com).

There are a bunch of untilities [here](https://github.com/rgfindl/finpics/blob/master/util).  I had a lot of images already that I had to process using [this script](https://github.com/rgfindl/finpics/blob/master/util/process-existing-images.js).