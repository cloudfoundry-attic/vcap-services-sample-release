Copyright (c) 2009-2011 VMware, Inc.

## Overview
### What is Cloud Foundry?

Cloud Foundry is an open platform-as-a-service (PaaS). The system supports
multiple frameworks, multiple application infrastructure services and
deployment to multiple clouds.

### What is this component?

This repo contains the BOSH packages and jobs specifications for a sample
Cloud Foundry service: [echo service](https://github.com/cloudfoundry/vcap-services/tree/master/echo).
The section below will walk you through how to deploy echo service into
Cloud Foundry.

## Deploy echo service with BOSH

Since current bosh supports deploying from multiple releases, deploying
echo service is a little different from that before.

#### Step 1: clone this repo and make a bosh release

    git clone git://github.com/cloudfoundry/vcap-services-sample-release.git  vcap-services-sample-release
    cd vcap-services-sample-release
    ./update
    bosh create release
    bosh upload release

#### Step 2: customize your bosh deployment manifest
1. add the lines below under `releases` sections (Assume the name of the
release created in step 1 is services-sample):

    <pre>
        - name: services-sample
          version: latest
    </pre>

2. add `echo` into `builtin_services` section of cloud controller configuration file - see the [example](https://github.com/cloudfoundry/cf-release/blob/master/jobs/cloud_controller/templates/cloud_controller.yml.erb#L161) . An simpler alternative way to achieve this is for CC to get the token from external_service_tokens - see [this]( https://github.com/cloudfoundry/cf-release/blob/master/jobs/cloud_controller/templates/cloud_controller.yml.erb#L174-L179). Add `external_service_tokens` under 'properties' section like this:

    <pre>
        external_service_tokens:
           oauth2: foooauth2token
    </pre>

3. add below code under `jobs` section, see [mysql service example](https://github.com/cloudfoundry/oss-docs/blob/master/bosh/samples/cloudfoundry.yml#L294-#L305)

    <pre>
        - name: echo_node
          release: services-sample
          template: echo_node
          instances: 1
          resource_pool: infrastructure
          persistent_disk: 128
          networks:
          - name: default
            static_ips:
            - 192.0.2.90

        - name: echo_gateway
          release: services-sample
          template: echo_gateway
          instances: 1
          resource_pool: infrastructure
          networks:
          - name: default
    </pre>
and add gateway token under `properties` section: (refer to the [example](https://github.com/cloudfoundry/oss-docs/blob/master/bosh/samples/cloudfoundry.yml#L630-#L635))

    <pre>
        echo_gateway:
          token: changeechotoken
          service_timeout: 15
          node_timeout: 10
        echoserver:
          port: 5002
    </pre>
change resource pool size if necessary, see [this](https://github.com/cloudfoundry/oss-docs/blob/master/bosh/samples/cloudfoundry.yml#L55)

When everything is done, fire `bosh deploy` to deploy echo service into Cloud Foundry.

## Deploy echo service under ccng

Deploying echo service under ccng is the same as that under legacy CC.
The difference is that to make echo service available, some additional
steps are necessary. To achieve that, you should use the high version
vmc and the vmc admin plugin:

    gem install vmc --pre
    gem install admin-vmc-plugin

login as the cc administator:

    vmc login $ADMIN_USER --password $ADMIN_PASSWORD
    vmc create-service-auth-token --provider core --token changeechotoken --label echo

$ADMIN_USER and $ADMIN_PASSWORD could be found in your bosh deployment
manifest under section 'uaa.scim.users'. It may take several minutes
before it takes effect. Issue `vmc info --services` to make sure that
echo service is available.

## License

Cloud Foundry uses the Apache 2 license.  See LICENSE for details.
