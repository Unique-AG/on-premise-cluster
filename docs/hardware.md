# Hardware

The three primary components of the cluster are **Unique microservices, AI models, and Kubernetes**. Since RAM and CPUs are relatively affordable, we recommend **oversizing the servers** to allow the application to scale for a larger user base. The **biggest limiting factor** will be GPU power, which depends on the specific models you intend to use with Unique. For Kubernetes, we opted for a **lightweight distribution** that requires minimal resources, making its impact on the system negligible.

## Test Cluster

For a test cluster, we recommend the following setup with **two servers**:

- **Application Server & Kubernetes Control Plane:**

  - 1x AMD EPYC 9474F (48 cores)
  - 768GB RAM
  - 1x RAID 1 480GB Boot NVMe
  - 2x 7.68TB NVMe Data Drives
  - 10/25 Gbit Networking

- **GPU Server:**

  - 2x AMD EPYC 9354 (32 cores each)
  - 384GB RAM
  - 1x RAID 1 480GB Boot NVMe
  - 2x 7.68TB NVMe Data Drives
  - 10/25 Gbit Networking
  - 2x Nvidia H100 NVL (94GB VRAM each)

- **Network Security:**
  - Hardware Firewall (e.g., Sophos XGS 2100) or Layer 3 Switch

This test cluster allows you to run a full Unique installation. However, it lacks redundancy and cannot ensure high availability. The application server manages Unique’s microservices and the Kubernetes overhead for this small cluster.

The system is installed on the Boot NVMe, ideally configured in RAID 1 to prevent data loss. The two data drives are utilized by Ceph for block, file, and object storage, automatically handling replication, eliminating the need for a traditional RAID system or dedicated hardware RAID controller for the data layer.

The network card allows future scalability into larger clusters where high bandwidth is critical. Ceph also benefits from fast networking for efficient file replication.

The GPU server requires less RAM than the application server but still needs sufficient capacity, as Nvidia NIMs and VLLM require both CPU and GPU RAM. The NVMe drives are shared with the application server as part of the Ceph cluster. The 10/25 Gbit network ensures a fast data layer.

At the time of writing, the two Nvidia H100 NVL GPUs allow us to run a LLaMA 3.1 70B model (full precision, no quantization) alongside a reranker and an embedding model, with Nvidia GPU time-slicing enabled.

## Production Cluster

For a production cluster, we recommend the following setup with at least 8 servers:

- **3x Kubernetes Control Plane Servers:**
  - 1x AMD EPYC 9124 (16 cores) or similar
  - 64GB RAM
  - 1x RAID 1 480GB Boot NVMe
  - 10/25 Gbit Networking

- **3x Application Servers:**
  - 1x AMD EPYC 9354 (32-cores) or 9474F (48-cores)
  - min. 384GB RAM, better 768GB RAM
  - 1x RAID 1 480GB Boot NVMe
  - 2x 7.68TB NVMe Data Drives
  - 10/25 Gbit Networking

- **2x GPU Servers:**
  - 2x AMD EPYC 9354 (32 cores each)
  - 384GB RAM
  - 1x RAID 1 480GB Boot NVMe
  - 2x 7.68TB NVMe Data Drives
  - 10/25 Gbit Networking
  - 4x Nvidia H100 NVL (94GB VRAM each)
  - *(optional)* 100Gb / 400Gb NICs if large models should be served across all GPUs in both servers

- **Network & Security:**
  - 2x Hardware Firewalls or Layer 3 Switches with HA failover setup for DSN
  - OOBM Switch
  - 25Gb Switch
  - *(optional)* 100Gb / 400Gb Switch for GPU networking

Using dedicated servers for the control plane ensures that Kubernetes’ internal load does not affect the application layer. It also allows for tighter firewall rules and better network isolation. The application servers can be configured with pod anti-affinity rules to ensure that unique services remain available even if a server goes down or requires maintenance. Additionally, redundancy enhances data security, as the Ceph cluster maintains multiple replicas.

Using two GPU servers primarily serves the purpose of redundancy. While this setup can also be leveraged to serve large models (e.g., 600B+ parameter models) across all GPUs in both servers, doing so requires additional networking hardware and reduces redundancy again. In such cases we recommend using the Nvidia DGX platform.

## Nvidia DGX platform

If your model requirements exceed the capabilities of the cluster defined above, you have the option to use the Nvidia DGX platform. These pre-configured servers include:
	•	8x H100 or 8x H200 GPUs
	•	CPUs, RAM, and NVMe storage
	•	Seamless integration into existing infrastructure
	•	Support for running the latest AI models

Such systems cost approximately 500K–600K CHF.