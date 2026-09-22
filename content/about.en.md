---
title: "About"
layout: "page"
url: "/about/"
summary: "about"
---

I'm Ilya, a DevOps engineer based in Moscow, with a bit over five years of
production experience that sits somewhere between classic DevOps and
database administration — a lot of the work I actually get pulled into is
"the cluster is unhappy" rather than "the pipeline is broken," and over
time I've ended up doing both.

## Where I've worked

**Wildberries** (Jan 2025 – present) — DevOps engineer on infrastructure
supporting high-load production clusters: PostgreSQL, Cassandra, ClickHouse,
MongoDB, Redis. Day to day this means query profiling and optimization,
troubleshooting under real load, managing cluster topology, and backup
strategy — plus writing and maintaining Ansible roles/playbooks to get rid
of manual steps in environment provisioning. I also spent a lot of time
revising alerting and logging (VictoriaMetrics, Grafana, OpenSearch) to cut
false positives and shorten the time between "something's wrong" and
"someone's looking at it," and on Kubernetes resource efficiency —
autoscaling (HPA/VPA/KEDA) and Kyverno policies to standardize
requests/limits instead of leaving them to guesswork, which was previously
causing both over-reservation and OOM kills.

**Voximplant** (Jan 2024 – Jan 2025) — DevOps engineer. Migrated Kubernetes
services for better scalability, built and optimized cross-regional cloud
infrastructure across AWS/Yandex Cloud/VK Cloud with Terraform, wrote and
maintained Ansible automation, set up monitoring (Grafana + VictoriaMetrics)
and logging (Vector + OpenSearch + Kibana). Also administered a RabbitMQ
cluster, including federation configuration for cross-regional queue
synchronization, and worked on scaling and optimizing PostgreSQL, ClickHouse
and MongoDB.

**DevsVault** (Feb 2022 – Jan 2024) — DevOps engineer. Built and maintained
Docker/Kubernetes-based infrastructure, wrote Helm charts for microservice
deployments, cut deployment time by introducing IaC with GitLab CI + ArgoCD,
set up monitoring with Zabbix & Graylog, and administered Apache Kafka
(topics, partitions, replication, performance and fault tolerance tuning).

**Aurus** (Mar 2021 – Jan 2022) — Linux system administrator. Built and
maintained Linux-based server infrastructure (CentOS, Debian) and network
equipment using Ansible, automated processes with bash scripts, monitored
and troubleshot servers.

## The DBA side of the job

A meaningful chunk of what I do in practice looks less like classic DevOps
and more like database administration under production load: reading
`EXPLAIN` plans and fixing the actual cause instead of just the symptom,
tuning replication topology, managing backup/restore strategy for
PostgreSQL and Cassandra clusters, keeping ClickHouse performant as data
volume grows, and generally being the person who gets paged when a
database — not a deployment — is the thing on fire.

## Education

National Research Nuclear University MEPhI, information systems and
programming, class of 2024.

## Stack

Linux · Docker · Kubernetes · Helm · Ansible · Terraform · GitLab CI ·
PostgreSQL · ClickHouse · MongoDB · Apache Cassandra · Kafka · RabbitMQ ·
Prometheus · Grafana · VictoriaMetrics · Zabbix · ELK/OpenSearch

## Contact

- Email: [nazarovilia.mifi@gmail.com](mailto:nazarovilia.mifi@gmail.com)
- Telegram: [@ilianazarovv](https://t.me/ilianazarovv)
