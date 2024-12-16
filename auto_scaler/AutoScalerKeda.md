# Horizontal Autoscaling of Kubernetes Kafka Applications using KEDA External Scaler Service

## Problem Statement

As modern cloud-native applications evolve, efficient resource management becomes crucial for ensuring optimal performance and cost-efficiency. Kubernetes (K8s) has become the de facto orchestration platform for containerized workloads, offering powerful features like dynamic scaling and high availability. However, managing the autoscaling of Kubernetes applications can be challenging, especially as traffic patterns fluctuate, resources grow, and infrastructure complexity increases.

The challenge lies in dynamically adjusting the number of pods in response to changing load conditions (e.g., CPU, memory usage, or custom metrics) while maintaining application performance, resource utilization, and cost efficiency. Without a proper autoscaling mechanism, applications can face under-provisioning, leading to poor performance and latency, or over-provisioning, resulting in unnecessary costs due to idle resources.

The objective is to implement an intelligent and efficient autoscaling solution that automatically adjusts the number of application pods to match real-time traffic and resource consumption, ensuring that the application remains highly available, responsive, and cost-effective.

### Key Challenges:
1. **Dynamic Scaling**: Ensuring the system can scale up or down based on real-time demand without human intervention.
2. **Resource Utilization**: Avoiding both over-provisioning (which wastes resources) and under-provisioning (which leads to degraded performance).
3. **Performance Consistency**: Maintaining consistent application performance during sudden traffic spikes or drops.
4. **Cost Efficiency**: Ensuring that autoscaling decisions optimize both resource usage and operational costs.
5. **Complex Metrics**: Handling autoscaling decisions based on not just CPU or memory, but also custom application-specific metrics (e.g., request latency, queue length, business-critical metrics).

The goal is to build an autoscaling mechanism that can intelligently adjust Kubernetes workloads in real-time, ensuring optimal application performance while minimizing cost and complexity.

## What is KEDA?

KEDA is a Kubernetes-based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.

KEDA is a single-purpose and lightweight component that can be added into any Kubernetes cluster. KEDA works alongside standard Kubernetes components like the Horizontal Pod Autoscaler and can extend functionality without overwriting or duplication. With KEDA you can explicitly map the apps you want to use event-driven scale, with other apps continuing to function. This makes KEDA a flexible and safe option to run alongside any number of any other Kubernetes applications or frameworks.

### How KEDA works

KEDA performs three key roles within Kubernetes:

1. **Agent** — KEDA activates and deactivates Kubernetes Deployments to scale to and from zero on no events. This is one of the primary roles of the keda-operator container that runs when you install KEDA.
2. **Metrics** — KEDA acts as a Kubernetes metrics server that exposes rich event data like queue length or stream lag to the Horizontal Pod Autoscaler to drive scale out. It is up to the Deployment to consume the events directly from the source. This preserves rich event integration and enables gestures like completing or abandoning queue messages to work out of the box. The metric serving is the primary role of the keda-operator-metrics-apiserver container that runs when you install KEDA.
3. **Admission Webhooks** - Automatically validate resource changes to prevent misconfiguration and enforce best practices by using an admission controller. As an example, it will prevent multiple ScaledObjects to target the same scale target.

## KEDA built-in Kafka scaler

1. Off the shelf Kafka scalers are not sufficient to auto scale the Kafka applications since it uses one metric, either;
   - Whole consumer group lag threshold
   - A Specified topic of a consumer group lag threshold
   - Specific partition(s) of a specified topic of a consumer group 
2. Doesn’t support average/max lag per partition
3. Lag threshold may not be a good metric. Few Downsides of Kafka consumer lag
   - **Scaling delay**: The lag in Kafka consumers may not reflect the need for scaling immediately
   - **Overreacting to short-term spikes**: Consumer lag might fluctuate quickly due to temporary bursts of data
   - **Lag as a symptom, not a cause**: Kafka consumer lag is usually a symptom of underlying issues like:
      - Slow processing within the consumer application.
      - Network latency between the Kafka brokers and the consumers.
      - Scaling based on lag might mask the root cause of the problem instead of addressing it. For example, if the consumer is slow due to inefficient code or inadequate resource allocation, simply adding more pods might not improve throughput effectively.

## KEDA External Scaler

While KEDA ships with a set of built-in scalers, users can also extend KEDA through a GRPC service that implements the same interface as the built-in scalers.

Built-in scalers run in the KEDA process/pod, while external scalers require an externally managed GRPC server that’s accessible from KEDA with optional TLS authentication. KEDA itself acts as a GRPC client and it exposes similar service interface for the built-in scalers, so external scalers can fully replace built-in ones.

### External Scaler GRPC interface
[External Scaler GRPC interface Details](https://keda.sh/docs/2.15/concepts/external-scalers/#external-scaler-grpc-interface)

## Using KEDA External Scaler Service for Autoscaling of K8s Kafka Applications
### Architecture Diagram
![Autoscaler Architecture Diagram](https://pradeep-ganachari.github.io/blogs/auto_scaler/autoscaler.png)
1. **Resource Management Service (External Scaler)**
   Resource Management Service is GRPC service that extends KEDA by implementing the same interface as the built-in scalers. This is an externally managed GRPC server which is accessible from KEDA.
   This service has mainly 3 modules
   - ***Collector***: Collector module is responsible for collecting the required metrics from metrics server (eg: prometheus) periodically
   - ***Recommender***: Recommender module is where the algorithm to calculate the number of pods runs and it uses the metrics collected by collector module to run the algorithm
   - ***Executor***: Executor module is the one which decides whether or when to apply the recommended number of pods using KEDA/HPA by returning pod count to periodic calls from KEDA. 
2. **KEDA**
   KEDA will attempt a GRPC connection to Resource Management Service immediately after reconciling the ScaledObject.
   It will then make the following RPC calls:
   - ***IsActive***: KEDA does an initial call to IsActive followed by one call on each pollingInterval
   - ***StreamIsActive***: KEDA does an initial call and the scaler is expected to maintain a long-lived connection (called a stream in GRPC terminology). The external push scaler can then send an IsActive event back to KEDA at any time. KEDA will only attempt another call to StreamIsActive if it needs to re-connect
   - ***GetMetricsSpec***: KEDA will do an initial call with the following data in the incoming ScaledObjectRef parameter:
   - ***GetMetrics***: KEDA will call this method every pollingInterval to get the point-in-time metric values for the names returned by GetMetricsSpec.

   Every periodic call to Resource Management Service, it gets a pod count for the target K8s Kafka application.
3. **Application Pods**
   Application pods are the Kafka applications that need to scaled up/down based on the current load in the system.
4. **Metric Server**
   Metrics server can be any thing as in Prometheus, KPOW, or Kafka broker itself which gives the required Kafka metrics for a give application.

### Algorithm

Algorithm to calculate the required number of replicas for K8s Kafka application depends on below 2 metrics
- **throughput of a single pod**: Number of messages per second from Kafka topics a single pod can consume and process. Its a config parameter given by each application via KEDA scaledObject to Resource Management Service (External Scaler)
- **Kafka topics**: Application has to provide a list of Kafka topics it is reading from via KEDA scaledObject to Resource Management Service (External Scaler)
- **Incoming message rate**: Number of messages per second being written to Kafka topics from which application is consuming messages. This incoming rate is read from metric server (example: Prometheus)

***Desired Pod Count = IncomingMessageRate/Throughput***

### Example Scaled Object
![Example Scaled Object](https://pradeep-ganachari.github.io/blogs/auto_scaler/ScaledObjectExample.png)

### Advantages
1. All the downsides of autoscaling based on Kafka consumer lag are addressed
2. Will have the explanability as and when the no. of replicas change since Resource Management Service returns the direct pod count to KEDA