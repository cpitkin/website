---
author: "Charlie Pitkin"
title: Setup OpenFaaS Nomad
date: 2017-12-10
draft: false
---

## Services
- [Docker](https://www.docker.com/)
- [Nomad](https://www.nomadproject.io/)
- [Consul](https://www.consul.io/)
- [OpenFaaS](https://github.com/openfaas/faas)

## Setup

We're going to be using Puppet to bootstrap Docker, Consul, and Nomad.

1. Login and gain root access
2. Install Puppet for your Ubuntu LTS version. Debian also works but for the sake of this tutorial we are going be to using Ubuntu. The Debian repos can be found [here](https://puppet.com/docs/puppet/5.3/puppet_platform.html#debian-9-stretch)

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
3. Exit the current terminal and start a new one to get `puppet` in your PATH
4. Clone [this](https://github.com/cpitkin/puppet-openfaas-nomad.git) repo to your server
5. `cd puppet-openfaas-nomad`
6. `sudo su - && puppet apply --modulepath=modules setup.pp`
7. Open \<server_ip\>:4646 in a browser to view the Nomad web UI
8. Open \<server_ip\>:8500 in a browser to view the Consul web UI
9. You are now ready to schedule jobs in Nomad

## OpenFaaS
With OpenFaaS you can package anything as a serverless function - from Node.js to Golang to CSharp, even binaries like ffmpeg or ImageMagick.

1. Clone [this](https://github.com/hashicorp/faas-nomad) repo on the server
2. Following [these](https://github.com/hashicorp/faas-nomad#running-the-openfaas-application) instructions will get you up and running in no time
