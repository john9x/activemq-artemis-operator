---
title: "Using the custom init image"  
description: "Introduction to the operator custom init image"
draft: false
images: []
menu:
  docs:
    parent: "tutorials"
weight: 110
toc: true
---

Starting from [v0.18.1](https://github.com/arkmq-org/activemq-artemis-operator/tree/v0.18.1), the arkmq-org Operator
enables you to specify a **custom Init Container image**. Specifying a custom Init Container image allows you to provide your own broker configuration within the Operator framework.

### What is a custom Init Container image?
The arkmq-org Operator uses a [Custom Resource Definition (CRD)](https://github.com/arkmq-org/activemq-artemis-operator/blob/v0.18.1/deploy/crds/broker_activemqartemis_crd.yaml) to define the broker configuration. In Kubernetes, a CRD is a schema of configuration items or parameters. By creating a corresponding Custom Resource (CR) instance, you can specify values for configuration items in the CRD. For example, you can define address settings
in this [sample CR file](https://github.com/arkmq-org/activemq-artemis-operator/blob/v0.18.1/deploy/examples/artemis-basic-address-settings-deployment.yaml).

For configuration that _isn't_ exposed in the CRD, you can specify a custom Init Container image to manipulate or add to the configuration that has been created by the Operator. When the CR is deployed and the broker instance is created, the Operator will then run a  user-provided post-configuration script.

### How it works
Internally, the arkmq-org Operator uses a specialized container called an _Init Container_ to configure each broker instance. If no custom image for the Init Container has been specified, the Operator uses a [built-in Init Container image](https://quay.io/repository/arkmq-org/activemq-artemis-broker-init), which
is responsible for the configuration. The broker configuration generated by the Init Container is passed to the [broker container image](https://quay.io/repository/arkmq-org/activemq-artemis-broker-kubernetes) to use when launching the broker.

If a custom Init Container image _is_ provided and specified in the Custom Resource (CR), then this is used to create the configuration. Since the custom image is built on top of the arkmq-org built-in Init Container image, the custom image first generates the broker configuration that is defined in the CR. Then, the Operator executes the post-configuration (`post-config.sh`) script provided by the custom Init Container.

For ease of use, the environment variable `CONFIG_INSTANCE_DIR` is set. This environment variable points to the broker instance directory, which has the following structure:

<a name="instancedir"></a>
```
${CONFIG_INSTANCE_DIR}
  |
  \--/bin
  \--/etc
  \--/data
  \--/lib
  \--/log
  \--/tmp
```
When the `post-config.sh` script is invoked, the directory specified in `CONFIG_INSTANCE_DIR` already contains all of the configuration
files generated by the built-in image (via the configuration specified in the CR) and is available for the `post-config.sh` script to do extra configuration.

For example, a custom Init Container might be used to modify the logging settings and add or override some specific configurations in `${CONFIG_INSTANCE_DIR}/etc`. In addition, the custom Init Container might put extra runtime dependencies (`.jar` files) in the `${CONFIG_INSTANCE_DIR}/lib` directory so that the broker can
use them.

After the `post-config.sh` script is executed, the broker instance is launched with the updated configuration.

### Specifying a custom Init Container image
1. First, you need to implement your custom Init Container image.

    The custom image should follow these predefined rules:

    - The custom image must use the arkmq-org Operator's [built-in Init Container image](https://quay.io/repository/arkmq-org/activemq-artemis-broker-init) as the base image in its Docker file. For example:

    ```
    FROM quay.io/arkmq-org/activemq-artemis-broker-init:0.2.3
    ...
    ```

    - The custom image must include a `post-config.sh` script in the `/amq/scripts` directory. The absolute path will be `/amq/scripts/post-config.sh`.

    The `post-config.sh` script is where you modify the configuration of the broker or add any third party dependencies.

    If you need additional resources (`.xml` files, `.jar` files, etc.) for your custom configuration, you need to add them to your image and make sure that they are accessible to your post-config scripts.

2. Next, you need to build your custom Init Container image and put it in a container repository (for example you can create a repository on [Red Hat Quay](https://quay.io)).

3. When you have added the image to a repository, you need to configure the Operator to use the custom Init Container image. To do this, edit the CR file. For the `image` property, specify the custom image. For example:

    ```yaml
    apiVersion: broker.amq.io/v2alpha4
    kind: ActiveMQArtemis
    metadata:
        name: ex-aao
        spec:
            deploymentPlan:
                size: 1
                image: quay.io/arkmq-org/activemq-artemis-broker-kubernetes:0.2.1
                initImage: <your_custom_init_image_url>
                ...
    ```

4. Finally, deploy the CR file in the usual manner. For more information, see [Getting Started with the arkmq-org Operator](using_operator.md).

### Further information
* A fully working example is available [here](https://github.com/arkmq-org/arkmq-org-examples/tree/main/operator/init/jdbc).

* For issues or suggestions please open an issue at [arkmq-org](https://github.com/arkmq-org/activemq-artemis-operator/issues).
