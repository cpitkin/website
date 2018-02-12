---
title: "Automated Video Transcoding with OpenFaaS, Minio, and Nomad"
date: 2018-02-07
linktitle: Automated Video Transcoding
draft: true
---

If you want to know how to setup Nomad and OpenFaaS refer to my previous post [here](https://charliepitkin.com/post/setup-openfaas-nomad/).

## Services
- [Minio](https://minio.io/)

## Minio
Minio is an open source S3 compatable object storage service. There are several ways you can run Minio but for the sake of this write-up we are going to use the preinstalled version on FreeNAS 11. The configuration is the same across all installations, only the file location will differ.

### Setup Minio

1. First we need to SSH into our FreeNAS server and edit the configuration in `/usr/local/etc/minio`.

    `nano /usr/local/etc/minio/config.json`

    There are a few things we need to update in the configuration so I'll go through them one at a time.

    ```json
    {
    ...
      "credential": {
        "accessKey": "<access_key>",
        "secretKey": "<secret_key>"
      },
      "region": "<region_name>",
      ...
      "notify": {
        ...
        "webhook": {
          "1": {
            "enable": true,
            "endpoint": "<OpenFaaS_Function_URL>"
          }
        }
      }
    }
    ```
    **accessKey:** Random string used to identify a user
    **secretKey:** Random string used as a password
    **region:** This can be anything meaningful about where the Minio server is located on your network
    **endpoint:** The endpoint Minio will POST to when events happen in a bucket. We'll get into bucket events later.

2. Go to the FreeNAS UI under the Services section. Click on the wrench next to the S3 label. Here you will need to fill in the IP, port, disks, and if you want to enable the web UI.

    ![FreeNAS Minio Config UI](img/minio-settings.png)

3. Now we can start the service in FreeNAS.

    ![FreeNAS Minio Service UI](img/minio-service-start.png)

4. Create a bucket called transcode. You can do this through the UI or with the Minio client (mc) cli tool.

    `mc mb <minio_server_alias>/<bucket_name> --region <region>`

5. Now we need to create a bucket notification to trigger on object put events. Using the mc tool we can enter the following to set the event trigger.

    `mc events add <minio_server_alias>/<bucket_name> arn:minio:sqs:<region>:1:webhook --events put --suffix .mkv`

## Setup Docker volumes over NFS

I know this is not ideal and will be publishing an update in the near future to make improvements. For now you will need to mount 3 Docker volumes over NFS to use with the trancode container.

    - tv
    - movies
    - minio

## Setup OpenFaaS

We will build a function to talk to the Nomad API. This function's primary job is to schedule batch jobs in Nomad. This provides several benefits over using the async queue worker. You get resource monitoring and job queueing for free. Since Nomad, or any scheduler, won't overallocate a node's resources. This means we can safly schedule lots of jobs on a single node knowing they will be processed when resources are available. Since we want this to be efficient and as automated as possible we can upload as many videos to the bucket as we want and the transcode job will get scheduled.

Here is an example JSON object from a Minio put operation.

```json
{
  "EventType": "s3:ObjectCreated:Put",
  "Key": "example-bucket/file.jpg",
  "Records": [{
    "eventVersion": "2.0",
    "eventSource": "aws:s3",
    "awsRegion": "us-west-2",
    "eventTime": "2017-12-02T15:14:21Z",
    "eventName": "s3:ObjectCreated:Put",
    "userIdentity": {
      "principalId": "F8F7ot2RxqtTdpwF2DBq"
    },
    "requestParameters": {
      "sourceIPAddress": "192.168.1.1:59661"
    },
    "responseElements": {
      "x-amz-request-id": "14FC8361D03D09D6",
      "x-minio-origin-endpoint": "http://minio-server:9000"
    },
    "s3": {
      "s3SchemaVersion": "1.0",
      "configurationId": "Config",
      "bucket": {
        "name": "example-bucket",
        "ownerIdentity": {
          "principalId": "F8F7ot2RxqtTdpwF2DBq"
        },
        "arn": "arn:aws:s3:::images"
      },
      "object": {
        "key": "file.jpg",
        "size": 6166,
        "eTag": "e856d62fe0ab35a45c47b876af2415bc",
        "sequencer": "14FC8361D03D09D6"
      }
    }
  }],
  "level": "info",
  "msg": "",
  "time": "2017-12-02T09:14:21-06:00"
}
```

Using the above object we can get the needed data for scheduling a Nomad job.

The function code can be found [here](https://github.com/cpitkin/nomad-openfaas-transcode-video).

The actual transcoding is done by an awesome piece of software from [Don Melton](https://github.com/donmelton/video_transcoding). Huge kudos to him for building and maintaining such a great piece of software. All I did was package the software into a Docker container for portability and easy dependancy management. The container used byt the Nomad job can be found [here](https://hub.docker.com/r/cpitkin/video_transcode/).

## Uploading to Minio

At this point you should just need to start uploading code to Minio. I have a handy Nodejs script to do just that [here](https://github.com/cpitkin/video-upload/releases)