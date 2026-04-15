# API Design — Quick-Fire Q&A

**Format:** Answer each question in ≤2 sentences before checking the answer.

---

**Q1. What does it mean for an HTTP method to be "safe"?**
A safe method does not modify server state — it only reads. GET and HEAD are safe; POST, PUT, PATCH, and DELETE are not.

---

**Q2. What does it mean for an HTTP method to be "idempotent"?**
Calling an idempotent method multiple times produces the same result as calling it once. GET, PUT, and DELETE are idempotent; POST is not (calling POST twice may create two resources).

---

**Q3. What status code do you return when a resource is successfully created?**
201 Created, with a `Location` response header pointing to the URI of the newly created resource.

---

**Q4. What is the difference between 401 and 403?**
401 Unauthorized means the client is not authenticated — it has no (or invalid) credentials. 403 Forbidden means the client is authenticated but does not have permission to access the resource.

---

**Q5. Why is cursor-based pagination preferred over offset pagination in production?**
Offset pagination performs a full `OFFSET N` scan (expensive at large N) and returns inconsistent results when rows are inserted/deleted during paging. Cursor pagination uses an index-friendly seek on the last seen ID, is stable under concurrent writes, and has O(1) cost per page.

---

**Q6. What is the N+1 problem in GraphQL and how do you solve it?**
The N+1 problem is when resolving a list of N items fires one query for the list plus N individual queries for a nested field — one per item. It is solved with DataLoader, which batches all nested field resolution within a single request tick into one database query.

---

**Q7. What are the four streaming modes in gRPC?**
Unary (one request, one response), Server Streaming (one request, stream of responses), Client Streaming (stream of requests, one response), and Bidirectional Streaming (both sides stream concurrently).

---

**Q8. What is the main advantage of Protobuf over JSON for API payloads?**
Protobuf is a binary format with a strict schema, producing payloads 30–60% smaller than equivalent JSON and faster to serialize/deserialize. It also enforces a contract via `.proto` files and enables code generation for both client and server.

---

**Q9. What does an idempotency key solve?**
It solves the problem of non-idempotent operations (like POST /payments) being retried by the client after a network timeout — the server recognises the duplicate key and returns the original response instead of processing the request again, preventing double charges or duplicate records.

---

**Q10. Which status code should a server return when a rate limit is exceeded, and what header should accompany it?**
429 Too Many Requests, accompanied by a `Retry-After` header specifying how many seconds the client must wait before retrying.

---

**Q11. When is it safe to retry a POST request without an idempotency key?**
Almost never. Without an idempotency key, retrying a POST may create duplicate resources or charge a card twice. Only retry a POST without a key if the server has documented it as idempotent (e.g., a POST search endpoint with no side effects).

---

**Q12. What is the difference between PUT and PATCH?**
PUT replaces the entire resource — you must send all fields, and any omitted fields are cleared/reset. PATCH applies a partial update — you send only the fields you want to change.

---

**Q13. What is a "breaking change" in an API?**
Any change that causes existing clients to stop working correctly: removing or renaming a field, changing a field's type, removing an endpoint, changing the HTTP method, or changing status codes in a way that alters error handling logic.

---

**Q14. What is the `Sunset` header used for in API versioning?**
The `Sunset` header (RFC 8594) communicates to clients when a deprecated endpoint will be shut down permanently, giving them a deadline to migrate to the newer version.

---

**Q15. What should you return from a DELETE endpoint for a resource that has already been deleted?**
204 No Content — DELETE is idempotent, so deleting an already-deleted resource should succeed silently rather than returning 404, because the desired end state (resource not existing) is already achieved.

---

**Q16. GraphQL vs REST: when does GraphQL cause more problems than it solves?**
For simple CRUD APIs with well-defined data shapes, GraphQL adds complexity (schema, resolvers, DataLoader) for no gain. It also requires query complexity limiting to prevent denial-of-service via deeply nested queries, and its flexible query model makes HTTP caching harder (everything is POST to a single endpoint).

---

**Q17. Why should API keys never be placed in URLs?**
URLs are logged by web servers, proxies, CDNs, and browser history in plaintext. An API key in a URL (`?api_key=secret`) is trivially leaked through any of these channels. API keys must go in the `Authorization` header or a custom header (`X-API-Key`).

---

**Q18. What is a "named action" endpoint and when do you use it?**
A named action endpoint is a POST to a sub-path that represents a state machine transition that doesn't fit CRUD — e.g., `POST /orders/42/cancel` or `POST /payments/99/refund`. It is used when the action has side effects and doesn't map cleanly to PUT/PATCH of a single field.

---

**Q19. What is exponential backoff with jitter and why is jitter necessary?**
Exponential backoff waits `base * 2^attempt` between retries. Jitter adds randomness to this delay. Without jitter, all clients that hit a rate limit simultaneously will retry at the same moment, creating a thundering herd that immediately re-triggers the rate limit.

---

**Q20. What HTTP method is correct for an endpoint that searches with a complex body (e.g., a multi-field filter)?**
Technically, GET with query params is preferred (searches should be safe and idempotent). However, when the filter is too complex for query params, `POST` to a search endpoint is acceptable (`POST /orders/search`) and must be documented as idempotent with no side effects. Some APIs use the non-standard `QUERY` method (proposed RFC).
