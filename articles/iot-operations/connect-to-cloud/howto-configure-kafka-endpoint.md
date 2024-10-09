---
title: Configure Azure Event Hubs and Kafka dataflow endpoints in Azure IoT Operations
description: Learn how to configure dataflow endpoints for Kafka in Azure IoT Operations.
author: PatAltimore
ms.author: patricka
ms.subservice: azure-data-flows
ms.topic: how-to
ms.date: 10/02/2024
ai-usage: ai-assisted

#CustomerIntent: As an operator, I want to understand how to configure dataflow endpoints for Kafka in Azure IoT Operations so that I can send data to and from Kafka endpoints.
---

# Configure Azure Event Hubs and Kafka dataflow endpoints

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

To set up bi-directional communication between Azure IoT Operations Preview and Apache Kafka brokers, you can configure a dataflow endpoint. This configuration allows you to specify the endpoint, Transport Layer Security (TLS), authentication, and other settings.

## Prerequisites

- An instance of [Azure IoT Operations Preview](../deploy-iot-ops/howto-deploy-iot-operations.md)
- A [configured dataflow profile](howto-configure-dataflow-profile.md)

## Create a Kafka dataflow endpoint

To create a dataflow endpoint for Kafka, you need to specify the Kafka broker host, authentication method, TLS settings, and other settings. You can use the endpoint as a source or destination in a dataflow. When used with Azure Event Hubs, you can use managed identity for authentication that eliminates the need to manage secrets.

### Azure Event Hubs

[Azure Event Hubs is compatible with the Kafka protocol](../../event-hubs/azure-event-hubs-kafka-overview.md) and can be used with dataflows with some limitations.

If you're using Azure Event Hubs, create an Azure Event Hubs namespace and a Kafka-enabled event hub for each Kafka topic. 

To configure a dataflow endpoint for a Kafka endpoint, we suggest using the managed identity of the Azure Arc-enabled Kubernetes cluster. This approach is secure and eliminates the need for secret management.

# [Portal](#tab/portal)

1. In the [operations experience](https://iotoperations.azure.com/), select the **Dataflow endpoints** tab.
1. Under **Create new dataflow endpoint**, select **Azure Event Hubs** > **New**.

    :::image type="content" source="media/howto-configure-kafka-endpoint/create-event-hubs-endpoint.png" alt-text="Screenshot using operations experience to create an Azure Event Hubs dataflow endpoint.":::

1. Enter the following settings for the endpoint:

    | Setting              | Description                                                                                       |
    | -------------------- | ------------------------------------------------------------------------------------------------- |
    | Name                 | The name of the dataflow endpoint.                                     |
    | Host                 | The hostname of the Kafka broker in the format `<HOST>.servicebus.windows.net:9093`. Include the port number `9093` in the host name for Event Hubs.|
    | Authentication method| The method used for authentication. Choose *System assigned managed identity*, *User assigned managed identity*, or *SASL*. |
    | SASL type            | The type of SASL authentication. Choose *Plain*, *ScramSha256*, or *ScramSha512*. Required if using *SASL*. |
    | Synced secret name   | The name of the secret. Required if using *SASL* or *X509*. |
    | Username reference of token secret | The reference to the username in the SASL token secret. Required if using *SASL*. |

# [Kubernetes](#tab/kubernetes)

1. Get the managed identity of the Azure IoT Operations Arc extension.
1. Assign the managed identity to the Event Hubs namespace with the `Azure Event Hubs Data Sender` or `Azure Event Hubs Data Receiver` role.
1. Create the *DataflowEndpoint* resource and specify the managed identity authentication method.

    ```yaml
    apiVersion: connectivity.iotoperations.azure.com/v1beta1
    kind: DataflowEndpoint
    metadata:
      name: eventhubs
      namespace: azure-iot-operations
    spec:
      endpointType: Kafka
      kafkaSettings:
        host: <HOST>.servicebus.windows.net:9093
        authentication:
          method: SystemAssignedManagedIdentity
          systemAssignedManagedIdentitySettings: {}
        tls:
          mode: Enabled
        consumerGroupId: mqConnector
    ```

The Kafka topic, or individual event hub, is configured later when you create the dataflow. The Kafka topic is the destination for the dataflow messages.

#### Use connection string for authentication to Event Hubs

To use connection string for authentication to Event Hubs, update the `authentication` section of the Kafka settings to use the `Sasl` method and configure the `saslSettings` with the `saslType` as `Plain` and the `secretRef` with the name of the secret that contains the connection string.

```yaml
spec:
  kafkaSettings:
    authentication:
      method: Sasl
      saslSettings:
        saslType: Plain
        secretRef: <YOUR-TOKEN-SECRET-NAME>
    tls:
      mode: Enabled
```

In the example, the `secretRef` is the name of the secret that contains the connection string. The secret must be in the same namespace as the Kafka dataflow resource. The secret must have both the username and password as key-value pairs. For example:

```bash
kubectl create secret generic cs-secret -n azure-iot-operations \
  --from-literal=username='$ConnectionString' \
  --from-literal=password='Endpoint=sb://<NAMESPACE>.servicebus.windows.net/;SharedAccessKeyName=<KEY-NAME>;SharedAccessKey=<KEY>'
```
> [!TIP]
> Scoping the connection string to the namespace (as opposed to individual event hubs) allows a dataflow to send and receive messages from multiple different event hubs and Kafka topics.

---

#### Limitations

Azure Event Hubs [doesn't support all the compression types that Kafka supports](../../event-hubs/azure-event-hubs-kafka-overview.md#compression). Only GZIP compression is supported in Azure Event Hubs premium and dedicated tiers currently. Using other compression types might result in errors.

### Other Kafka brokers

To configure a dataflow endpoint for non-Event-Hub Kafka brokers, set the host, TLS, authentication, and other settings as needed.

# [Portal](#tab/portal)

1. In the [operations experience](https://iotoperations.azure.com/), select the **Dataflow endpoints** tab.
1. Under **Create new dataflow endpoint**, select **Custom Kafka Broker** > **New**.

    :::image type="content" source="media/howto-configure-kafka-endpoint/create-kafka-endpoint.png" alt-text="Screenshot using operations experience to create a Kafka dataflow endpoint.":::

1. Enter the following settings for the endpoint:

    | Setting              | Description                                                                                       |
    | -------------------- | ------------------------------------------------------------------------------------------------- |
    | Name                 | The name of the dataflow endpoint.                                     |
    | Host                 | The hostname of the Kafka broker in the format `<HOST>.servicebus.windows.net:xxxx`. Include the port number in the host name. |
    | Authentication method| The method used for authentication. Choose *System assigned managed identity*, *User assigned managed identity*, *SASL*, or *X509 certificate*. |
    | SASL type            | The type of SASL authentication. Choose *Plain*, *ScramSha256*, or *ScramSha512*. Required if using *SASL*. |
    | Synced secret name   | The name of the secret. Required if using *SASL* or *X509*. |
    | Username reference of token secret | The reference to the username in the SASL token secret. Required if using *SASL*. |
    | X509 client certificate | The X.509 client certificate used for authentication. Required if using *X509*. |
    | X509 intermediate certificates | The intermediate certificates for the X.509 client certificate chain. Required if using *X509*. |
    | X509 client key | The private key corresponding to the X.509 client certificate. Required if using *X509*. |

1. Select **Apply** to provision the endpoint.

> [!NOTE]
> Currently, the operations experience doesn't support using a Kafka dataflow endpoint as a source. You can create a dataflow with a source Kafka dataflow endpoint using the Kubernetes or Bicep.

# [Kubernetes](#tab/kubernetes)

```yaml
apiVersion: connectivity.iotoperations.azure.com/v1beta1
kind: DataflowEndpoint
metadata:
  name: kafka
  namespace: azure-iot-operations
spec:
  endpointType: Kafka
  kafkaSettings:
    host: <KAFKA-HOST>:<PORT>
    authentication:
      method: Sasl
      saslSettings:
        saslType: ScramSha256
        secretRef: <YOUR-TOKEN-SECRET-NAME>
    tls:
      mode: Enabled
    consumerGroupId: mqConnector
```

---

## Use the endpoint in a dataflow source or destination

Once the endpoint is created, you can use it in a dataflow by specifying the endpoint name in the dataflow's source or destination settings.

# [Portal](#tab/portal)

1. In the Azure IoT Operations Preview portal, create a new dataflow or edit an existing dataflow by selecting the **Dataflows** tab. If creating a new dataflow, select **Create dataflow** and replace `<new-dataflow>` with a name for the dataflow.
1. In the editor, select the source endpoint. Kafka endpoints can be used as both source and destination. Currently, you can only use the portal to create a dataflow with a Kafka endpoint as a destination. Use a Kubernetes custom resource or Bicep to create a dataflow with a Kafka endpoint as a source.
1. Choose the Kafka dataflow endpoint that you created previously.
1. Specify the Kafka topic where messages are sent.

    :::image type="content" source="media/howto-configure-kafka-endpoint/dataflow-mq-kafka.png" alt-text="Screenshot using operations experience to create a dataflow with an MQTT source and Azure Event Hubs destination.":::

# [Kubernetes](#tab/kubernetes)

```yaml
apiVersion: connectivity.iotoperations.azure.com/v1beta1
kind: Dataflow
metadata:
  name: my-dataflow
  namespace: azure-iot-operations
spec:
  profileRef: default
  mode: Enabled
  operations:
    - operationType: Source
      sourceSettings:
        endpointRef: mq
        dataSources:
          *
    - operationType: Destination
      destinationSettings:
        endpointRef: kafka
```

---

For more information about dataflow destination settings, see [Create a dataflow](howto-create-dataflow.md).

To customize the endpoint settings, see the following sections for more information.

### Available authentication methods

The following authentication methods are available for Kafka broker dataflow endpoints.  For more information about enabling secure settings by configuring an Azure Key Vault and enabling workload identities, see [Enable secure settings in Azure IoT Operations Preview deployment](../deploy-iot-ops/howto-enable-secure-settings.md).

#### SASL

# [Portal](#tab/portal)

In the operations experience dataflow endpoint settings page, select the **Basic** tab then choose **Authentication method** > **SASL**.

Enter the following settings for the endpoint:

| Setting                        | Description                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------------------- |
| SASL type                      | The type of SASL authentication to use. Supported types are `Plain`, `ScramSha256`, and `ScramSha512`. |
| Synced secret name             | The name of the Kubernetes secret that contains the SASL token.                                   |
| Username reference or token secret | The reference to the username or token secret used for SASL authentication.                     |
| Password reference of token secret | The reference to the password or token secret used for SASL authentication.                     |

# [Kubernetes](#tab/kubernetes)

To use SASL for authentication, update the `authentication` section of the Kafka settings to use the `Sasl` method and configure the `saslSettings` with the `saslType` and the `secretRef` with the name of the secret that contains the SASL token.

```yaml
kafkaSettings:
  authentication:
    method: Sasl
    saslSettings:
      saslType: Plain
      secretRef: <YOUR-TOKEN-SECRET-NAME>
```

The supported SASL types are:

- `Plain`
- `ScramSha256`
- `ScramSha512`

The secret must be in the same namespace as the Kafka dataflow resource. The secret must have the SASL token as a key-value pair. For example:

```bash
kubectl create secret generic sasl-secret -n azure-iot-operations \
  --from-literal=token='your-sasl-token'
```

---

#### X.509

# [Portal](#tab/portal)

In the operations experience dataflow endpoint settings page, select the **Basic** tab then choose **Authentication method** > **X509 certificate**.

Enter the following settings for the endpoint:

| Setting               | Description                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------- |
| Synced secret name   | The name of the secret. |
| X509 client certificate | The X.509 client certificate used for authentication. |
| X509 intermediate certificates | The intermediate certificates for the X.509 client certificate chain.  |
| X509 client key       | The private key corresponding to the X.509 client certificate. |

# [Kubernetes](#tab/kubernetes)

To use X.509 for authentication, update the `authentication` section of the Kafka settings to use the `X509Certificate` method and configure the `x509CertificateSettings` with the `secretRef` with the name of the secret that contains the X.509 certificate.

```yaml
kafkaSettings:
  authentication:
    method: X509Certificate
    x509CertificateSettings:
      secretRef: <YOUR-TOKEN-SECRET-NAME>
```

The secret must be in the same namespace as the Kafka dataflow resource. Use Kubernetes TLS secret containing the public certificate and private key. For example:

```bash
kubectl create secret tls my-tls-secret -n azure-iot-operations \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

---

#### System-assigned managed identity

To use system-assigned managed identity for authentication, first assign a role to the Azure IoT Operation managed identity that grants permission to send and receive messages from Event Hubs, such as Azure Event Hubs Data Owner or Azure Event Hubs Data Sender/Receiver. To learn more, see [Authenticate an application with Microsoft Entra ID to access Event Hubs resources](../../event-hubs/authenticate-application.md#built-in-roles-for-azure-event-hubs).

# [Portal](#tab/portal)

In the operations experience dataflow endpoint settings page, select the **Basic** tab then choose **Authentication method** > **System assigned managed identity**.

# [Kubernetes](#tab/kubernetes)

Update the `authentication` section of the DataflowEndpoint Kafka settings to use the `SystemAssignedManagedIdentity` method. In most cases, you can set the `systemAssignedManagedIdentitySettings` with an empty object.

```yaml
kafkaSettings:
  authentication:
    method: SystemAssignedManagedIdentity
    systemAssignedManagedIdentitySettings:
      {}
```

This sets the audience to the default value, which is the same as the Event Hubs namespace host value in the form of `https://<NAMESPACE>.servicebus.windows.net`. However, if you need to override the default audience, you can set the `audience` field to the desired value. The audience is the resource that the managed identity is requesting access to. For example:

```yaml
kafkaSettings:
  authentication:
    method: SystemAssignedManagedIdentity
    systemAssignedManagedIdentitySettings:
      audience: <YOUR-AUDIENCE-OVERRIDE-VALUE>
```

---

#### User-assigned managed identity

# [Portal](#tab/portal)

In the operations experience dataflow endpoint settings page, select the **Basic** tab then choose **Authentication method** > **User assigned managed identity**.

Enter the user assigned managed identity client ID and tenant ID in the appropriate fields.

# [Kubernetes](#tab/kubernetes)

To use a user-assigned managed identity, specify the `UserAssignedManagedIdentity` authentication method.


```yaml
kafkaSettings:
  authentication:
    method: UserAssignedManagedIdentity
    userAssignedManagedIdentitySettings:
        {}
```

<!-- TODO: Add link to WLIF docs -->

---

#### Anonymous

To use anonymous authentication, update the `authentication` section of the Kafka settings to use the `Anonymous` method.

```yaml
kafkaSettings:
  authentication:
    method: Anonymous
    anonymousSettings:
      {}
```

## Advanced settings

You can set advanced settings for the Kafka dataflow endpoint such as TLS, trusted CA certificate, Kafka messaging settings, batching, and CloudEvents. You can set these settings in the dataflow endpoint **Advanced** portal tab or within the dataflow endpoint custom resource.

# [Portal](#tab/portal)

In the operations experience, select the **Advanced** tab for the dataflow endpoint.

:::image type="content" source="media/howto-configure-kafka-endpoint/kafka-advanced.png" alt-text="Screenshot using operations experience to set Kafka dataflow endpoint advanced settings.":::

| Setting                        | Description                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------------------- |
| Consumer group ID              | The ID of the consumer group for the Kafka endpoint. The consumer group ID is used to identify the consumer group that the dataflow uses to read messages from the Kafka topic. The consumer group ID must be unique within the Kafka broker. |
| Compression                    | The compression type used for messages sent to Kafka topics. Supported types are `None`, `Gzip`, `Snappy`, and `Lz4`. Compression helps to reduce the network bandwidth and storage space required for data transfer. However, compression also adds some overhead and latency to the process. This setting takes effect only if the endpoint is used as a destination where the dataflow is a producer. |
| Copy MQTT properties           | Whether to copy MQTT message properties to Kafka message headers. For more information, see [Copy MQTT properties](#copy-mqtt-properties). |
| Kafka acknowledgement          | The level of acknowledgement requested from the Kafka broker. Supported values are `None`, `All`, `One`, and `Zero`. For more information, see [Kafka acknowledgements](#kafka-acknowledgements). |
| Partition handling strategy    | The partition handling strategy controls how messages are assigned to Kafka partitions when sending them to Kafka topics.  Supported values are `Default`, `Static`, `Topic`, and `Property`. For more information, see [Partition handling strategy](#partition-handling-strategy). |
| TLS mode enabled               | Enables TLS for the Kafka endpoint.                         |
| Trusted CA certificate config map | The ConfigMap containing the trusted CA certificate for the Kafka endpoint. This ConfigMap should contain the CA certificate in PEM format. The ConfigMap must be in the same namespace as the Kafka dataflow resource. For more information, see [Trusted CA certificate](#trusted-ca-certificate). |
| Batching enabled               | Enables batching. Batching allows you to group multiple messages together and compress them as a single unit, which can improve the compression efficiency and reduce the network overhead. This setting takes effect only if the endpoint is used as a destination where the dataflow is a producer. |
| Batching latency               | The maximum time interval in milliseconds that messages can be buffered before being sent. If this interval is reached, then all buffered messages are sent as a batch, regardless of how many or how large they are. |
| Maximum bytes                  | The maximum size in bytes that can be buffered before being sent. If this size is reached, then all buffered messages are sent as a batch, regardless of how many they are or how long they are buffered.             |
| Message count                  | The maximum number of messages that can be buffered before being sent. If this number is reached, then all buffered messages are sent as a batch, regardless of how large they are or how long they are buffered.                 |
| Cloud event attributes         | The CloudEvents attributes to include in the Kafka messages.                                      |

# [Kubernetes](#tab/kubernetes)

### TLS settings

Under `kafkaSettings.tls`, you can configure additional settings for the TLS connection to the Kafka broker.

#### TLS mode

To enable or disable TLS for the Kafka endpoint, update the `mode` setting in the TLS settings. For example:

```yaml
kafkaSettings:
  tls:
    mode: Enabled
```

The TLS mode can be set to `Enabled` or `Disabled`. If the mode is set to `Enabled`, the dataflow uses a secure connection to the Kafka broker. If the mode is set to `Disabled`, the dataflow uses an insecure connection to the Kafka broker.

#### Trusted CA certificate

To configure the trusted CA certificate for the Kafka endpoint, update the `trustedCaCertificateConfigMapRef` setting in the TLS settings. For example:

```yaml
kafkaSettings:
  tls:
    trustedCaCertificateConfigMapRef: <YOUR-CA-CERTIFICATE>
```

This ConfigMap should contain the CA certificate in PEM format. The ConfigMap must be in the same namespace as the Kafka dataflow resource. For example:

```bash
kubectl create configmap client-ca-configmap --from-file root_ca.crt -n azure-iot-operations
```

This setting is important if the Kafka broker uses a self-signed certificate or a certificate signed by a custom CA that isn't trusted by default.

However in the case of Azure Event Hubs, the CA certificate isn't required because the Event Hubs service uses a certificate signed by a public CA that is trusted by default.

### Kafka messaging settings

Under `kafkaSettings`, you can configure additional settings for the Kafka endpoint.

#### Consumer group ID

To configure the consumer group ID for the Kafka endpoint, update the `consumerGroupId` setting in the Kafka settings. For example:

```yaml
spec:
  kafkaSettings:
    consumerGroupId: fromMq
```

The consumer group ID is used to identify the consumer group that the dataflow uses to read messages from the Kafka topic. The consumer group ID must be unique within the Kafka broker.

<!-- TODO: check for accuracy -->

This setting takes effect only if the endpoint is used as a source (that is, the dataflow is a consumer).

#### Compression

To configure the compression type for the Kafka endpoint, update the `compression` setting in the Kafka settings. For example:

```yaml
kafkaSettings:
  compression: Gzip
```

The compression field enables compression for the messages sent to Kafka topics. Compression helps to reduce the network bandwidth and storage space required for data transfer. However, compression also adds some overhead and latency to the process. The supported compression types are listed in the following table.

| Value | Description |
| ----- | ----------- |
| `None` | No compression or batching is applied. None is the default value if no compression is specified. |
| `Gzip` | GZIP compression and batching are applied. GZIP is a general-purpose compression algorithm that offers a good balance between compression ratio and speed. Only [GZIP compression is supported in Azure Event Hubs premium and dedicated tiers](../../event-hubs/azure-event-hubs-kafka-overview.md#compression) currently. |
| `Snappy` | Snappy compression and batching are applied. Snappy is a fast compression algorithm that offers moderate compression ratio and speed. |
| `Lz4` | LZ4 compression and batching are applied. LZ4 is a fast compression algorithm that offers low compression ratio and high speed. |

This setting takes effect only if the endpoint is used as a destination where the dataflow is a producer.

#### Batching

Aside from compression, you can also configure batching for messages before sending them to Kafka topics. Batching allows you to group multiple messages together and compress them as a single unit, which can improve the compression efficiency and reduce the network overhead.

| Field | Description | Required |
| ----- | ----------- | -------- |
| `mode` | Enable batching or not. If not set, the default value is Enabled because Kafka doesn't have a notion of *unbatched* messaging. If set to Disabled, the batching is minimized to create a batch with a single message each time. | No |
| `latencyMs` | The maximum time interval in milliseconds that messages can be buffered before being sent. If this interval is reached, then all buffered messages are sent as a batch, regardless of how many or how large they are. If not set, the default value is 5. | No |
| `maxMessages` | The maximum number of messages that can be buffered before being sent. If this number is reached, then all buffered messages are sent as a batch, regardless of how large they are or how long they are buffered. If not set, the default value is 100000.  | No |
| `maxBytes` | The maximum size in bytes that can be buffered before being sent. If this size is reached, then all buffered messages are sent as a batch, regardless of how many they are or how long they are buffered. The default value is 1000000 (1 MB). | No |

An example of using batching is:

```yaml
kafkaSettings:
  batching:
    enabled: true
    latencyMs: 1000
    maxMessages: 100
    maxBytes: 1024
```

In the example, messages are sent either when there are 100 messages in the buffer, or when there are 1,024 bytes in the buffer, or when 1,000 milliseconds elapse since the last send, whichever comes first.

This setting takes effect only if the endpoint is used as a destination where the dataflow is a producer.

#### Partition handling strategy

The partition handling strategy controls how messages are assigned to Kafka partitions when sending them to Kafka topics. Kafka partitions are logical segments of a Kafka topic that enable parallel processing and fault tolerance. Each message in a Kafka topic has a partition and an offset, which are used to identify and order the messages.

This setting takes effect only if the endpoint is used as a destination where the dataflow is a producer.

<!-- TODO: double check for accuracy -->

By default, a dataflow assigns messages to random partitions, using a round-robin algorithm. However, you can use different strategies to assign messages to partitions based on some criteria, such as the MQTT topic name or an MQTT message property. This can help you to achieve better load balancing, data locality, or message ordering.

| Value | Description |
| ----- | ----------- |
| `Default` | Assigns messages to random partitions, using a round-robin algorithm. This is the default value if no strategy is specified. |
| `Static` | Assigns messages to a fixed partition number that is derived from the instance ID of the dataflow. This means that each dataflow instance sends messages to a different partition. This can help to achieve better load balancing and data locality. |
| `Topic` | Uses the MQTT topic name from the dataflow source as the key for partitioning. This means that messages with the same MQTT topic name are sent to the same partition. This can help to achieve better message ordering and data locality. |
| `Property` | Uses an MQTT message property from the dataflow source as the key for partitioning. Specify the name of the property in the `partitionKeyProperty` field. This means that messages with the same property value are sent to the same partition. This can help to achieve better message ordering and data locality based on a custom criterion. |

An example of using partition handling strategy is:

```yaml
kafkaSettings:
  partitionStrategy: Property
  partitionKeyProperty: device-id
```

This means that messages with the same "device-id" property are sent to the same partition.

#### Kafka acknowledgements

Kafka acknowledgements (acks) are used to control the durability and consistency of messages sent to Kafka topics. When a producer sends a message to a Kafka topic, it can request different levels of acknowledgements from the Kafka broker to ensure that the message is successfully written to the topic and replicated across the Kafka cluster.

This setting takes effect only if the endpoint is used as a destination (that is, the dataflow is a producer).

| Value | Description |
| ----- | ----------- |
| `None` | The dataflow doesn't wait for any acknowledgements from the Kafka broker. This is the fastest but least durable option. |
| `All` | The dataflow waits for the message to be written to the leader partition and all follower partitions. This is the slowest but most durable option. This is also the default option|
| `One` | The dataflow waits for the message to be written to the leader partition and at least one follower partition. |
| `Zero` | The dataflow waits for the message to be written to the leader partition but doesn't wait for any acknowledgements from the followers. This is faster than `One` but less durable. |

<!-- TODO: double check for accuracy -->

An example of using Kafka acknowledgements is:

```yaml
kafkaSettings:
  kafkaAcks: All
```

This means that the dataflow waits for the message to be written to the leader partition and all follower partitions.

#### Copy MQTT properties

By default, the copy MQTT properties setting is enabled. These user properties include values such as `subject` that stores the name of the asset sending the message. 

```yaml
kafkaSettings:
  copyMqttProperties: Enabled
```

To disable copying MQTT properties, set the value to `Disabled`.

The following sections describe how MQTT properties are translated to Kafka user headers and vice versa when the setting is enabled.

##### Kafka endpoint is a destination

When a Kafka endpoint is a dataflow destination, all MQTT v5 specification defined properties are translated Kafka user headers. For example, an MQTT v5 message with "Content Type" being forwarded to Kafka translates into the Kafka **user header** `"Content Type":{specifiedValue}`. Similar rules apply to other built-in MQTT properties, defined in the following table.

| MQTT Property | Translated Behavior |
|---------------|----------|
| Payload Format Indicator  | Key: "Payload Format Indicator" <BR> Value: "0" (Payload is bytes) or "1" (Payload is UTF-8)
| Response Topic | Key: "Response Topic" <BR> Value: Copy of Response Topic from original message.
| Message Expiry Interval | Key: "Message Expiry Interval" <BR> Value: UTF-8 representation of number of seconds before message expires. See [Message Expiry Interval property](#the-message-expiry-interval-property) for more details.
| Correlation Data: | Key: "Correlation Data" <BR> Value: Copy of Correlation Data from original message. Unlike many MQTT v5 properties that are UTF-8 encoded, correlation data can be arbitrary data.
| Content Type: | Key: "Content Type" <BR> Value: Copy of Content Type from original message.

MQTT v5 user property key value pairs are directly translated to Kafka user headers. If a user header in a message has the same name as a built-in MQTT property (for example, a user header named "Correlation Data") then whether forwarding the MQTT v5 specification property value or the user property is undefined.

Dataflows never receive these properties from an MQTT Broker. Thus, a dataflow never forwards them:

* Topic Alias
* Subscription Identifiers

###### The Message Expiry Interval property

The [Message Expiry Interval](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html#_Toc3901112) specifies how long a message can remain in an MQTT broker before being discarded.

When a dataflow receives an MQTT message with the Message Expiry Interval specified, it:

* Records the time the message was received.
* Before the message is emitted to the destination, time is subtracted from the message has been queued from the original expiry interval time.
 * If the message has not yet expired (the operation above is > 0), then the message is emitted to the destination and contains the updated Message Expiry Time.
 * If the message has expired (the operation above is <= 0), then the message isn't emitted by the Target.

Examples:

* A dataflow receives an MQTT message with Message Expiry Interval = 3600 seconds. The corresponding destination is temporarily disconnected but is able to reconnect. 1,000 seconds pass before this MQTT Message is sent to the Target. In this case, the destination's message has its Message Expiry Interval set as 2600 (3600 - 1000) seconds.
* The dataflow receives an MQTT message with Message Expiry Interval = 3600 seconds. The corresponding destination is temporarily disconnected but is able to reconnect. In this case, however, it takes 4,000 seconds to reconnect. The message expired and dataflow doesn't forward this message to the destination.

##### Kafka endpoint is a dataflow source

> [!NOTE]
> There's a known issue when using Event Hubs endpoint as a dataflow source where Kafka header gets corrupted as its translated to MQTT. This only happens if using Event Hub though the Event Hub client which uses AMQP under the covers. For for instance "foo"="bar", the "foo" is translated, but the value becomes"\xa1\x03bar".

When a Kafka endpoint is a dataflow source, Kafka user headers are translated to MQTT v5 properties. The following table describes how Kafka user headers are translated to MQTT v5 properties.


| Kafka Header | Translated Behavior |
|---------------|----------|
| Key | Key: "Key" <BR> Value: Copy of the Key from original message. |
| Timestamp | Key: "Timestamp" <BR> Value: UTF-8 encoding of Kafka Timestamp, which is number of milliseconds since Unix epoch.

Kafka user header key/value pairs - provided they're all encoded in UTF-8 - are directly translated into MQTT user key/value properties.

###### UTF-8 / Binary Mismatches

MQTT v5 can only support UTF-8 based properties. If dataflow receives a Kafka message that contains one or more non-UTF-8 headers, dataflow will:

* Remove the offending property or properties.
* Forward the rest of the message on, following the previous rules.

Applications that require binary transfer in Kafka Source headers => MQTT Target properties must first UTF-8 encode them - for example, via Base64.

###### >=64KB property mismatches

MQTT v5 properties must be smaller than 64 KB. If dataflow receives a Kafka message that contains one or more headers that is >= 64KB, dataflow will:

* Remove the offending property or properties.
* Forward the rest of the message on, following the previous rules.

###### Property translation when using Event Hubs and producers that use AMQP

If you have a client forwarding messages a Kafka dataflow source endpoint doing any of the following actions:

- Sending messages to Event Hubs using client libraries such as *Azure.Messaging.EventHubs*
- Using AMQP directly

There are property translation nuances to be aware of.

You should do one of the following:

- Avoid sending properties
- If you must send properties, send values encoded as UTF-8.

When Event Hubs translates properties from AMQP to Kafka, it includes the underlying AMQP encoded types in its message. For more information on the behavior, see [Exchanging Events Between Consumers and Producers Using Different Protocols](https://github.com/Azure/azure-event-hubs-for-kafka/tree/master/tutorials/interop).

In the following code example when the dataflow endpoint receives the value `"foo":"bar"`, it receives the property as `<0xA1 0x03 "bar">`.

```csharp
using global::Azure.Messaging.EventHubs;
using global::Azure.Messaging.EventHubs.Producer;

var propertyEventBody = new BinaryData("payload");

var propertyEventData = new EventData(propertyEventBody)
{
  Properties =
  {
    {"foo", "bar"},
  }
};

var propertyEventAdded = eventBatch.TryAdd(propertyEventData);
await producerClient.SendAsync(eventBatch);
```

The dataflow endpoint can't forward the payload property `<0xA1 0x03 "bar">` to an MQTT message because the data isn't UTF-8. However if you specify a UTF-8 string, the dataflow endpoint translates the string before sending onto MQTT. If you use a UTF-8 string, the MQTT message would have `"foo":"bar"` as user properties. 

Only UTF-8 headers are translated. For example, given the following scenario where the property is set as a float:

```csharp
Properties = 
{
  {"float-value", 11.9 },
}
```

The dataflow endpoint discards packets that contain the `"float-value"` field.

Not all event data properties including propertyEventData.correlationId are not forwarded. For more information, see [Event User Properties](https://github.com/Azure/azure-event-hubs-for-kafka/tree/master/tutorials/interop#event-user-properties),

### CloudEvents

[CloudEvents](https://cloudevents.io/) are a way to describe event data in a common way. The CloudEvents settings are used to send or receive messages in the CloudEvents format. You can use CloudEvents for event-driven architectures where different services need to communicate with each other in the same or different cloud providers.

The `CloudEventAttributes` options are `Propagate` or`CreateOrRemap`. For example:

```yaml
mqttSettings:
  CloudEventAttributes: Propagate # or CreateOrRemap
```

#### Propagate setting

CloudEvent properties are passed through for messages that contain the required properties. If the message doesn't contain the required properties, the message is passed through as is. If the required properties are present, a `ce_` prefix is added to the CloudEvent property name.

| Name              | Required | Sample value                                           | Output name          | Output value                                                                |
| ----------------- | -------- | ------------------------------------------------------ | -------------------- |---------------------------------------------------------------------------- |
| `specversion`     | Yes      | `1.0`                                                  | `ce-specversion`     | Passed through as is                                                        |
| `type`            | Yes      | `ms.aio.telemetry`                                     | `ce-type`            | Passed through as is                                                        |
| `source`          | Yes      | `aio://mycluster/myoven`                               | `ce-source`          | Passed through as is                                                        |
| `id`              | Yes      | `A234-1234-1234`                                       | `ce-id`              | Passed through as is                                                        |
| `subject`         | No       | `aio/myoven/telemetry/temperature`                     | `ce-subject`         | Passed through as is                                                        |
| `time`            | No       | `2018-04-05T17:31:00Z`                                 | `ce-time`            | Passed through as is. It's not restamped.                                   |
| `datacontenttype` | No       | `application/json`                                     | `ce-datacontenttype` | Changed to the output data content type after the optional transform stage. |
| `dataschema`      | No       | `sr://fabrikam-schemas/123123123234234234234234#1.0.0` | `ce-dataschema`      | If an output data transformation schema is given in the transformation configuration, `dataschema` is changed to the output schema.  |

#### CreateOrRemap setting

CloudEvent properties are passed through for messages that contain the required properties. If the message doesn't contain the required properties, the properties are generated.

| Name              | Required | Output name          | Generated value if missing                                                    |
| ----------------- | -------- | -------------------- | ------------------------------------------------------------------------------|
| `specversion`     | Yes      | `ce-specversion`     | `1.0`                                                                         |
| `type`            | Yes      | `ce-type`            | `ms.aio-dataflow.telemetry`                                                   |
| `source`          | Yes      | `ce-source`          | `aio://<target-name>`                                                         |
| `id`              | Yes      | `ce-id`              | Generated UUID in the target client                                           |
| `subject`         | No       | `ce-subject`         | The output topic where the message is sent                                    |
| `time`            | No       | `ce-time`            | Generated as RFC 3339 in the target client                                    |
| `datacontenttype` | No       | `ce-datacontenttype` | Changed to the output data content type after the optional transform stage    |
| `dataschema`      | No       | `ce-dataschema`      | Schema defined in the schema registry                                         |

---