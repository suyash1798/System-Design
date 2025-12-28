# Scalability Basics

## Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)
- Increase resources on a single server (e.g., CPU 8 → 32 cores, RAM 16GB → 128GB)
- Simple to implement
- Limited by hardware capacity
- Causes downtime in many cases
- Creates a single point of failure
- Example: upgrading a DB server from t3.large → m5.8xlarge

### Horizontal Scaling (Scale Out)
- Add more servers and distribute load
- Requires load balancers and stateless services
- More complex to implement
- High availability
- Scales almost infinitely
- Example: adding more API servers behind a load balancer

### Cost considerations
- Vertical scaling may be cheaper initially
- Horizontal scaling is better long-term
- At scale, horizontal scaling is the only viable option

## Latency vs Throughput

### Latency
- Time taken to process one request
- Measured in milliseconds
- Important for user-facing systems
- Examples:
  - Online gaming
  - Video calls
  - Search results

### Throughput
- Amount of work done per unit time
- Measured as requests/sec, jobs/sec
- Important for bulk systems
- Examples:
  - Log processing
  - Batch analytics
  - Data pipelines

Note: You usually trade one for the other.

## Single Point of Failure (SPOF)

### What it is
- Any component whose failure can bring down the entire system

### Examples
- Single DB instance
- Single load balancer
- Single cache node

### How to avoid SPOF
- Replication (DB replicas, cache replicas)
- Redundancy (multiple instances)
- Horizontal scaling
- Health checks and failover

> "Every critical component should have at least one backup."
