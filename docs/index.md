---
title: "Free PostgreSQL Course: psql, MVCC, Indexes, EXPLAIN, JSONB, Partitioning, Replication, Backups and Tuning"
description: "Free PostgreSQL course: 40 lessons run on PostgreSQL 18 — MVCC, VACUUM, indexes, EXPLAIN, JSONB, RLS, replication, PITR and production operations."
---

# PostgreSQL Mastery Path

<div class="mp-hero-lockup">
  <div class="mp-hero-globe">
    <svg viewBox="0 0 160 160" class="mp-globe-svg mp-mark">
      <circle cx="80" cy="80" r="74" fill="var(--md-default-bg-color, #FAF7EF)" stroke="#2FA98D" stroke-width="5"/>
      <rect class="mp-bar mp-bar-1" x="36" y="90" width="17" height="24" rx="5" fill="#2FA98D"/>
      <rect class="mp-bar mp-bar-2" x="59" y="74" width="17" height="40" rx="5" fill="#7FC9B7"/>
      <rect class="mp-bar mp-bar-3" x="82" y="56" width="17" height="58" rx="5" fill="#E0B457"/>
      <rect class="mp-bar mp-bar-4" x="105" y="38" width="17" height="76" rx="5" fill="#C97B4A"/>
      <circle class="mp-dot" cx="44.5" cy="82" r="7" fill="#22261f"/>
    </svg>
  </div>
  <div class="mp-hero-divider"></div>
  <div class="mp-hero-text">
    <div class="mp-hero-title">Mastery Path</div>
    <div class="mp-hero-sub">BOOTCAMP</div>
    <div class="mp-hero-rule"></div>
    <div class="mp-hero-byline">VINOD BHANUJA</div>
  </div>
</div>

PostgreSQL is the database people reach for when they want one system to do a great deal: strict
transactions, rich types, JSON documents, full-text search, geospatial and vector extensions,
partitioning, replication — all open source, all in one server. It rewards understanding. The same
database that serves one team effortlessly will bloat, lock up or lose data for another, and the
difference is almost never the hardware. It is whether someone on the team knows what an `UPDATE`
really writes, why a forgotten transaction stops VACUUM, which `ALTER TABLE` takes the whole table
offline, and how to get back to the second before someone ran `DELETE` without a `WHERE`.

This course teaches that understanding. It is not an SQL course — the
[SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers queries, joins and
aggregation, and this course assumes them. Here the subject is **PostgreSQL itself**: how it stores and
versions rows, how it chooses plans, how it locks, logs and replicates, and how to design, tune, secure
and operate it.

**Level 1** covers the foundations of working with PostgreSQL rather than generic SQL: the server's
process model and data directory, `psql`, the data types that matter, schemas and `search_path`, roles
and `pg_hba.conf`, constraints that enforce rules under concurrency, isolation levels reproduced with
real concurrent sessions, upserts and `MERGE`, and logical backups — ending with a ticketing database
that survives 40 simultaneous buyers fighting for 5 seats. **Level 2** opens the engine: MVCC row
versions read straight off disk pages, VACUUM and wraparound, TOAST and HOT updates, B-tree, GIN, GiST,
BRIN and hash indexes, reading `EXPLAIN ANALYZE`, planner statistics, locking and deadlocks, and the
write-ahead log — ending with a slow help-desk database taken from 29 to over 22,000 transactions per
second. **Level 3** covers the advanced features: JSONB, full-text search, PL/pgSQL, triggers and audit
logs, LATERAL and recursive queries, partitioning, row-level security, extensions including pgvector,
and a job queue built on `SKIP LOCKED` — ending with a multi-tenant SaaS backend. **Level 4** is
production: configuration, connection pooling with PgBouncer, streaming and logical replication,
point-in-time recovery, monitoring, zero-downtime migrations, security hardening, upgrades and high
availability — ending with a hardened primary and replica proven by a recovery drill and a failover
drill.

Every lesson has a **How It Actually Works** section — what a snapshot is and how a row decides whether
you can see it, why a rolled-back delete leaves its transaction ID behind, why the first change after a
checkpoint writes a whole page to the log, why a cast on a partition key disables pruning, how a
replication slot can fill a disk, and why `pg_upgrade` can upgrade a terabyte in minutes.

!!! note "Versions and what was actually run"
    Every command, query and script in this course was run on **PostgreSQL 18.6** (Homebrew build on
    macOS), and every output shown is the output it produced — including the errors, which are some of
    the most useful parts. Supporting tools: psycopg 3.3 for multi-session demonstrations, PgBouncer
    1.26, pgvector 0.8.7, and PostgreSQL 17.11 for the upgrade lesson. Timings and throughput numbers
    come from a laptop and are there to show proportions, not to benchmark hardware; where a lesson
    compares numbers, both sides were measured the same way on the same machine.

    Features new in PostgreSQL 16, 17 and 18 are marked as such. Things that were not run — managed
    cloud services, Patroni and other failover managers, pgBackRest and similar backup tools, PostGIS,
    TimescaleDB — are described, never shown with invented output.

## How the program is organized

| Level | Focus | Modules | What you can do after |
|-------|-------|---------|------------------------|
| **1 · PostgreSQL Foundations** | Server, psql, types, schemas, roles, constraints, isolation, backups | 10 | Design a schema where the database enforces your rules, and back it up |
| **2 · Internals: MVCC, Storage & Indexes** | MVCC, VACUUM, storage, index types, EXPLAIN, statistics, locks, WAL | 10 | Diagnose and fix slow queries and bloat from first principles |
| **3 · Advanced Features** | JSONB, search, PL/pgSQL, triggers, partitioning, RLS, extensions, queues | 10 | Build rich backends without bolting on extra systems |
| **4 · Operations & Production** | Tuning, pooling, replication, PITR, monitoring, migrations, security, upgrades | 10 | Run PostgreSQL in production and recover when things go wrong |

Each level has 9 teaching modules plus a hands-on project (module 10), for **40 lessons** total.

## How to use this site

- **Run your own server.** Level 1 · 01 shows how to create a throwaway cluster in a directory you can
  delete. Several lessons need superuser access or several servers on different ports — a laptop is
  ideal; a shared company database is not.
- **Reproduce the outputs.** Your transaction IDs, timings and sizes will differ; the behaviour should
  not. When something differs in kind, find out why — that is usually where the learning is.
- **Do the concurrency experiments with two sessions.** Isolation, locking, queues and replication only
  make sense when you watch two things happen at once.
- **Do the projects.** Each one combines its level into something you could adapt for real work, and
  each was tested end to end, failures included.

## More from the Mastery Path series

Free, structured, module-wise training across 97 other languages, platforms and disciplines:

<div class="mastery-grid-wrap">
<p class="mastery-grid-category">Languages</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/python-mastery-path/">🐍 Python</a>
  <a href="https://sigilipelli.github.io/java-mastery-path/">☕ Java</a>
  <a href="https://sigilipelli.github.io/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://sigilipelli.github.io/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://sigilipelli.github.io/csharp-mastery-path/">🔵 C#</a>
  <a href="https://sigilipelli.github.io/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://sigilipelli.github.io/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://sigilipelli.github.io/c-mastery-path/">🇨 C</a>
  <a href="https://sigilipelli.github.io/cpp-mastery-path/">➕ C++</a>
  <a href="https://sigilipelli.github.io/go-mastery-path/">🐹 Go</a>
  <a href="https://sigilipelli.github.io/rust-mastery-path/">🦀 Rust</a>
  <a href="https://sigilipelli.github.io/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://sigilipelli.github.io/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://sigilipelli.github.io/php-mastery-path/">🐘 PHP</a>
  <a href="https://sigilipelli.github.io/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://sigilipelli.github.io/swift-mastery-path/">🐦 Swift</a>
  <a href="https://sigilipelli.github.io/dart-mastery-path/">🎯 Dart</a>
  <a href="https://sigilipelli.github.io/scala-mastery-path/">🔴 Scala</a>
  <a href="https://sigilipelli.github.io/r-mastery-path/">📊 R</a>
  <a href="https://sigilipelli.github.io/matlab-mastery-path/">🟧 MATLAB</a>
</div>
<p class="mastery-grid-category">Web Frameworks</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/react-mastery-path/">⚛️ React</a>
  <a href="https://sigilipelli.github.io/nextjs-mastery-path/">▲ Next.js</a>
  <a href="https://sigilipelli.github.io/angular-mastery-path/">🅰️ Angular</a>
  <a href="https://sigilipelli.github.io/vue-mastery-path/">💚 Vue.js</a>
  <a href="https://sigilipelli.github.io/html-css-mastery-path/">🎨 HTML & CSS</a>
  <a href="https://sigilipelli.github.io/tailwind-mastery-path/">🌬️ Tailwind CSS</a>
  <a href="https://sigilipelli.github.io/django-mastery-path/">🎸 Django</a>
  <a href="https://sigilipelli.github.io/fastapi-mastery-path/">🚀 FastAPI</a>
  <a href="https://sigilipelli.github.io/graphql-mastery-path/">🕸️ GraphQL</a>
  <a href="https://sigilipelli.github.io/nodejs-mastery-path/">🟩 Node.js</a>
  <a href="https://sigilipelli.github.io/spring-boot-mastery-path/">🍃 Spring Boot</a>
</div>
<p class="mastery-grid-category">Mobile</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/flutter-mastery-path/">💙 Flutter</a>
  <a href="https://sigilipelli.github.io/react-native-mastery-path/">📲 React Native</a>
</div>
<p class="mastery-grid-category">Testing & QA</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/java-testing-mastery-path/">🧪 Java Testing</a>
  <a href="https://sigilipelli.github.io/cpp-testing-mastery-path/">🧪 C/C++ Testing</a>
  <a href="https://sigilipelli.github.io/python-testing-mastery-path/">🧪 Python Testing</a>
  <a href="https://sigilipelli.github.io/automotive-testing-mastery-path/">🚗 Automotive Testing</a>
  <a href="https://sigilipelli.github.io/selenium-mastery-path/">🧭 Selenium</a>
  <a href="https://sigilipelli.github.io/cypress-mastery-path/">🌲 Cypress</a>
</div>
<p class="mastery-grid-category">Security</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/cybersecurity-mastery-path/">🛡️ Cybersecurity</a>
  <a href="https://sigilipelli.github.io/ethical-hacking-mastery-path/">🎯 Ethical Hacking</a>
</div>
<p class="mastery-grid-category">Cloud Platforms</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/aws-mastery-path/">☁️ AWS</a>
  <a href="https://sigilipelli.github.io/azure-mastery-path/">☁️ Azure</a>
  <a href="https://sigilipelli.github.io/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://sigilipelli.github.io/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://sigilipelli.github.io/adobe-mastery-path/">🎨 Adobe</a>
</div>
<p class="mastery-grid-category">Data & Analytics</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/data-engineering-mastery-path/">🛠️ Data Engineering</a>
  <a href="https://sigilipelli.github.io/data-science-mastery-path/">📈 Data Science</a>
  <a href="https://sigilipelli.github.io/tableau-mastery-path/">📊 Tableau</a>
  <a href="https://sigilipelli.github.io/excel-mastery-path/">📗 Excel</a>
  <a href="https://sigilipelli.github.io/pyspark-mastery-path/">⚡ PySpark</a>
  <a href="https://sigilipelli.github.io/etl-datalake-mastery-path/">🏞️ ETL & Data Lake</a>
  <a href="https://sigilipelli.github.io/power-bi-mastery-path/">📉 Power BI</a>
</div>
<p class="mastery-grid-category">Databases</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/mongodb-mastery-path/">🍃 MongoDB</a>
  <a href="https://sigilipelli.github.io/redis-mastery-path/">🟥 Redis</a>
  <a href="https://sigilipelli.github.io/snowflake-mastery-path/">❄️ Snowflake</a>
  <a href="https://sigilipelli.github.io/databricks-mastery-path/">🧱 Databricks</a>
</div>
<p class="mastery-grid-category">AI / ML / LLM</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://sigilipelli.github.io/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://sigilipelli.github.io/rag-mastery-path/">📚 RAG</a>
  <a href="https://sigilipelli.github.io/edge-ai-mastery-path/">📱 Edge AI</a>
  <a href="https://sigilipelli.github.io/claude-training-mastery-path/">🔶 Claude Training</a>
  <a href="https://sigilipelli.github.io/ai-tools-mastery-path/">🧰 AI Tools</a>
  <a href="https://sigilipelli.github.io/ml-math-mastery-path/">➗ ML Math Foundations</a>
  <a href="https://sigilipelli.github.io/prompt-engineering-mastery-path/">✍️ Prompt Engineering</a>
  <a href="https://sigilipelli.github.io/ai-agents-mastery-path/">🕹️ AI Agents</a>
  <a href="https://sigilipelli.github.io/mlops-mastery-path/">🔁 MLOps</a>
  <a href="https://sigilipelli.github.io/computer-vision-mastery-path/">👁️ Computer Vision</a>
</div>
<p class="mastery-grid-category">Embedded Systems</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://sigilipelli.github.io/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://sigilipelli.github.io/embedded-python-mastery-path/">🐍 Embedded Python</a>
  <a href="https://sigilipelli.github.io/freertos-mastery-path/">⏱️ FreeRTOS</a>
  <a href="https://sigilipelli.github.io/s32k-mastery-path/">🔧 S32K</a>
</div>
<p class="mastery-grid-category">Leadership & Management</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/product-manager-mastery-path/">📋 Product Manager</a>
  <a href="https://sigilipelli.github.io/product-lead-mastery-path/">🧭 Product Lead</a>
  <a href="https://sigilipelli.github.io/project-manager-mastery-path/">📅 Project Manager</a>
  <a href="https://sigilipelli.github.io/ai-manager-mastery-path/">🤖 AI Manager</a>
  <a href="https://sigilipelli.github.io/servant-leadership-mastery-path/">🤝 Servant Leadership</a>
</div>
<p class="mastery-grid-category">Professional Skills</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/english-fluency-mastery-path/">🗣️ English Fluency & IELTS</a>
  <a href="https://sigilipelli.github.io/workday-mastery-path/">🧑‍💼 Workday</a>
</div>
<p class="mastery-grid-category">Careers & Interviews</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/dsa-interview-mastery-path/">🧩 DSA & Coding Interviews</a>
  <a href="https://sigilipelli.github.io/system-design-mastery-path/">🏗️ System Design</a>
</div>
<p class="mastery-grid-category">Process & APIs</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/agile-mastery-path/">🔄 Agile/Scrum/Kanban</a>
  <a href="https://sigilipelli.github.io/rest-api-mastery-path/">🔗 REST API</a>
  <a href="https://sigilipelli.github.io/playwright-mastery-path/">🎭 Playwright</a>
</div>
<p class="mastery-grid-category">Infrastructure & Ops</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/server-ops-mastery-path/">🖥️ Server Ops</a>
  <a href="https://sigilipelli.github.io/linux-mastery-path/">🐧 Linux</a>
  <a href="https://sigilipelli.github.io/nodemcu-mastery-path/">📶 NodeMCU/IoT</a>
  <a href="https://sigilipelli.github.io/terraform-mastery-path/">🌍 Terraform</a>
  <a href="https://sigilipelli.github.io/kubernetes-mastery-path/">☸️ Kubernetes</a>
  <a href="https://sigilipelli.github.io/docker-mastery-path/">🐳 Docker</a>
  <a href="https://sigilipelli.github.io/github-mastery-path/">🐙 GitHub & Git</a>
  <a href="https://sigilipelli.github.io/cicd-mastery-path/">🔁 CI/CD</a>
  <a href="https://sigilipelli.github.io/ansible-mastery-path/">🤖 Ansible</a>
  <a href="https://sigilipelli.github.io/kafka-mastery-path/">📨 Kafka</a>
  <a href="https://sigilipelli.github.io/microservices-mastery-path/">🧩 Microservices</a>
  <a href="https://sigilipelli.github.io/observability-mastery-path/">📈 Observability</a>
</div>
</div>

