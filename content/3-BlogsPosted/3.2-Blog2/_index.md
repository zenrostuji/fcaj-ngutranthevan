---
title: "Running a Web App on AWS Nitro Enclaves"
date: 2026-07-01
weight: 2
chapter: false
pre: " <b> 3.2 </b> "
---

## Running a traditional web application on AWS Nitro Enclaves without modifying source code

Hi everyone in AWS Study Group VN!

While learning about AWS Nitro Enclaves, I came across an interesting AWS article about how to move traditional web applications into an Enclave environment with almost no changes to the application's source code.

Because the original document was published on AWS China, I read, translated, and summarized the main points along the way. If anything is inaccurate or missing, I'd really appreciate your feedback.

### What are Nitro Enclaves?

Nitro Enclaves is an isolated compute environment created from an EC2 instance through the AWS Nitro Hypervisor.

The main goal of this service is to process highly sensitive data such as:

- Encryption keys
- Financial data
- Healthcare data
- User identity information
- Workloads requiring a high level of security

What makes Nitro Enclaves special is that the environment is almost fully isolated:

- No IP address
- No internet connection
- No direct SSH access
- No persistent storage

All communication with the Enclave must go through VSOCK and the parent EC2 instance. This is exactly what gives Nitro Enclaves its high level of security, but it also makes moving existing applications into an Enclave more difficult.

### The challenge of migrating web applications

Most web applications today run over TCP/IP through HTTP or HTTPS. Nitro Enclaves, however, does not support conventional network communication and only uses VSOCK.

If deployed directly, developers would have to modify a fair amount of source code to switch the entire communication mechanism from TCP/IP to VSOCK. For systems already running stably or for long-lived applications, this is both time-consuming and risky, because it can easily affect the application's existing logic.

### The solution AWS proposes

What I found interesting in the article is that AWS proposes using **SOCAT** as an intermediate proxy layer to convert between HTTP and VSOCK.

The model uses two proxies:

- A proxy on the parent EC2 that receives external HTTP connections and converts them to VSOCK to send into the Enclave.
- A proxy inside the Enclave that receives VSOCK and converts it back to HTTP for the application to handle.

![Two-layer SOCAT proxy architecture between EC2 and the Nitro Enclave](images/socat-proxy-architecture.jpg)

The reverse direction works the same way. If the application inside the Enclave needs to reach out, the proxy converts traffic from HTTP to VSOCK so the parent EC2 can perform network access on the Enclave's behalf.

Thanks to this approach, the application can run almost as if it were on a normal Linux server, with virtually no changes to its processing logic.

### Deployment workflow

According to the AWS article, the deployment workflow can be summarized as follows:

1. Install the Nitro Enclaves CLI and Developer Tools.
2. Build the application into a Docker image.
3. Convert the Docker image into an EIF (Enclave Image File).
4. Launch the Enclave using the Nitro CLI.
5. Deploy the two SOCAT proxies on the parent EC2 and inside the Enclave.
6. Access the application through the parent instance; all traffic is forwarded into the Enclave via VSOCK.

![Packaging the app into a Docker image, then building it into an Enclave Image File with the Nitro CLI](images/docker-to-eif.jpg)

### What I found interesting

What I appreciate about this approach is that AWS does not require developers to rewrite the entire application just because of a different communication mechanism. Instead, adding a proxy layer lets you reuse existing applications, significantly cutting the cost and time of moving a system into a more secure environment.

It is also a nice example of combining open-source tools like SOCAT with AWS services to solve a compatibility problem while preserving the isolation and security of Nitro Enclaves.

Of course, this model has a few things to watch, such as proxy-layer performance, traffic monitoring, and how to design the overall architecture to fit each system. So before applying it in production, you should still evaluate carefully according to each project's needs.

### Conclusion

In my opinion, Nitro Enclaves is a fairly interesting AWS service if you're studying security or building systems that process sensitive data.

This article also shows a practical direction: instead of rewriting the whole application to adapt to the Enclave, you can leverage a protocol-conversion layer to significantly reduce migration effort while keeping the security Nitro Enclaves provides.

This is my first time researching and summarizing this topic. If there are gaps in my translation or research, I'd love your feedback so I can improve.

### References

- AWS China Blog – Running Traditional Web Application Migration Practices in AWS Nitro Enclaves (original document in Chinese): <https://aws.amazon.com/cn/blogs/china/running-traditional-web-application-migration-practices-in-aws-nitro-enclaves/>
