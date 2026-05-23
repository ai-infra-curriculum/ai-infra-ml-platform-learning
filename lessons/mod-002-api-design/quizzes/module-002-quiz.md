# Module 02: API Design — Quiz

- **Total Questions**: 20
- **Time Limit**: 30 minutes
- **Passing Score**: 75%

### 1. The best choice for an internal feature-lookup API serving 100K+ RPS is:
- [ ] a) REST + JSON
- [x] b) gRPC + protobuf
- [ ] c) GraphQL
- [ ] d) Async events only

### 2. Idempotency keys are most important on:
- [x] a) POST / PUT requests that create or modify state
- [ ] b) GET requests
- [ ] c) DELETE requests on read-only resources
- [ ] d) Health-check endpoints

### 3. The recommended default versioning scheme for new platform APIs is:
- [x] a) URL versioning (`/v1/...`)
- [ ] b) Custom header versioning
- [ ] c) Query parameter versioning
- [ ] d) No versioning; evolve in place

### 4. Within a major version, which change is NOT breaking?
- [ ] a) Removing a field from a response
- [ ] b) Changing a field's type
- [x] c) Adding an optional field to a request
- [ ] d) Tightening validation rules

### 5. Cursor pagination is preferred over offset pagination because:
- [x] a) It tolerates list mutations between page fetches
- [ ] b) It's required by REST
- [ ] c) It's always faster
- [ ] d) It supports sorting better

### 6. Async pattern is appropriate when:
- [x] a) The operation takes longer than the caller wants to wait
- [ ] b) Latency is critical
- [ ] c) The caller is a human
- [ ] d) Always

### 7. Spec-first development means:
- [x] a) Writing the OpenAPI YAML before any server code
- [ ] b) Generating the spec from the server code
- [ ] c) Versioning the API
- [ ] d) Skipping the spec entirely

### 8. A "Sunset" HTTP header tells consumers:
- [x] a) The date this version will be removed
- [ ] b) The last time the response was modified
- [ ] c) When their session expires
- [ ] d) The server's local sunset time (literal)

### 9. The minimum deprecation timeline for an externally-customer-facing API breaking change should be:
- [ ] a) 30 days
- [ ] b) 90 days
- [ ] c) 180 days
- [x] d) 365 days (one year)

### 10. Field-level evolution is acceptable for changes that are:
- [x] a) Purely additive
- [ ] b) Always
- [ ] c) Breaking
- [ ] d) Removed in a future version

### 11. Contract testing tools like Schemathesis are used to:
- [x] a) Generate property-based tests from your spec
- [ ] b) Mock external dependencies
- [ ] c) Load-test endpoints
- [ ] d) Build SDKs

### 12. A generated low-level SDK should be wrapped by hand to add:
- [x] a) Retry, pagination iterator, auth helper, idempotency key generation
- [ ] b) Nothing; generated is sufficient
- [ ] c) Database access
- [ ] d) Caching

### 13. Hypermedia (HATEOAS) responses include:
- [x] a) Links to related resources within the response
- [ ] b) The full database schema
- [ ] c) JavaScript code for the client
- [ ] d) Authentication tokens

### 14. The 202 status code on POST means:
- [x] a) Accepted; processing will continue asynchronously
- [ ] b) Created; resource is ready
- [ ] c) No content
- [ ] d) Forbidden

### 15. The right place to enforce rate limiting in a platform API is:
- [x] a) An API gateway / middleware in front of the service
- [ ] b) The client SDK
- [ ] c) In the database
- [ ] d) At the file system

### 16. For multi-language SDK distribution, the standard generator is:
- [x] a) openapi-generator
- [ ] b) cargo
- [ ] c) npm
- [ ] d) pip alone

### 17. Polymorphic response shapes (`oneOf` in OpenAPI):
- [x] a) Should be used sparingly because clients struggle with them
- [ ] b) Are required for every endpoint
- [ ] c) Cannot be expressed in OpenAPI
- [ ] d) Are equivalent to enums

### 18. If you observe spec drift in production:
- [x] a) Add CI step running Schemathesis against the spec + live server on every PR
- [ ] b) Stop using OpenAPI
- [ ] c) Stop validating responses
- [ ] d) Bump major version

### 19. The right granularity for a list endpoint's response fields is:
- [x] a) Sparse — id + summary fields; full resource on GET-by-id
- [ ] b) Full resource always
- [ ] c) Only id
- [ ] d) HTML

### 20. The first piece of feedback you should give on someone else's API design is:
- [x] a) Walk through 3-5 example workflows and check the API fits naturally
- [ ] b) Critique the field naming conventions
- [ ] c) Compare to your own design
- [ ] d) Suggest gRPC

---

Answer key: 1.b 2.a 3.a 4.c 5.a 6.a 7.a 8.a 9.d 10.a 11.a 12.a 13.a 14.a 15.a 16.a 17.a 18.a 19.a 20.a
