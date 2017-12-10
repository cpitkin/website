---
title: "Automated Video Transcoding with OpenFaaS, Minio, and Nomad"
date: 2017-12-10
tags: [infra, puppet, open-source]
draft: true
---
**Need to check steps for proper ordering**

# Setup an automated video transcoder

## Services
- [Minio](https://minio.io/)
- [Docker](https://www.docker.com/)
- [Nomad](https://www.nomadproject.io/)
- [Consul](https://www.consul.io/)
- [OpenFaaS](https://github.com/hashicorp/faas-nomad)

## Docker
Docker containers running on a single machine share that machine's operating system kernel; they start instantly and use less compute and RAM. Images are constructed from filesystem layers and share common files. This minimizes disk usage and image downloads are much faster.

### Setup Docker
You will need a machine that is capable of running Docker. This can be any machine you would like but remember we are going to be doing video transcoding so you will want to have a plenty of extra CPU.

1. **Test this out with Puppet setup**

## Nomad
Nomad is a tool for managing a cluster of machines and unning applications on them. Nomad abstracts away machines and the location of applications, and instead enables users to declare what they want to run and Nomad handles where they should run and how to run them.

We're going to be using Puppet to bootstrap a Nomad server and client. [Nomad](https://github.com/dudemcbacon/puppet-nomad) Puppet module reference.

1. Install Puppet for your Ubuntu. Debian also works but for the sake of this tutorial we are going be to using Ubuntu. The Debian repos can be found [here](https://puppet.com/docs/puppet/5.3/puppet_platform.html#debian-9-stretch)

16.04 LTS
```bash
wget https://apt.puppetlabs.com/puppet5-release-xenial.deb
sudo dpkg -i puppet5-release-xenial.deb
sudo apt update
sudo apt-get install puppet-agent
```
14.04 LTS
```bash
wget https://apt.puppetlabs.com/puppet5-release-trusty.deb
sudo dpkg -i puppet5-release-trusty.deb
sudo apt-get update
sudo apt-get install puppet-agent
```
2. Exit the current terminal and start a new one to get `puppet` in your PATH
3. Clone [this]() repo with all the module dependancies needed to setup Nomad with Puppet
3. `cd puppet-bootstrap-nomad`
4. `puppet apply --modulepath=modules setup.pp`
5. Open <server_ip>:4646 in a browser to view the web UI
6. You are now ready to schedule jobs in Nomad

**_TODO:_** Fill in Puppet bootstrap repo URL above (step 2)

**REMOVE:** Download Nomad from [here](https://www.nomadproject.io/downloads.html)

## OpenFaaS
With OpenFaaS you can package anything as a serverless function - from Node.js to Golang to CSharp, even binaries like ffmpeg or ImageMagick.

1. We create a function or you can use mine and edit it how you want
2. Let go back and edit the

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

**ADD certificates to your server**

2. Go to the FreeNAS UI under the Services section. Click on the wrench next to the S3 label. Here you will need to fill in IP, port, disks, and if you want to enable the web UI.

**INSERT PICTURE HERE**

3. Now we can start the service in FreeNAS.

**INSERT PICTURE HERE**

4. Create a bucket called transcode. You can do this through the UI or with the Minio client (mc) cli tool.
`mc mb <minio_server_alias>/<bucket_name> --region <region>`

5. Now we need to create a bucket notification to trigger on object put events. Using the mc tool we can enter the following to set the event trigger.
`mc events add <minio_server_alias>/<bucket_name> arn:minio:sqs:<region>:1:webhook --events put --suffix .mkv`

6.
