# Distributed Media Processing Pipeline

A Python microservices prototype that accepts video uploads, queues conversion work, converts video audio to MP3, stores media in MongoDB GridFS, and notifies users through an asynchronous event workflow.

> **Project status:** Local Kubernetes / Docker-oriented prototype. The repository contains service code and Kubernetes manifests, but the current implementation has important security and reliability gaps. It should not be presented as production-grade without those issues being addressed.

## Services

- **Gateway:** Flask API for login, upload, and MP3 download. It validates a JWT through the auth service and stores uploaded videos in GridFS.
- **Auth:** Flask service backed by MySQL that issues and validates JWTs.
- **Converter:** RabbitMQ consumer that retrieves a video from GridFS, converts its audio using MoviePy, stores the MP3 in GridFS, and publishes a completion event.
- **Notification:** RabbitMQ consumer that sends completion notifications through SMTP.
- **Message broker:** RabbitMQ with Kubernetes manifests for broker deployment and storage.

## Technology

- Python, Flask, PyJWT.
- RabbitMQ / Pika.
- MongoDB, PyMongo, GridFS.
- MySQL / Flask-MySQLdb.
- MoviePy for video/audio conversion.
- Docker and Kubernetes (Minikube manifests).

## Processing flow

```text
Client
  │
  ▼
Flask Gateway ──► MongoDB GridFS (video)
  │
  └──► RabbitMQ video queue
             │
             ▼
       Converter worker ──► GridFS (MP3)
             │
             └──► RabbitMQ MP3 queue
                         │
                         ▼
                  Notification worker
                         │
                         ▼
                       Email
```

## Repository layout

- `microservices/gateway/` — upload/download API and Kubernetes manifests.
- `microservices/auth/` — login/JWT validation API, SQL initialization, and manifests.
- `microservices/converter/` — RabbitMQ consumer and MoviePy conversion module.
- `microservices/notification/` — notification consumer and email sender.
- `microservices/rabbit/` — RabbitMQ StatefulSet, service, and configuration manifests.
- `setup.md` — local Minikube deployment walkthrough.
- `python/github_descriptions.md` — repository description notes.

## Run and deploy

The documented deployment target is Minikube with Docker, kubectl, RabbitMQ, MySQL, and MongoDB reachable from the cluster.

Follow `setup.md` for the intended deployment flow, but note that its example directory paths should be checked against the actual repository layout under `microservices/`. Create service-specific environment configuration from the manifests and requirements files. Do not commit database passwords, JWT secrets, or SMTP app passwords.

## Important implementation limitations

- The auth service compares the submitted password directly with the stored database password; it does not hash/verify passwords with a password-hashing algorithm. This is unsafe for real accounts.
- Gateway, converter, and notification code reads/writes complete media payloads in memory in several places. Do not claim constant-memory streaming for the current conversion path.
- The converter consumer calls `basic_nack` without enabling requeue. A failed conversion is therefore not guaranteed to be retried; notification failures explicitly request requeue.
- The services use fixed internal hostnames / MongoDB addresses in code. Configure them for the target Kubernetes network before deployment.
- The README's throughput and latency claims should be treated as local demonstration measurements, not general capacity guarantees. Reproduce them with the referenced workload before quoting them.
- Review RabbitMQ queue durability, dead-letter handling, idempotency, file cleanup, upload limits, and retry/backoff behavior before production use.
- There is no evidence in the current tree of a comprehensive automated end-to-end test suite.

## Further reading

See `setup.md` for the deployment walkthrough.

## License

See repository licensing and third-party dependency terms.
