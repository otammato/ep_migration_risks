# ep_migration_risks

## Capacity and performance risks

- **WLM queue contention.** Stored procedures and analytical queries compete for the same workload management slots. Scaling the cluster adds compute but doesn't eliminate contention - it just raises the ceiling before contention reappears.
- **Unpredictable query latency for existing consumers.** Analytics queries that currently meet SLAs may start missing them intermittently as the CDC pipeline runs, with no clean way to isolate cause.
- **H1 objectives at risk.** Your committed deliverables depend on cluster capacity that's already allocated. Adding a second workload means either slippage, scope reduction, or forced reprioritization.
- **No headroom for growth.** Even if the scaled cluster absorbs both workloads today, there's no buffer for data volume growth, new consumers, or seasonal spikes. You'll be back at this conversation in six months.
- **Concurrency limits.** Redshift has hard limits on concurrent queries per WLM queue. A CDC pipeline running on a cadence consumes slots that would otherwise serve users.
- **Vacuum and maintenance windows disrupted.** Heavy write activity from CDC affects vacuum, analyze, and sort key maintenance - which in turn degrades query performance over time in ways that are hard to attribute.

## Architectural risks

- **Collapsed failure domains.** Source, transform, and target are normally independent systems. Collapsing them means a failure in any layer becomes a failure in all layers. The warehouse itself becomes a single point of failure for ingestion.
- **Blast radius expansion.** Today, if ingestion tooling fails, Redshift keeps serving queries. Under the proposed design, a bad stored procedure, a Lambda retry storm, or a malformed payload can degrade or take down the warehouse - affecting every downstream consumer, not just the CDC job.
- **Reimplementation of CDC primitives in SQL.** Schema drift handling, checkpointing, replay, ordering guarantees, dead-letter queues, and idempotency are all handled natively by tools like DMS, Glue, and Debezium. Implementing them in stored procedures means building a CDC engine from scratch, badly, and owning it forever.
- **Loss of independent scalability.** You can no longer scale ingestion independently of query workload. Every capacity decision is now a joint decision affecting both.
- **No clean rollback path.** Stored procedures triggered by Lambda are harder to version, test, and roll back than pipeline configurations in a managed ETL tool.
- **Debugging complexity.** When something breaks, the boundary between "pipeline problem" and "warehouse problem" disappears. Root cause analysis gets slower and harder.
- **Observability gaps.** Managed ETL tools come with built-in metrics, logs, and lineage. A custom stored-procedure pipeline requires building all of that yourself, or going without.

## Operational risks

- **Ongoing maintenance burden.** Someone has to own the custom CDC pipeline indefinitely - patching stored procedures, updating Lambda code, handling Azure SQL DB schema changes, responding to incidents. That someone is likely to be you, by default, because it touches your cluster.
- **On-call implications.** If the pipeline breaks at 3am, the person paged is whoever owns the warehouse - again, likely you - regardless of who wrote the CDC logic.
- **Knowledge concentration.** The Azure developer owns the pipeline logic. If they leave, get reassigned, or become unavailable, the organization has a custom CDC pipeline in production that nobody fully understands.
- **Incident severity inflation.** Because ingestion and warehouse share infrastructure, incidents that would have been "pipeline is lagging" become "warehouse is down." Severity levels, escalation paths, and stakeholder communications all shift upward.
- **Deployment coupling.** Changes to the pipeline require changes to the warehouse. Deployment cadences, approval processes, and change windows now have to be coordinated across what were previously independent systems.
- **Testing difficulty.** You can't easily test the pipeline in isolation from the warehouse, or the warehouse in isolation from the pipeline. Staging environments become more expensive and harder to keep representative.

## Cost risks

- **Scaling cost not in original analysis.** The original cost comparison that favored Azure SQL DB assumed the pipeline would run on the existing cluster. Scaling changes that math, and the updated numbers may or may not still favor the current path.
- **Lost AWS credits.** Scaling Redshift to absorb the CDC workload consumes credits faster, or forces choices between this workload and others that would have used them. Credits that expire unused are a direct financial loss.
- **Hidden engineering cost.** Building and maintaining the custom CDC pipeline costs engineering hours that don't appear in any infrastructure bill but are real. Off-the-shelf tools cost more in license fees but less in labor.
- **Cost of future migration.** If you later need to move to a proper ETL architecture, you'll pay the migration cost then - plus the cost of whatever data correctness issues accumulated in the interim.
- **Opportunity cost.** Every hour spent maintaining a custom CDC pipeline is an hour not spent on your H1 objectives or other higher-value work.

## Strategic and lock-in risks

- **Azure SQL DB lock-in.** Once the migration from MS SQL lands on Azure SQL DB, moving to Azure SQL Managed Instance later is a second full migration, not an upgrade. The window to choose Managed Instance cheaply is now, before migration completes.
- **Tooling lock-in.** A custom stored-procedure CDC pipeline is not portable. If you later change warehouses, change source databases, or adopt a different ingestion strategy, the pipeline has to be rewritten from scratch.
- **Precedent risk.** Accepting this design sets a precedent that ETL logic belongs inside the warehouse. Future projects may follow the same pattern, compounding the problems.
- **Vendor relationship risk.** Forfeiting AWS credits or underusing AWS managed services may affect the organization's standing with AWS, future credit allocations, or support relationships.

## Data and correctness risks

- **Ordering guarantees.** CDC requires strict ordering for correctness. Custom stored-procedure implementations often get this subtly wrong under failure conditions, producing silent data corruption that's only noticed weeks later.
- **Duplicate handling.** Idempotency and deduplication are hard. Managed tools handle them; custom implementations frequently have edge cases.
- **Schema drift handling.** When the source schema changes, managed CDC tools detect and adapt. A stored-procedure pipeline typically breaks, and breaks in ways that may not be immediately visible.
- **Backfill and replay.** Reprocessing historical data after a bug fix or failure is a first-class feature in DMS/Glue. In a custom pipeline, it's a project.
- **Transactional consistency.** Guaranteeing that a set of related changes land together, or not at all, is a solved problem in CDC tools and a hard problem in stored procedures.
- **Silent data loss.** The combination of the above risks means data loss may occur without anyone noticing until a downstream consumer reports incorrect numbers - at which point attribution and recovery are both expensive.

## Organizational and personal risks

- **Ownership ambiguity.** When incidents happen, the line between "Azure developer's pipeline" and "your warehouse" will be contested. You may end up owning problems you didn't create.
- **Reputation risk if you're overruled and stay silent.** If the predicted problems occur and there's no written record that you flagged them, the organizational memory will place responsibility on whoever was closest to the incident - likely you.
- **Reputation risk if you fight too hard.** Equally, continuing to push back after a decision is made will mark you as difficult, regardless of whether you're right. The written-note-and-execute approach protects you from both failure modes.
- **Burnout and morale.** Maintaining a system you believe is architecturally wrong, while also hitting your committed objectives, is exhausting in a way that accumulates. This is a real risk to you personally, and it affects your work quality over time.
- **Career trajectory.** If your objectives slip because of capacity absorbed by someone else's project, that shows up in your performance review regardless of the reason. Make sure the reason is documented in writing, in real time, so the review conversation has context.

## Risks of the mitigations themselves

Worth naming, because intellectual honesty matters and your boss may raise these:

- **Scaling the cluster *might* actually work** if the CDC workload is small and bursty. You can't know for certain without running it longer than you did. Your observed data is strong but not exhaustive.
- **Managed Instance + DMS/Glue has its own operational cost.** It's not free. It requires DMS task management, schema mapping, and its own monitoring. The comparison is "different operational burden," not "no operational burden."
- **The Azure developer may have context you don't.** Timeline pressure, a commitment upstream, or constraints from the migration project that make their choice more defensible than it appears. Worth asking directly rather than assuming.
