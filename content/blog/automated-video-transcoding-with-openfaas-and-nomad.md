---
title: "Automated Video Transcoding with OpenFaaS, Minio, and Nomad"
date: 2018-02-24
linktitle: Automated Video Transcoding
draft: false
---

If you want to know how to setup Nomad and OpenFaaS refer to my previous post [here](https://charliepitkin.com/post/setup-openfaas-nomad/).

## Services

- [Minio](https://minio.io/)
- Docker
- OpenFaaS
- Nomad

## Considerations

- Transcoding is a very CPU intensive task. Regardless of how you configure the setup, it is going to take significant CPU and disk i/o resources. That being said you can run this setup on pretty much any hardware but the more resources you give the transcode container the faster it will run. This setup has been tested using an Ubuntu Linux box with a 6 core 3.5 GHz AMD processor.

- We will be uploading raw video to one bucket (transcode) and placing the transcoded video in another bucket (complete). This means you will need enough disk space to accommodate at least your original file size*2. Ideally, for lots of files, you would want at least 500GB of free space.

### Setup

#### Minio

1. First, we need to login to the machine and create a couple of directories for us to use. We will need one to mount for the configs and one to mount for storage. Run the following as root to make the needed directories.

    `sudo mkdir {/mino,/transcode}`

2. Next, we are going to set up a Minio server on our compute node using Nomad. Then we get the benefits of having Nomad manage the resources for us. First, we need a Nomad job file. You need to copy the below job file and save it to minio.hcl

    ```hcl
      job "minio" {

      datacenters = [
        "dc1"
      ]

      priority = 10

      group "cluster" {
        count = 1

        restart {
          attempts = 3
          delay    = "5s"
          interval = "10m"
          mode     = "delay"
        }

        task "minio" {
          driver = "docker"
          config {
            image = "minio/minio",
            command = "server",
            args = [
              "/transcode"
            ],
            port_map {
              http = 9000
            }

            volumes = [
              "/transcode:/transcode",
              "/minio:/root/.minio"
            ]
          }

          resources {
            cpu = 500
            memory = 512
            network {
              port "http" {
                static = 9000
              }
            }
          }

          service {
            tags = [
              "transcode-minio"
            ]
            name = "transcode-minio"
            port = "http"

            check {
              type = "http"
              protocol = "http"
              method = "GET"
              port = "http"
              path = "/"
              interval = "10s"
              timeout  = "2s"
            }
          }
        }
      }
    }
    ```

3. Once the container starts for the first time you will need to get the access key and secret that were generated at startup. We need to login to the compute node and edit the config file and grab the access key and secret. The edits for the config are below.

    ```json
    {
      ...
      "credential": {
              "accessKey": "<random_string>",
              "secretKey": "<random_string>"
      },
      "region": "rapture",
      "browser": "on",
      "domain": "",
      "storageclass": {
              "standard": "",
              "rrs": ""
      },
      "notify": {
        ...
        "webhook": {
          "1": {
            "enable": true,
            "endpoint": "http://faas.rapture:8080/function/transcode-video"
          }
        }
        ...
      }
    }
    ```

    Copy:
      - accessKey
      - secretKey

    Edit:
      - webhook

4. Finally, we need to restart the job for the changes to take effect. Run the following in the same place you saved the Nomad job file.

    `nomad stop minio.hcl && nomad run minio.hcl`

#### OpenFaaS

We will build a function to talk to the Nomad API. This function's primary job is to schedule batch jobs in Nomad. Since Nomad, or any scheduler, won't over-allocate a node's resources. This means we can safely schedule lots of jobs across out client nodes knowing they will be processed when resources are available. Since we want this to be efficient and as automated as possible we can upload as many videos to the bucket as we want and the transcode job will get scheduled and processed when resources are available.

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

## Transcoding

The actual transcoding is done by an awesome piece of software from [Don Melton](https://github.com/donmelton/video_transcoding). Huge kudos to him for building and maintaining such a great piece of software. All I did was package the software into a Docker container for portability and easy dependency management. The container used by the Nomad job can be found [here](https://hub.docker.com/r/cpitkin/video_transcode/).

## Uploading to Minio

At this point, you should just need to start uploading code to Minio. I have a handy Go binary to do just that [here](https://github.com/cpitkin/video-upload/releases). Check out the README for details.