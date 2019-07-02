+++
date = "2018-06-09"
title = "Automated Video Transcoding with OpenFaaS"
description = "Transcoding video using OpenFaaS functions as a long running job"
images = ["posts/quinten-de-graaf-L4gN0aeaPY4-unsplash.jpg"]
tags = ["open source", "openfaas", "nomad", "minio", "automation", "docker"]
categories = ["tutorial"]
+++

# Purpose

I have a very large movie collection and use Plex to keep it organized and easy to manage. That means a lot of storage space is needed for movie files, especially when they are Blue-ray. To save space on disk while maintaining the good quality I built this transcode pipeline to automate the process of adding a new movie to the library. You can find the repo [here](https://github.com/cpitkin/openfaas-transcode)

## Uploader

The companion uploader executable can be found [here](https://github.com/cpitkin/video-upload/)

# Architecture

![architecuture diagram](/img/tp.png)

## Considerations

- Transcoding is a very CPU intensive task. Regardless of how you configure the setup, it is going to take significant CPU and disk i/o resources. That being said you can run this setup on pretty much any hardware but the more resources you give the transcode container the faster it will run. This setup has been tested using an Ubuntu Linux box with an AMD FX-6300 Six-Core Processor at 3.5 GHz and 16 GB of RAM.

- We will be uploading raw video to one bucket (transcode) and placing the transcoded video in another bucket (complete). This means you will need enough disk space to accommodate at least your original file size*2. Since you want to upload files continuously you will need enough space to hold the raw files plus the smaller files until the move function can put them in the correct place.

    Example: Blue-rays are around 25 GB in size but the transcoded version will be in the 10-12 GB range. That means, you will need roughly 600 GB to hold them all until clean up happens. `(25 GB * 15 raw files) + (12 GB * 15 transcoded files) = 555 GB`

### Setup

### Prerequisites

You will need a one or two Minio servers setup. In my case, I use two located on two different physical servers but you can easily do it with one. Since it is a bit out of the scope of this post I'm not going to go into detail but [here](https://docs.minio.io/docs/minio-docker-quickstart-guide) is a link to the Docker quickstart guide. 

Buckets:

- transcode
- complete
- movies
- tv

#### Minio config

You should set the Access Key, Secret Key, and webhook values before deploying the container. That will save you the hassle of getting the generated keys and setting the webhook later and needing to restart the container. An example config is below:

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
        "endpoint": "http://<IP or URL>:8080/function/transcode-entrypoint"
      }
    }
    ...
  }
}
```

**NOTE:** If you're using the uploader executable mentioned above then the PutObject event trigger will be set for you. If you will need to manually set a PutObject event trigger for the webhook on the transcode bucket.

## OpenFaaS Functions (in order)

### transcode-entrypoint

This is the entry point into the pipeline that is used to start the process. You can also add things like Slack integration calls here.

### transcode-worker

This is the main worker for the pipeline. The worker is called using the `/async-function` endpoint. This allows the transcoding to take hours without holding up adding new media to the queue. Since NATS is used in the background we can pull media from the queue as the transcode function finishes. The transcoding itself is done by a great Ruby executable found [here](https://github.com/donmelton/video_transcoding). Huge shoutout to donmelton for making a great library.

#### Note on async worker timeout

You should set the queue-worker environment variable `ack_timeout` to a suitable value based on the number of computing resources of each node. I have a 6 core processor doing the lifting with the `ack_timeout` set to 28800s (8 hours). This has worked well for me so far since some Blue-ray media can be in the 30-40 GB range.

#### Steps

- Download the media from the `transcode` Minio bucket
- Transcode the media into `/tmp`
- Upload the finished file to the `complete` Minio bucket
- Delete the raw file from the local container

### transcode-move

This step moves the completed media file from the `complete` Minio bucket to another Minio server of your choice.

#### Steps

- Download from the `complete` bucket
- Upload to the `media` bucket on another server

### transcode-delete

This function just provides clean up for the raw and intermediary files. We wait until the end to clean up so we can debug if something failed to move between buckets or severs.

#### Steps

- Check for the presence of the media at the final destination
- Delete the media from the `complete` and `transcode` buckets on the other server

Photo by Quinten de Graaf on Unsplash
