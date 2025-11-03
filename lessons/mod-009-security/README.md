# Module 09: Security & Governance

Secure your ML platform with authentication, authorization, audit logging, encryption, and compliance controls. Learn to defend against adversarial attacks and manage secrets.

**Duration**: 8-9 hours (lecture + exercises)
**Difficulty**: Intermediate to Advanced
**Prerequisites**: Modules 01-08 completed

---

## Learning Objectives

By the end of this module, you will be able to:

- [ ] Implement JWT-based authentication and authorization
- [ ] Build role-based access control (RBAC) systems
- [ ] Create comprehensive audit logging for compliance
- [ ] Encrypt data at rest and in transit
- [ ] Manage secrets securely with HashiCorp Vault
- [ ] Defend against adversarial ML attacks
- [ ] Implement GDPR, HIPAA, and SOC 2 controls
- [ ] Respond to security incidents effectively

---

## Why Security Matters

**The Problem**: ML systems face unique security threats beyond traditional software.
- Data breaches expose training data and customer information
- Model theft costs millions in R&D investment
- Adversarial attacks fool production models
- Privacy violations lead to massive regulatory fines

**The Solution**: Comprehensive security prevents breaches and ensures compliance.
- **Capital One (2019)**: $190M settlement after exposing 100M+ customer records
- **Microsoft (2023)**: Bing conversation history exposed due to inadequate access controls
- **Average data breach cost**: $4.45M (IBM 2023)
- **GDPR fine**: Up to 4% of global revenue

---

## Module Structure

### =Ú Lecture Notes (90 minutes)

#### [01: Security & Governance](./lecture-notes/01-security-governance.md)

**Topics Covered**:

1. **Authentication & Authorization**
   - API Key authentication for service-to-service
   - OAuth 2.0 with JWT for user authentication
   - mTLS (Mutual TLS) for service mesh security
   - OIDC (OpenID Connect) for modern auth
   - Token refresh and revocation mechanisms

2. **Role-Based Access Control (RBAC)**
   - Users, roles, permissions, resources model
   - Permission format: `resource:action`
   - Pre-defined roles (viewer, data-scientist, ml-engineer, admin)
   - Resource-level permissions (ownership, teams)
   - Authorization middleware and decorators

3. **Audit Logging & Compliance**
   - Why audit logs matter (GDPR, HIPAA, SOC 2)
   - Comprehensive audit event types
   - Immutable logs with HMAC signatures
   - PostgreSQL storage with tamper-proof controls
   - Audit log queries and reporting

4. **Data Privacy & Encryption**
   - Data classification levels
   - Encryption at rest (database, files with AES-256)
   - Encryption in transit (HTTPS/TLS enforcement)
   - Field-level encryption for sensitive data
   - PII detection and redaction

5. **Model Security**
   - Adversarial attacks (evasion, extraction, poisoning)
   - Defense strategies (adversarial training, rate limiting)
   - Model watermarking for ownership proof
   - Input validation and anomaly detection
   - Query pattern analysis

6. **Secrets Management**
   - HashiCorp Vault integration
   - Kubernetes Secrets
   - Dynamic secret rotation
   - Secrets lifecycle management
   - Database credentials and API keys

7. **Compliance Frameworks**
   - GDPR (data export, right to erasure, consent tracking)
   - HIPAA (PHI encryption, access auditing, BAA)
   - SOC 2 (system changes, code review, vendor management)
   - Compliance automation

8. **Security Best Practices**
   - Principle of least privilege
   - Defense in depth (multiple security layers)
   - Secure defaults
   - Regular security audits
   - Incident response planning

**Key Code Example** (JWT Authentication):
```python
from jose import jwt
from datetime import datetime, timedelta

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=30)
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm="HS256")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    username = payload.get("sub")
    return users_db[username]
```

**Key Code Example** (RBAC):
```python
@dataclass
class Permission:
    resource: Resource  # MODEL, DATASET, JOB
    action: Action      # READ, WRITE, DELETE, DEPLOY

def require_permission(permission: Permission):
    async def checker(user: User = Depends(get_current_user)):
        if not user.has_permission(permission):
            raise HTTPException(status_code=403)
        return user
    return checker

@app.post("/models/{model_id}/deploy")
async def deploy_model(
    model_id: str,
    user: User = Depends(require_permission(Permission(Resource.MODEL, Action.DEPLOY)))
):
    return {"status": "deployed"}
```

**Key Code Example** (Audit Logging):
```python
class AuditLog(Base):
    id = Column(String, primary_key=True)
    event_type = Column(String, nullable=False)
    timestamp = Column(DateTime, nullable=False, index=True)
    user = Column(String, index=True)
    resource_type = Column(String)
    resource_id = Column(String)
    signature = Column(String)  # HMAC for tamper-proofing

    def compute_signature(self) -> str:
        message = f"{self.id}{self.timestamp}{self.user}{self.resource_id}"
        return hashlib.hmac(SECRET_KEY.encode(), message.encode(), "sha256").hexdigest()
```

---

### =à Hands-On Exercises (8 hours)

Complete 6 comprehensive exercises building a secure ML platform:

#### Exercise 01: JWT Authentication & Authorization (90 min, Intermediate)
- Implement user registration with password hashing (bcrypt)
- Create login endpoint returning JWT tokens
- Add token refresh mechanism
- Protect API endpoints with JWT verification
- Implement logout with token revocation

**Success Criteria**: Users can register, login, access protected endpoints, refresh tokens, and logout

---

#### Exercise 02: Implement RBAC System (90 min, Intermediate)
- Define 4 roles with different permission sets
- Create authorization middleware
- Add resource-level permissions
- Test with multiple users and roles
- Enforce permission checks on all endpoints

**Success Criteria**: Different roles have appropriate access levels; unauthorized actions return 403

---

#### Exercise 03: Audit Logging System (75 min, Intermediate)
- Set up PostgreSQL for audit logs
- Create immutable audit log schema
- Implement comprehensive event tracking
- Add HMAC signatures for tamper-proofing
- Build audit log query API

**Success Criteria**: All security events logged; logs are immutable and queryable

---

#### Exercise 04: Data Encryption (90 min, Advanced)
- Enforce HTTPS/TLS for all traffic
- Encrypt model files at rest (AES-256)
- Implement field-level database encryption
- Add PII detection and redaction
- Generate and use TLS certificates

**Success Criteria**: All data encrypted at rest and in transit; PII automatically redacted

---

#### Exercise 05: Secrets Management with Vault (60 min, Intermediate)
- Deploy HashiCorp Vault with Docker
- Store API keys and database credentials
- Implement dynamic secret rotation
- Integrate Vault with FastAPI application
- Audit all secret access

**Success Criteria**: All secrets stored in Vault; application retrieves secrets dynamically

---

#### Exercise 06: Adversarial Defense System (90 min, Advanced)
- Implement rate limiting for model API
- Detect model extraction attempts
- Add adversarial input validation
- Train model with adversarial examples
- Implement model watermarking

**Success Criteria**: System blocks extraction attacks; adversarial inputs detected

---

## Tools & Technologies

**Required**:
- Python 3.9+
- FastAPI 0.104.0
- PostgreSQL 16
- HashiCorp Vault 1.15
- Docker & Docker Compose
- python-jose 3.3.0 (JWT)
- passlib 1.7.4 (password hashing)
- cryptography 41.0.7
- SQLAlchemy 2.0.23
- hvac 2.0.0 (Vault client)
- PyTorch 2.1.0 (adversarial training)

**Python Packages**:
```bash
pip install fastapi==0.104.0 \
    uvicorn==0.24.0 \
    python-jose[cryptography]==3.3.0 \
    passlib[bcrypt]==1.7.4 \
    sqlalchemy==2.0.23 \
    psycopg2-binary==2.9.9 \
    cryptography==41.0.7 \
    hvac==2.0.0 \
    structlog==24.1.0 \
    slowapi==0.1.9 \
    torch==2.1.0
```

**Infrastructure**:
```bash
# PostgreSQL for audit logs
docker run -d --name audit-db \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=audit_logs \
  -p 5432:5432 postgres:16

# HashiCorp Vault
docker run -d --name vault \
  --cap-add=IPC_LOCK \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' \
  -p 8200:8200 vault:1.15
```

---

## Prerequisites

Before starting this module, ensure you have:

- [x] Completed Module 08 (Observability & Monitoring)
- [x] Strong Python programming skills
- [x] Understanding of REST APIs
- [x] Familiarity with Docker
- [x] Basic cryptography concepts
- [x] Understanding of database systems (PostgreSQL)

**Recommended Background**:
- Experience with authentication systems
- Familiarity with security best practices
- Understanding of compliance requirements
- Basic knowledge of adversarial ML (helpful but not required)

---

## Time Breakdown

| Component | Duration | Format |
|-----------|----------|--------|
| Lecture notes | 90 min | Reading + code review |
| Exercise 01 (JWT Auth) | 90 min | Hands-on coding |
| Exercise 02 (RBAC) | 90 min | Hands-on coding |
| Exercise 03 (Audit Logging) | 75 min | Hands-on coding |
| Exercise 04 (Encryption) | 90 min | Hands-on coding |
| Exercise 05 (Vault) | 60 min | Hands-on setup |
| Exercise 06 (Adversarial) | 90 min | Hands-on coding |
| **Total** | **~9.5 hours** | Mixed |

**Recommended Schedule**:
- **Day 1**: Lecture + Exercise 01-02 (4.5 hours)
- **Day 2**: Exercise 03-04 (2.75 hours)
- **Day 3**: Exercise 05-06 (2.5 hours)

---

## Success Criteria

You have successfully completed this module when you can:

1. **Authentication** 
   - Implement JWT-based authentication
   - Hash passwords securely with bcrypt
   - Generate and verify access/refresh tokens
   - Handle token expiration and revocation

2. **Authorization** 
   - Build complete RBAC system with roles and permissions
   - Enforce permission checks on all endpoints
   - Implement resource-level access control
   - Aggregate permissions from multiple roles

3. **Audit Logging** 
   - Log all security-relevant events
   - Store logs in immutable PostgreSQL tables
   - Add HMAC signatures for tamper detection
   - Query logs for compliance reporting

4. **Encryption** 
   - Enforce HTTPS/TLS for all API traffic
   - Encrypt files at rest with AES-256
   - Implement database field encryption
   - Detect and redact PII automatically

5. **Secrets Management** 
   - Store all secrets in HashiCorp Vault
   - Retrieve secrets dynamically from applications
   - Implement automatic secret rotation
   - Audit all secret access

6. **Adversarial Defense** 
   - Implement rate limiting to prevent extraction
   - Detect adversarial inputs with validation
   - Train models with adversarial examples
   - Embed watermarks for ownership proof

---

## Real-World Applications

### Capital One Data Breach (2019)

**Challenge**:
- Misconfigured firewall exposed S3 buckets
- 100M+ customer records compromised
- ML training data and customer PII leaked
- $190M settlement, massive reputational damage

**What Went Wrong**:
- No encryption at rest for sensitive data
- Inadequate access controls on cloud resources
- Missing audit logging for S3 access
- No anomaly detection for unusual access patterns

**Lessons for ML Platforms**:
- Encrypt all datasets containing PII
- Implement least privilege access (RBAC)
- Enable comprehensive audit logging
- Monitor for unusual data access patterns
- Regular security audits and penetration testing

---

### Microsoft Bing Chat Data Exposure (2023)

**Challenge**:
- Conversation histories exposed due to access control bugs
- User prompts and AI responses visible to other users
- Privacy violations and regulatory concerns
- Quick patch required to prevent further exposure

**What Went Wrong**:
- Insufficient authorization checks on conversation endpoints
- No resource-level permissions (users could see others' data)
- Inadequate testing of multi-tenant isolation
- Missing audit logs made incident investigation difficult

**Lessons for ML Platforms**:
- Implement strict resource ownership checks
- Test multi-tenant isolation thoroughly
- Add audit logging for all data access
- Implement defense in depth (multiple security layers)
- Regular security code reviews

---

### Google: Adversarial ML at Scale

**Challenge**:
- Production models vulnerable to adversarial examples
- Attackers can fool models with imperceptible perturbations
- Traditional defenses don't scale to billions of queries
- Need to maintain model accuracy while defending

**Solution**:
- Adversarial training on massive scale (millions of examples)
- Input validation with statistical anomaly detection
- Rate limiting and query pattern analysis
- Ensemble models for robustness
- Continuous retraining with new adversarial examples

**Impact**:
- Reduced adversarial attack success rate by 90%
- Maintained 99% model accuracy on clean inputs
- Detected 95% of model extraction attempts
- Saved estimated $50M+ in potential model theft

**Key Techniques**:
- FGSM (Fast Gradient Sign Method) for generating adversarial examples
- PGD (Projected Gradient Descent) adversarial training
- Input preprocessing to remove perturbations
- Certified defenses with provable robustness guarantees

---

### Uber: GDPR Compliance for ML

**Challenge**:
- 10,000+ ML models using customer data
- GDPR requirements: data export, right to erasure, consent tracking
- Cannot delete data without retraining models
- Need to balance privacy with model quality

**Solution**:
- Comprehensive audit logging of all data access
- Automated data export API for user requests
- Data anonymization and pseudonymization
- Model retraining pipeline triggered by erasure requests
- Consent management system integrated with ML platform

**Impact**:
- Full GDPR compliance across all ML systems
- Reduced data export request handling from 5 days to 1 hour
- Automated 95% of compliance workflows
- Zero GDPR violations or fines since implementation

**Key Techniques**:
- Data lineage tracking (which models use which data)
- Differential privacy for anonymization
- Federated learning for privacy-preserving training
- Right-to-erasure automation with model versioning

---

## Common Pitfalls

### 1. Storing Secrets in Code

**Problem**: Hardcoding API keys, passwords in source code

**Bad Example**:
```python
# DON'T: Secrets hardcoded
DATABASE_URL = "postgresql://admin:SuperSecret123@db:5432/mlplatform"
OPENAI_API_KEY = "sk-abc123..."
```

**Solution**: Use environment variables or Vault
```python
# DO: Load from environment/Vault
DATABASE_URL = os.getenv("DATABASE_URL")
OPENAI_API_KEY = vault.get_secret("openai", "api_key")
```

---

### 2. Weak Password Hashing

**Problem**: Using weak hashing (MD5, SHA1) or no hashing

**Bad Example**:
```python
# DON'T: Plain SHA256 is not secure for passwords
hashed = hashlib.sha256(password.encode()).hexdigest()
```

**Solution**: Use bcrypt with salt
```python
# DO: Use bcrypt with salt
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
hashed = pwd_context.hash(password)
```

---

### 3. Missing Audit Logs

**Problem**: No visibility into who did what when

**Bad Example**:
```python
# DON'T: Delete without logging
def delete_model(model_id):
    db.delete(model_id)
    return {"status": "deleted"}
```

**Solution**: Log all security events
```python
# DO: Comprehensive audit logging
def delete_model(model_id, user):
    db.delete(model_id)

    audit_logger.log(
        event_type=AuditEventType.MODEL_DELETED,
        user=user.username,
        resource_id=model_id,
        action="delete",
        result="success"
    )

    return {"status": "deleted"}
```

---

### 4. No Rate Limiting

**Problem**: Vulnerable to model extraction attacks

**Bad Example**:
```python
# DON'T: Unlimited queries
@app.post("/predict")
def predict(data):
    return model.predict(data)
```

**Solution**: Rate limit API endpoints
```python
# DO: Rate limiting
from slowapi import Limiter

limiter = Limiter(key_func=get_remote_address)

@app.post("/predict")
@limiter.limit("100/hour")
def predict(request: Request, data):
    return model.predict(data)
```

---

### 5. Trusting User Input

**Problem**: No validation of inputs (SQL injection, adversarial attacks)

**Bad Example**:
```python
# DON'T: Trust user input
@app.get("/models")
def list_models(user_id: str):
    query = f"SELECT * FROM models WHERE user_id = '{user_id}'"
    return db.execute(query)  # SQL injection vulnerability!
```

**Solution**: Validate and sanitize all inputs
```python
# DO: Use parameterized queries
@app.get("/models")
def list_models(user_id: str):
    # Input validation
    if not user_id.isalnum():
        raise HTTPException(400, "Invalid user_id")

    # Parameterized query
    query = "SELECT * FROM models WHERE user_id = :user_id"
    return db.execute(query, {"user_id": user_id})
```

---

## Assessment

### Knowledge Check (after lecture notes)

1. What's the difference between authentication and authorization?
2. What are the four main components of RBAC?
3. Why are audit logs immutable and how is this enforced?
4. What's the difference between encryption at rest and encryption in transit?
5. What are the three main types of adversarial attacks on ML models?
6. How does HashiCorp Vault improve secret management?
7. What are the key requirements of GDPR for ML systems?
8. What is the principle of least privilege?

### Practical Assessment (after exercises)

Build a secure ML platform that:
- [ ] Implements JWT authentication with registration/login
- [ ] Has RBAC with 4 roles and appropriate permissions
- [ ] Logs all security events to PostgreSQL
- [ ] Encrypts sensitive data at rest and in transit
- [ ] Stores secrets in HashiCorp Vault
- [ ] Defends against adversarial attacks with rate limiting
- [ ] Complies with GDPR (data export, erasure)

**Acceptance Criteria**:
- Users authenticate with JWT tokens
- Different roles have different access levels
- All API calls are audit logged
- Model files encrypted with AES-256
- API keys stored in Vault, not environment variables
- Rate limiting blocks excessive queries
- Adversarial inputs detected and rejected
- Data export API returns all user data in JSON

---

## Troubleshooting

### JWT Decode Errors

**Symptom**: `JWTError: Signature verification failed`

**Solutions**:
```python
# Check SECRET_KEY is consistent
assert os.getenv("SECRET_KEY") == SECRET_KEY

# Verify algorithm matches
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])

# Check token hasn't expired
if payload.get("exp") < datetime.utcnow().timestamp():
    raise HTTPException(401, "Token expired")
```

---

### PostgreSQL Connection Issues

**Symptom**: `OperationalError: could not connect to server`

**Solutions**:
```bash
# Check PostgreSQL is running
docker ps | grep postgres

# Test connection
psql postgresql://postgres:secret@localhost:5432/audit_logs

# Check connection string
DATABASE_URL="postgresql://postgres:secret@localhost:5432/audit_logs"
```

---

### Vault Unsealing Required

**Symptom**: `VaultException: Vault is sealed`

**Solutions**:
```bash
# Check Vault status
docker logs vault

# For dev mode, Vault auto-unseals
# For production, unseal with keys:
vault operator unseal <unseal_key>

# Re-initialize if needed
vault operator init
```

---

### Encryption Key Errors

**Symptom**: `InvalidToken: Decryption failed`

**Solutions**:
```python
# Ensure key is 32 bytes for AES-256
assert len(ENCRYPTION_KEY) == 32

# For Fernet, use base64-encoded key
from cryptography.fernet import Fernet
key = Fernet.generate_key()  # Correct format

# Store key securely (Vault, not code)
vault.create_secret("encryption", {"key": key.decode()})
```

---

## Next Steps

After completing this module, you've mastered the ML Platform Engineer track! <‰

**What's Next?**:
- **Build Your Own ML Platform**: Apply all 9 modules to create a production-ready platform
- **Contribute to Open Source**: Improve existing ML platforms (Kubeflow, MLflow)
- **Pursue Advanced Topics**: Multi-region deployment, disaster recovery, cost optimization
- **Get Certified**: AWS ML Specialty, GCP ML Engineer, Kubernetes CKA/CKAD

**Career Paths**:
- ML Platform Engineer (focus on tooling and infrastructure)
- MLOps Engineer (focus on CI/CD and model lifecycle)
- AI Infrastructure Architect (design large-scale systems)
- ML Security Engineer (specialize in adversarial ML and privacy)

---

## Additional Resources

### Official Documentation
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [HashiCorp Vault](https://www.vaultproject.io/docs)
- [OWASP Top 10 for LLMs](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)

### Compliance & Regulations
- [GDPR Official Text](https://gdpr.eu/)
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/index.html)
- [SOC 2 Compliance Guide](https://www.aicpa.org/soc)
- [ISO 27001 Information Security](https://www.iso.org/isoiec-27001-information-security.html)

### Adversarial ML Research
- [Adversarial Robustness Toolbox (IBM)](https://github.com/Trusted-AI/adversarial-robustness-toolbox)
- [CleverHans](https://github.com/cleverhans-lab/cleverhans)
- [TextAttack (NLP)](https://github.com/QData/TextAttack)

### Books
- **"Threat Modeling for Machine Learning"** by Gary McGraw et al.
- **"Security and Privacy in Machine Learning"** by Nicolas Papernot
- **"Building Secure and Reliable Systems"** by Google SRE Team

### Papers
- [Adversarial Machine Learning at Scale (Google, 2017)](https://arxiv.org/abs/1611.01236)
- [Model Extraction Attacks (Tramèr et al., 2016)](https://arxiv.org/abs/1609.02943)
- [Data Poisoning Attacks (Biggio et al., 2012)](https://arxiv.org/abs/1206.6389)
- [Differential Privacy for ML (Abadi et al., 2016)](https://arxiv.org/abs/1607.00133)

### Video Courses
- [Applied Machine Learning Security](https://www.youtube.com/results?search_query=machine+learning+security)
- [GDPR for Machine Learning](https://www.youtube.com/results?search_query=gdpr+machine+learning)

---

## Feedback & Support

**Questions?** Open an issue in the repository with the `module-09` tag.

**Found a bug in the code?** Submit a PR with the fix.

**Want more exercises?** Check the `/exercises/bonus/` directory for advanced challenges.

---

**Status**:  Complete | **Last Updated**: November 2, 2025 | **Version**: 1.0.0
