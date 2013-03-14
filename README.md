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

See [example manifest](http://cloudfoundry.github.com/docs/running/deploying-cf/vsphere/cloud-foundry-example-manifest.html)

1. add the lines below under `releases` sections (Assume the name of the
release created in step 1 is services-sample):

    <pre>
        - name: services-sample
          version: latest
    </pre>

2. append echo service into `external_service_tokens` under 'properties' section like this:

    <pre>
        external_service_tokens:
           oauth2: foooauth2token
           echo: changeechotoken
    </pre>

3. add below code under `jobs` section:

    <pre>
        - name: echo_node
          release: services-sample
          template: echo_node
          instances: 1
          resource_pool: infrastructure
          persistent_disk: 512
          networks:
          - name: default
            static_ips:
            - ***.***.***.*** # fit with your own network zone ip address

        - name: echo_gateway   # this gateway contact with legacy cc 
          release: services-sample
          template: echo_gateway
          instances: 1
          resource_pool: infrastructure
          networks:
          - name: default

        - name: echo_gateway2  # this gateway contact with ccng
          release: services-sample
          template: echo_gateway
          instances: 1
          resource_pool: infrastructure
          networks:
          - name: default
          properties:
            nats_props: nats2
            cc:
              srv_api_uri: ***  # fit with ccng service api uri, the value is under manifest section 'properties.ccng.srv_api_uri'
            supported_plans: ["free"]
            echo_gateway:
              cc_api_version: v2
              default_plan: "free"
            uaa_client_id: "vmc"
            uaa_endpoint: https://uaa.***  # fit with domain info, the value is under manifest section 'properties.domain'
            uaa_client_auth_credentials:
              username: *** # the username value is under manifest section 'properties.uaa.scim.users'
              password: *** # the password value is under manifest section 'properties.uaa.scim.users'
    </pre>
and add echo service gateway token under `properties` section:

    <pre>
        echo_gateway:
          token: changeechotoken
          service_timeout: 15
          node_timeout: 10

        echoserver:
          port: 5002
    </pre>
change resource pool size if necessary, the value is under manifest section `resource_pools.infrastructure.size`

When everything is done, fire `bosh deploy` to deploy echo service into Cloud Foundry.

## Deploy echo service under ccng v2

Deploying echo service under ccng is the same as that under legacy CC.
The difference is that to make echo service available, some additional
steps are necessary. To achieve that, you should use the high version
vmc and the vmc admin plugin:

    gem install vmc --pre
    gem install admin-vmc-plugin
    touch ~/.vmc/use-ng

login as the cc administator:

    vmc login $ADMIN_USER --password $ADMIN_PASSWORD
    vmc create-service-auth-token --provider core --token changeechotoken --label echo

$ADMIN_USER and $ADMIN_PASSWORD could be found in your bosh deployment
manifest under section `uaa.scim.users`. It may take several minutes
before it takes effect. Issue `vmc info --services` to make sure that
echo service is available.

The value of token argument, here is `changeechotoken`, must be same
with the value under manifest section `properties.echo_gateway.token`.

## License

Cloud Foundry uses the Apache 2 license.  See LICENSE for details.
