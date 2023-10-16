- Streaming Data - RedPanda
- Time Series API - Druid
- HPC MOnitoring API
- SPUE - Writing data into Kafka

# Lessons

- Encapsulate as much of the deployment logic as possible.
- Understand the hardware backing the cluster.
- Storage and networking.

# User Limiting

- Namespaces plus Kubernetes policies.

# Storage

- Ceph Storage, but they encountered network problems.
- Kubernetes plus a local Ceph instance provided an improvement.
- RedPanda is going to be moved outside of Kubernetes

# Routing

- Configure Kubernetes to intercept DNS?
