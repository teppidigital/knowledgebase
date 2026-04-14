# Real-World Scaling — Production War Stories

A collection of documented scaling challenges from high-traffic production systems, organised as case studies. Each entry covers the initial architecture, the specific failure mode or bottleneck encountered, the solution applied, and transferable engineering lessons.

**Format per case study:**
- Scale context (numbers at the time of the problem)
- Initial architecture (what broke)
- The problem (failure mode, bottleneck)
- The solution (changes made, step by step)
- Key learnings (generic lessons for any project)
- Architecture diagram (Mermaid)
- Code / config sample (runnable reference)
- References (original engineering blog posts and papers)

---

## Case Studies

| # | Company | Problem | Technology | Scale |
|---|---------|---------|------------|-------|
| 01 | [OpenAI — PostgreSQL at 800M Users](./01-openai-postgresql.md) | MVCC write amplification, connection storms, cache miss cascades | PostgreSQL, PgBouncer, Redis | 800M users, millions QPS |
| 02 | [Discord — Messages to ScyllaDB](./02-discord-scylladb.md) | Cassandra hot partitions, GC pauses | Cassandra → ScyllaDB, Rust | Billions of messages |
| 03 | [Twitter — Timeline Fan-Out](./03-twitter-timeline-fanout.md) | Celebrity fan-out write storms | Redis, Memcached | 500M tweets/day |
| 04 | [Stripe — Rate Limiting & Idempotency](./04-stripe-rate-limiting.md) | Retry storms, duplicate charges under failure | Redis, token bucket, idempotency keys | Millions of API requests/day |
| 05 | [Uber — MySQL to Schemaless](./05-uber-schemaless.md) | Single MySQL bottleneck for all writes | MySQL sharding, consistent hashing | 10M+ trips/day |
| 06 | [Netflix — Thundering Herd & EVCache](./06-netflix-thundering-herd.md) | Cache invalidation storms, cold start floods | Memcached (EVCache), probabilistic early expiry | 230M subscribers |
| 07 | [Shopify — MySQL Sharding with Vitess](./07-shopify-vitess.md) | Single MySQL cluster bottleneck, online DDL blocking | MySQL, Vitess, VSchema | Millions of merchants |
| 08 | [WhatsApp — 2M Connections Per Server](./08-whatsapp-connections.md) | C10k problem at 10× scale | Erlang, FreeBSD, lightweight actors | 2B users |
| 09 | [Pinterest — Sharding from Day One](./09-pinterest-sharding.md) | Unshardable schema bought too late | MySQL, Redis, consistent hashing | 250M+ objects |
| 10 | [GitHub — MySQL High Availability](./10-github-mysql-ha.md) | Manual failover, replication lag, schema migration locks | MySQL, Orchestrator, ProxySQL, gh-ost | Millions of repositories |
| 11 | [Cloudflare — Distributed Rate Limiting](./11-cloudflare-rate-limiting.md) | Sliding window rate limiting across 200+ edge nodes globally | Nginx, Riak, approximate sliding window | Trillions of requests/month |
| 12 | [Slack — Job Queue Evolution](./12-slack-job-queue.md) | Redis queue saturation, job loss, priority inversion | Redis, MySQL, Kafka, priority queues | Millions of workspaces |

---

## Recurring Failure Modes

Across all case studies, a small set of failure patterns appear repeatedly:

| Failure Mode | Appears In | Root Cause |
|-------------|-----------|-----------|
| **Thundering herd / cache stampede** | OpenAI, Netflix, Twitter | Multiple clients simultaneously miss on the same cache key following an invalidation or restart |
| **Retry storm** | OpenAI, Stripe | Timeouts under load trigger retries that amplify the original surge |
| **Single writer bottleneck** | OpenAI, Uber, GitHub | Single-primary designs cannot scale writes horizontally |
| **Hot partition / hot shard** | Discord, Pinterest | Uneven shard key distribution concentrates load |
| **Connection exhaustion** | OpenAI, GitHub | Database connection limits reached under traffic burst |
| **GC / stop-the-world pauses** | Discord | JVM-based storage engines (Cassandra) pause under heap pressure |
| **Fan-out write amplification** | Twitter | Writing to N followers on each write; fails for high-follower accounts |
| **Online schema change blocking** | Shopify, GitHub | ALTER TABLE locks block production traffic |
| **Priority inversion in queues** | Slack | Low-priority jobs consume all workers, starving high-priority ones |

---

## Cross-Cutting Lessons

1. **Measure before you scale** — every case involved instrumenting the system and finding the actual bottleneck, not guessing
2. **Connection pooling is table stakes** — PgBouncer, ProxySQL, or a similar proxy belongs in front of every production database
3. **Caching requires a stampede mitigation strategy** — cache stampede is not hypothetical; every large system encounters it
4. **Shard key selection is a one-way door** — choose carefully when the schema is simple, before migrating is painful
5. **Single points of failure create correlated blast radius** — isolate workloads; a slow query from one feature should not degrade another
6. **Retries must be bounded with backoff and jitter** — unbounded retries turn a partial failure into a full outage
7. **Schema migration must be online or invisible** — never ALTER TABLE in production without an online migration tool
8. **Languages and runtimes matter for I/O-bound services** — GC pauses (JVM), async model (Erlang/Go), and memory layout (Rust) have real production consequences

---

## References

- [OpenAI Engineering Blog](https://openai.com/blog/engineering)
- [Discord Engineering Blog](https://discord.com/blog/engineering)
- [Twitter Engineering Blog](https://blog.twitter.com/engineering)
- [Stripe Engineering Blog](https://stripe.com/blog/engineering)
- [Uber Engineering Blog](https://www.uber.com/blog/engineering/)
- [Netflix Tech Blog](https://netflixtechblog.com/)
- [Shopify Engineering Blog](https://shopify.engineering/)
- [GitHub Engineering Blog](https://github.blog/engineering/)
- [Cloudflare Blog](https://blog.cloudflare.com/)
- [Slack Engineering Blog](https://slack.engineering/)
