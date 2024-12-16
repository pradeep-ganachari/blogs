# Intelligent Rate Limiter for TCP Traffic with Sidecar Proxy: A Solution for Database Stability

## Problem Statement

In modern distributed systems, applications frequently interact with databases to store and retrieve data. However, one common challenge is handling unexpected spikes in traffic from clients, often due to a few rogue devices or misbehaving applications. These traffic bursts can overwhelm the database, leading to poor read/write performance and potentially destabilizing the entire system.

The objective of this approach is to implement a system that enforces rate limiting on TCP traffic between applications and databases. By limiting excessive data writes, we can prevent rogue clients from degrading the performance and stability of the entire database infrastructure.

## Abstract: Enhancing Stability with a Sidecar Proxy

One of the most powerful deployment patterns for modern cloud-native applications is the **sidecar proxy** model. This pattern allows you to augment existing systems with new features—such as rate limiting—without requiring any changes to the underlying application code. The sidecar proxy runs alongside the main application service, intercepting and managing network traffic.

In this case, we’ll employ a **sidecar proxy** (such as **Envoy Proxy**) in combination with **WebAssembly (Wasm) plugins** to implement intelligent rate limiting for TCP traffic between applications and databases. This setup enables granular control over data traffic, ensuring the database remains stable while providing dynamic handling of traffic spikes.

## TCP Data Rate Limiting: A Control Mechanism for Stability

TCP data rate limiting is designed to manage the number of data transfers and connections between an application and a database. It establishes thresholds for the number of bytes an application can send or receive per unit of time, ensuring the database is not overwhelmed by excessive requests. Typical thresholds might be something like **100K bytes per minute** or **5K bytes per second**.

By implementing intelligent rate limiting, we can:

- Prevent sudden bursts of data from overwhelming the database.
- Safeguard database performance during periods of high traffic.
- Maintain overall system stability and reliability.

### Key Requirements

To effectively limit TCP traffic and maintain database performance, the proposed solution must fulfill the following criteria:

1. **Dynamic Burst Detection**: The system must detect sudden, unexpected bursts of traffic in real-time and identify abnormal spikes.
  
2. **Application Pausing**: If a burst is detected, the system should automatically pause the affected application’s ability to read/write data to/from the database, ensuring that the database is not overwhelmed.

3. **Resumption of Normal Operation**: Once traffic levels return to normal, the application should be able to resume its operations without issues.

## The Algorithm: Intelligent Rate Limiting with Envoy Proxy and Wasm

### Overview

To implement intelligent rate limiting, we will use an **Envoy Proxy** with a **Wasm plugin**. This combination enables us to dynamically enforce rate limits for each application pod, responding to traffic patterns in real-time. Below is a step-by-step flow of the algorithm used to implement the rate-limiting mechanism.

### Step-by-Step Flow

1. **Initialization**:
   - The Wasm plugin initializes within each application pod running Envoy Proxy.
   - Rate-limiting parameters are configured, including the threshold values for the bytes sent/received and the number of active TCP connections.

2. **Monitoring**:
   - Envoy Proxy continuously tracks outbound bytes sent to the downstream service (database) and inbound bytes received.
   - It also monitors the number of active TCP connections established between the application and the database.

3. **Threshold Check**:
   - Periodically, the Wasm plugin checks whether the application has exceeded the pre-configured thresholds for data transmission (in bytes) or the number of TCP connections.
   - If the thresholds are exceeded, the system proceeds to the next step.

4. **Blocking Decision**:
   - Based on the current rate-limiting conditions, the system determines whether to block or pause the application’s traffic.
   - If a block is necessary, the application is paused from sending or receiving data to/from the database, preventing the system from becoming overwhelmed.

5. **Monitoring Normalcy**:
   - After blocking, the system continuously monitors traffic levels to detect when normal operation has resumed.
  
6. **Unblocking Decision**:
   - Once normal traffic patterns are restored, the system decides whether to unblock the application and allow it to resume database interactions.

This dynamic process ensures that applications are rate-limited in real time based on actual traffic conditions, rather than static thresholds or fixed rules.

## Block Diagram
![TCP ratelimit Architecture](https://pradeep-ganachari.github.io/blogs/tcp_ratelimit/TcpRateLimitArch.png)

## Sequence Diagram
![TCP ratelimit Sequence Diagram](https://pradeep-ganachari.github.io/blogs/tcp_ratelimit/TcpRateLimitSeq.png)

## Benefits of Using a Wasm Plugin with Envoy Proxy for Rate Limiting

The combination of **Envoy Proxy** and **WebAssembly** for rate limiting TCP traffic offers several advantages:

### 1. **Granular Control**
   - Instead of blocking entire applications or services, only the problematic pods of an application are blocked. This means other parts of the application can continue operating normally, minimizing overall system disruption.

### 2. **Dynamic Blocking**
   - The blocking mechanism is dynamic and adaptive. Pods are only blocked when necessary, and they are unblocked as soon as traffic levels return to normal, ensuring that performance bottlenecks are addressed swiftly.

### 3. **Scalability**
   - This approach scales effectively as the application grows. Each pod can independently enforce its own rate-limiting policies, ensuring performance is not compromised even as the number of deployed pods increases.

### 4. **Customization**
   - Wasm plugins are highly customizable, allowing you to fine-tune rate-limiting policies according to specific application needs, workload patterns, or business requirements.

### 5. **Real-time Adaptation**
   - The rate-limiting algorithm adapts to real-time traffic conditions, allowing the system to respond to fluctuations in traffic patterns and ensure continued stability.

### 6. **Isolation of Issues**
   - With the ability to isolate individual problematic pods, troubleshooting and debugging become more manageable. Developers can quickly identify the root cause of issues and resolve them without disrupting the entire system.

## Conclusion

In an era where application traffic can be unpredictable, intelligent rate limiting provides a powerful mechanism to ensure database stability. By leveraging a sidecar proxy architecture with Envoy and a Wasm plugin, we can dynamically monitor and limit TCP traffic in real-time, effectively addressing traffic bursts while maintaining system performance.

This solution enables scalability, flexibility, and reliability, making it ideal for modern cloud-native applications that require real-time traffic management without compromising on performance.

With this approach, you can safeguard your database from rogue traffic, ensure seamless application performance, and build a resilient infrastructure capable of handling even the most unpredictable traffic conditions.