# System Design Framework — Quick-Fire Questions

*Short answers. Ideal for rapid revision before an interview.*

---

**Q1: What are the two types of requirements in system design?**
Functional (what the system does) and Non-functional (how it performs — latency, availability, scale).

---

**Q2: How do you calculate QPS from DAU?**
`QPS = (DAU × requests_per_user_per_day) / 86,400`

---

**Q3: A system has 10M DAU, each making 5 requests/day. What's the QPS?**
`(10M × 5) / 86,400 ≈ 578 QPS`

---

**Q4: What is p99 latency?**
The response time within which 99% of all requests complete. The slowest 1% takes longer.

---

**Q5: What is the typical read:write ratio for most web apps?**
~100:1 (read-heavy systems).

---

**Q6: What does 99.9% availability mean in downtime per year?**
~8.7 hours of downtime per year.

---

**Q7: What is the first thing you should do when given a system design problem?**
Clarify requirements — both functional and non-functional. Never start drawing without asking questions.

---

**Q8: Name 3 non-functional requirements you should always ask about.**
Scale (QPS/DAU), Latency (p99 target), Availability (SLA %).

---

**Q9: What's the difference between vertical and horizontal scaling?**
Vertical = bigger machine (more CPU/RAM). Horizontal = more machines.

---

**Q10: When would you use a message queue in your design?**
When you need async processing, decoupling between services, or to absorb bursty write traffic.

---

**Q11: What is a stateless server?**
A server that doesn't store any session data locally — each request contains all needed context (e.g., via JWT).

---

**Q12: Why is statelessness important for scalability?**
Because any server instance can handle any request — easy to add/remove instances behind a load balancer.

---

**Q13: What should you always consider that most candidates forget?**
Failure handling / fault tolerance, and observability (logging, metrics, tracing).

---

**Q14: What's the difference between throughput and latency?**
Latency = time to complete one request. Throughput = how many requests/sec the system handles.

---

**Q15: Name 3 standard components in almost every HLD.**
Load balancer, Cache (Redis), and a persistent Database.

---

**Q16: What is back-of-envelope estimation?**
Rough calculation to size key system properties (storage, QPS, bandwidth) using simple math — used to justify design decisions.

---

**Q17: How much storage does 1 million photo uploads per day require? (avg 200KB each)**
`1M × 200KB = 200GB/day`

---

**Q18: What is an SLA?**
Service Level Agreement — a commitment on availability, latency, or other metrics (e.g., 99.9% uptime).

---

**Q19: Push model vs pull model — which is better for timelines?**
Neither is universally better. Push (fanout on write) is fast to read but expensive for celebrities. Pull (fanout on read) is simple to write but slow to read. Hybrid is the real-world answer.

---

**Q20: What does "scope the problem" mean in an interview?**
Agree on which features are in vs out of scope so you don't spend time designing things that don't matter for the problem.
