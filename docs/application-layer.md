# Application Layer

The application layer is the layer where the Unique platform, its dependencies and the LLM models are deployed.

## Unique Platform Dependencies

### Postgresql

All of our stateful services use Postgresql as their database. We provision a single Postgresql cluster with three nodes with the help of the Stackgres operator. The cluster is configured with automated hourly backups to a Ceph object storage bucket, a dedicated distributed logs server and init scripts to bootstrap the databases and the user roles.

### Qdrant

To store vector embeddings we use Qdrant. We deploy a single Qdrant instance without replication. Managing database clusters is complicated and we try to keep the complexity of the infrastructure minimal. All data stored within the Qdrant database can be easily restored from the Postgres database in the event of a failure.

### Redis

Redis is used as a cache backend for pub sub messaging when sending stream updates to the frontend. An example are the messages that contain the stream chunks when a LLM is generating text. We stream those to the frontend, so the user can see the text being generated.

### RabbitMQ

RabbitMQ is our central message broker for all event based communication in the Unique platform. We deploy a single instance cluster with the help of the RabbitMQ operator. A single instance is easy to manage and offers high throughput and low latency for our event based communication. However, updating the broker requires a maintenance window and the system is not highly available.

However, this is at the moment our preferred operating mode and runs very stable in our production environments. We recommend waiting with the rollout of a more complex HA cluster until this is really necessary for your use case.

## Large Language Models

Our example setup allows you to deploy LLMs via the Nvidia NIM platform and operator or via other hosting tools like vLLM.

The Unique platform requires two AI models for its operation:

- An Embedding model: At the time of writing we recommend the `llama-3.2-nv-embedqa-1b-v2` embedding model served via Nvidia NIM.
- An LLM model: At the time of writing we recommend the `llama-3.3-70b` LLM model served via Nvidia NIM.

## Unique Backend Services

We deploy the following backend services:

- Chat
- Ingestion
- Ingestion Worker & Chat Worker
- Scope Management
- Theme
- App Repo
- Webhook Scheduler & Worker
- Debug (A echo service to health check the cluster and ingress infrastructure)

## Unique Frontend

The Unique frontend consists of several web apps:

- Chat
- Knowledge Upload
- Admin
- Theme





