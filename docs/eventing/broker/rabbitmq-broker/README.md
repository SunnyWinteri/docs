# Creating a RabbitMQ Broker

This topic describes how to create a RabbitMQ Broker.

## Prerequisites

To use the RabbitMQ Broker, you must have the following installed:

1. [Knative Eventing](../../../install/yaml-install/eventing/install-eventing-with-yaml.md)
1. [RabbitMQ Cluster Operator](https://github.com/rabbitmq/cluster-operator) - our recommendation is [latest release](https://github.com/rabbitmq/cluster-operator/releases/latest)
1. [CertManager v1.5.4](https://github.com/jetstack/cert-manager/releases/tag/v1.5.4) - easiest integration with RabbitMQ Messaging Topology Operator
1. [RabbitMQ Messaging Topology Operator](https://github.com/rabbitmq/messaging-topology-operator) - our recommendation is [latest release](https://github.com/rabbitmq/messaging-topology-operator/releases/latest) with CertManager

## Install the RabbitMQ controller

1. Install the RabbitMQ controller by running the command:

    ```bash
    kubectl apply -f {{ artifact(org="knative-sandbox", repo="eventing-rabbitmq", file="rabbitmq-broker.yaml") }}
    ```

1. Verify that `rabbitmq-broker-controller` and `rabbitmq-broker-webhook` are running:

    ```bash
    kubectl get deployments.apps -n knative-eventing
    ```

    Example output:

    ```{ .bash .no-copy }
    NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
    eventing-controller            1/1     1            1           10s
    eventing-webhook               1/1     1            1           9s
    rabbitmq-broker-controller     1/1     1            1           3s
    rabbitmq-broker-webhook        1/1     1            1           4s
    ```

## Create a RabbitMQ cluster

1. Deploy a RabbitMQ cluster:

    1. Create a YAML file using the following template:

        ```yaml
        apiVersion: rabbitmq.com/v1beta1
        kind: RabbitmqCluster
        metadata:
          name: <cluster-name>
          annotations:
            # A single RabbitMQ cluster per Knative Eventing installation
            rabbitmq.com/topology-allowed-namespaces: "*"
        ```
        Where `<cluster-name>` is the name you want for your RabbitMQ cluster,
        for example, `rabbitmq`.

    1. Apply the YAML file by running the command:

        ```bash
        kubectl create -f <filename>
        ```
        Where `<filename>` is the name of the file you created in the previous step.

1. Wait for the cluster to become ready. When the cluster is ready, `ALLREPLICASREADY`
will be `true` in the output of the following command:

    ```bash
    kubectl get rmq <cluster-name>
    ```
    Where `<cluster-name>` is the name you gave your cluster in the step above.

    Example output:

    ```{ .bash .no-copy }
    NAME          ALLREPLICASREADY   RECONCILESUCCESS   AGE
    rabbitmq      True               True               38s
    ```

For more information about configuring the `RabbitmqCluster` CRD, see the
[RabbitMQ website](https://www.rabbitmq.com/kubernetes/operator/using-operator.html).

## Create a RabbitMQ broker config object

1. Create a YAML file using the following template:
    ```yaml
    apiVersion: eventing.knative.dev/v1alpha1
    kind: RabbitmqBrokerConfig
    metadata:
      name: <rabbitmq-broker-config-name>
    spec:
      rabbitmqClusterReference:
        name: <cluster-name>
      queueType: quorum
    ```
    Where <cluster-name> is the name of the RabbitMQ cluster created in the step above.

1. Apply the YAML file by running the command:

    ```bash
    kubectl create -f <filename>
    ```
   Where `<filename>` is the name of the file you created in the previous step.

## Create a RabbitMQ broker object

1. Create a YAML file using the following template:

    ```yaml
    apiVersion: eventing.knative.dev/v1
    kind: Broker
    metadata:
      annotations:
        eventing.knative.dev/broker.class: RabbitMQBroker
      name: <broker-name>
    spec:
      config:
        apiVersion: rabbitmq.com/v1beta1
        kind: RabbitmqBrokerConfig
        name: <rabbitmq-broker-config-name>
    ```
    Where `<rabbitmq-broker-config-name>` is the name you gave your RabbitMQ Broker config in the step above.

1. Apply the YAML file by running the command:

    ```bash
    kubectl apply -f <filename>
    ```
    Where `<filename>` is the name of the file you created in the previous step.

## Configure message ordering

By default, Triggers will consume messages one at a time to preserve ordering. If ordering of events isn't important and higher performance is desired, you can configure this by using the
`parallelism` annotation. Setting `parallelism` to `n` creates `n` workers for the Trigger that will all consume messages in parallel.

The following YAML shows an example of a Trigger with parallelism set to `10`:

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: high-throughput-trigger
  annotations:
    rabbitmq.eventing.knative.dev/parallelism: "10"
...

## Additional information

To report a bug or request a feature, open an issue in the [eventing-rabbitmq repository](https://github.com/knative-sandbox/eventing-rabbitmq).
