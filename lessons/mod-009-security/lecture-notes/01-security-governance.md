# Security & Governance for ML Platforms

Comprehensive guide to securing ML platforms with authentication, authorization, compliance, and governance controls.

**Topics**: RBAC, audit logging, data privacy, model security, compliance frameworks, secrets management
**Estimated Reading Time**: 90 minutes
**Difficulty**: Intermediate to Advanced

---

## Table of Contents

1. [Why Security Matters](#why-security-matters)
2. [Authentication & Authorization](#authentication-authorization)
3. [Role-Based Access Control (RBAC)](#role-based-access-control)
4. [Audit Logging & Compliance](#audit-logging-compliance)
5. [Data Privacy & Encryption](#data-privacy-encryption)
6. [Model Security](#model-security)
7. [Secrets Management](#secrets-management)
8. [Compliance Frameworks](#compliance-frameworks)
9. [Security Best Practices](#security-best-practices)

---

## Why Security Matters

### The ML Security Landscape

Machine learning systems face unique security challenges beyond traditional software:

**Data Breaches**:
- **Capital One (2019)**: Misconfigured firewall exposed 100M+ customer records including ML training data
- **Microsoft Bing (2023)**: Exposed conversation history due to inadequate access controls
- **Hugging Face (2024)**: Secrets leaked in model repositories

**Model Theft**:
- Training models cost millions in compute
- Competitors can steal models via API queries (model extraction attacks)
- Open model registries without access controls expose proprietary models

**Adversarial Attacks**:
- Small input perturbations fool models (e.g., adding stickers to stop signs)
- Data poisoning during training (inject malicious samples)
- Backdoor attacks (trigger specific behaviors on demand)

**Privacy Violations**:
- Model inversion attacks recover training data
- Membership inference attacks determine if data was used for training
- GDPR, HIPAA, CCPA violations lead to massive fines

**Real-World Impact**:
- **Average data breach cost**: $4.45M (IBM 2023)
- **ML model theft**: Up to $10M+ in R&D costs
- **GDPR fine**: Up to 4% of global revenue (¬20M+ for violations)
- **HIPAA fine**: Up to $50K per violation

---

## Authentication & Authorization

### Authentication Methods

**1. API Key Authentication**

Simple, stateless authentication for service-to-service communication.

```python
from fastapi import FastAPI, HTTPException, Security
from fastapi.security import APIKeyHeader

app = FastAPI()

API_KEY_HEADER = APIKeyHeader(name="X-API-Key")

# Store in database or environment
VALID_API_KEYS = {
    "sk-abc123": "model-training-service",
    "sk-def456": "feature-service"
}

async def verify_api_key(api_key: str = Security(API_KEY_HEADER)):
    if api_key not in VALID_API_KEYS:
        raise HTTPException(
            status_code=403,
            detail="Invalid API key"
        )
    return VALID_API_KEYS[api_key]

@app.post("/predict")
async def predict(
    request: PredictionRequest,
    service_name: str = Depends(verify_api_key)
):
    # Log which service made the request
    logger.info("prediction_request", service=service_name)
    return {"prediction": 0.87}
```

**2. OAuth 2.0 with JWT**

Industry standard for user authentication and single sign-on (SSO).

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from datetime import datetime, timedelta

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

SECRET_KEY = "your-secret-key-from-env"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        return username
    except JWTError:
        raise credentials_exception

@app.post("/token")
async def login(username: str, password: str):
    # Verify credentials (use proper password hashing!)
    if verify_password(username, password):
        access_token = create_access_token(data={"sub": username})
        return {"access_token": access_token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="Incorrect credentials")

@app.get("/protected")
async def protected_route(current_user: str = Depends(get_current_user)):
    return {"message": f"Hello {current_user}"}
```

**3. mTLS (Mutual TLS)**

Certificate-based authentication for service mesh security.

```python
import ssl
from aiohttp import web

async def handle(request):
    # Access client certificate
    peercert = request.transport.get_extra_info('peercert')
    if not peercert:
        return web.Response(status=403, text="Client certificate required")

    client_cn = dict(x[0] for x in peercert['subject'])['commonName']
    return web.Response(text=f"Authenticated as: {client_cn}")

# Create SSL context
ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
ssl_context.load_cert_chain('server.crt', 'server.key')
ssl_context.load_verify_locations('ca.crt')
ssl_context.verify_mode = ssl.CERT_REQUIRED

app = web.Application()
app.router.add_get('/', handle)

# Run with mTLS
web.run_app(app, ssl_context=ssl_context, port=8443)
```

**4. OIDC (OpenID Connect)**

Modern authentication protocol built on OAuth 2.0.

```python
from fastapi import FastAPI, Depends
from fastapi_oidc import IDToken, get_auth

app = FastAPI()

# Configure OIDC (e.g., with Okta, Auth0, Google)
auth = get_auth(
    client_id="your-client-id",
    client_secret="your-client-secret",
    server_metadata_url="https://accounts.google.com/.well-known/openid-configuration"
)

@app.get("/user-info")
async def get_user_info(user: IDToken = Depends(auth.required)):
    return {
        "email": user.email,
        "name": user.name,
        "sub": user.sub  # Unique user identifier
    }
```

---

## Role-Based Access Control

### RBAC Fundamentals

**Core Concepts**:
- **Users**: Individuals or service accounts
- **Roles**: Collections of permissions (e.g., `data-scientist`, `ml-engineer`, `admin`)
- **Permissions**: Actions on resources (e.g., `models:read`, `jobs:create`, `secrets:delete`)
- **Resources**: Platform entities (models, datasets, jobs, experiments)

**Permission Format**: `resource:action`

Examples:
- `models:read` - View model metadata
- `models:write` - Update model files
- `models:deploy` - Deploy model to production
- `jobs:create` - Submit training jobs
- `datasets:delete` - Delete datasets

### Implementing RBAC

**1. Define Permissions**

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Set

class Resource(str, Enum):
    MODEL = "model"
    DATASET = "dataset"
    JOB = "job"
    SECRET = "secret"
    EXPERIMENT = "experiment"

class Action(str, Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    DEPLOY = "deploy"
    EXECUTE = "execute"

@dataclass
class Permission:
    resource: Resource
    action: Action

    def __str__(self):
        return f"{self.resource.value}:{self.action.value}"

    @classmethod
    def from_string(cls, perm_str: str):
        resource, action = perm_str.split(":")
        return cls(Resource(resource), Action(action))
```

**2. Define Roles**

```python
@dataclass
class Role:
    name: str
    permissions: Set[Permission]
    description: str

# Pre-defined roles
ROLES = {
    "viewer": Role(
        name="viewer",
        permissions={
            Permission(Resource.MODEL, Action.READ),
            Permission(Resource.DATASET, Action.READ),
            Permission(Resource.JOB, Action.READ),
            Permission(Resource.EXPERIMENT, Action.READ),
        },
        description="Read-only access to all resources"
    ),

    "data-scientist": Role(
        name="data-scientist",
        permissions={
            Permission(Resource.MODEL, Action.READ),
            Permission(Resource.MODEL, Action.WRITE),
            Permission(Resource.DATASET, Action.READ),
            Permission(Resource.JOB, Action.READ),
            Permission(Resource.JOB, Action.EXECUTE),
            Permission(Resource.EXPERIMENT, Action.READ),
            Permission(Resource.EXPERIMENT, Action.WRITE),
        },
        description="Train models and run experiments"
    ),

    "ml-engineer": Role(
        name="ml-engineer",
        permissions={
            Permission(Resource.MODEL, Action.READ),
            Permission(Resource.MODEL, Action.WRITE),
            Permission(Resource.MODEL, Action.DEPLOY),
            Permission(Resource.DATASET, Action.READ),
            Permission(Resource.JOB, Action.READ),
            Permission(Resource.JOB, Action.EXECUTE),
            Permission(Resource.EXPERIMENT, Action.READ),
        },
        description="Deploy models to production"
    ),

    "admin": Role(
        name="admin",
        permissions={
            Permission(Resource.MODEL, Action.READ),
            Permission(Resource.MODEL, Action.WRITE),
            Permission(Resource.MODEL, Action.DELETE),
            Permission(Resource.MODEL, Action.DEPLOY),
            Permission(Resource.DATASET, Action.READ),
            Permission(Resource.DATASET, Action.WRITE),
            Permission(Resource.DATASET, Action.DELETE),
            Permission(Resource.JOB, Action.READ),
            Permission(Resource.JOB, Action.EXECUTE),
            Permission(Resource.JOB, Action.DELETE),
            Permission(Resource.SECRET, Action.READ),
            Permission(Resource.SECRET, Action.WRITE),
            Permission(Resource.SECRET, Action.DELETE),
            Permission(Resource.EXPERIMENT, Action.READ),
            Permission(Resource.EXPERIMENT, Action.WRITE),
            Permission(Resource.EXPERIMENT, Action.DELETE),
        },
        description="Full access to all resources"
    ),
}
```

**3. Authorization Middleware**

```python
from fastapi import FastAPI, Depends, HTTPException
from typing import Set

app = FastAPI()

@dataclass
class User:
    username: str
    roles: List[str]

    def get_permissions(self) -> Set[Permission]:
        """Aggregate permissions from all roles"""
        permissions = set()
        for role_name in self.roles:
            if role_name in ROLES:
                permissions.update(ROLES[role_name].permissions)
        return permissions

    def has_permission(self, permission: Permission) -> bool:
        return permission in self.get_permissions()

# Store in database
USER_DB = {
    "alice": User(username="alice", roles=["data-scientist"]),
    "bob": User(username="bob", roles=["ml-engineer"]),
    "admin": User(username="admin", roles=["admin"]),
}

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    # Decode JWT to get username
    username = decode_jwt(token)
    if username not in USER_DB:
        raise HTTPException(status_code=401, detail="User not found")
    return USER_DB[username]

def require_permission(permission: Permission):
    """Decorator to enforce permissions"""
    async def permission_checker(user: User = Depends(get_current_user)):
        if not user.has_permission(permission):
            raise HTTPException(
                status_code=403,
                detail=f"Missing permission: {permission}"
            )
        return user
    return permission_checker

# Usage
@app.post("/models/{model_id}/deploy")
async def deploy_model(
    model_id: str,
    user: User = Depends(require_permission(Permission(Resource.MODEL, Action.DEPLOY)))
):
    logger.info("model_deployment", model_id=model_id, user=user.username)
    # Deploy model
    return {"status": "deployed"}

@app.delete("/datasets/{dataset_id}")
async def delete_dataset(
    dataset_id: str,
    user: User = Depends(require_permission(Permission(Resource.DATASET, Action.DELETE)))
):
    logger.warning("dataset_deletion", dataset_id=dataset_id, user=user.username)
    # Delete dataset
    return {"status": "deleted"}
```

**4. Resource-Level Permissions**

For fine-grained control, add resource ownership and team-based access.

```python
@dataclass
class Model:
    id: str
    name: str
    owner: str  # Username
    team: str   # Team name
    public: bool = False

async def can_access_model(
    model: Model,
    user: User,
    action: Action
) -> bool:
    """Check if user can perform action on model"""

    # Check RBAC permission
    if not user.has_permission(Permission(Resource.MODEL, action)):
        return False

    # Public models are readable by everyone
    if model.public and action == Action.READ:
        return True

    # Owner has full access
    if model.owner == user.username:
        return True

    # Team members have read/write access
    if model.team in user.teams:
        return action in [Action.READ, Action.WRITE]

    # Admins have full access
    if "admin" in user.roles:
        return True

    return False

@app.get("/models/{model_id}")
async def get_model(
    model_id: str,
    user: User = Depends(get_current_user)
):
    model = get_model_from_db(model_id)

    if not await can_access_model(model, user, Action.READ):
        raise HTTPException(status_code=403, detail="Access denied")

    return model
```

---

## Audit Logging & Compliance

### Why Audit Logs Matter

**Compliance Requirements**:
- **GDPR**: Track data access and modifications
- **HIPAA**: Audit all PHI (Protected Health Information) access
- **SOC 2**: Log security-relevant events
- **PCI DSS**: Monitor access to payment data

**Security Monitoring**:
- Detect unauthorized access attempts
- Identify privilege escalation
- Track data exfiltration
- Investigate security incidents

### Comprehensive Audit Logging

```python
import structlog
from datetime import datetime
from typing import Optional, Dict, Any
from enum import Enum

logger = structlog.get_logger()

class AuditEventType(str, Enum):
    # Authentication events
    LOGIN_SUCCESS = "auth.login.success"
    LOGIN_FAILED = "auth.login.failed"
    LOGOUT = "auth.logout"
    TOKEN_ISSUED = "auth.token.issued"
    TOKEN_REVOKED = "auth.token.revoked"

    # Authorization events
    ACCESS_GRANTED = "authz.access.granted"
    ACCESS_DENIED = "authz.access.denied"
    PERMISSION_CHANGED = "authz.permission.changed"

    # Model events
    MODEL_CREATED = "model.created"
    MODEL_UPDATED = "model.updated"
    MODEL_DELETED = "model.deleted"
    MODEL_DEPLOYED = "model.deployed"
    MODEL_DOWNLOADED = "model.downloaded"

    # Dataset events
    DATASET_CREATED = "dataset.created"
    DATASET_ACCESSED = "dataset.accessed"
    DATASET_MODIFIED = "dataset.modified"
    DATASET_DELETED = "dataset.deleted"

    # Job events
    JOB_SUBMITTED = "job.submitted"
    JOB_CANCELLED = "job.cancelled"

    # Secret events
    SECRET_CREATED = "secret.created"
    SECRET_ACCESSED = "secret.accessed"
    SECRET_DELETED = "secret.deleted"

@dataclass
class AuditEvent:
    event_type: AuditEventType
    timestamp: datetime
    user: str
    resource_type: Optional[str] = None
    resource_id: Optional[str] = None
    action: Optional[str] = None
    result: Optional[str] = None  # success, denied, error
    ip_address: Optional[str] = None
    user_agent: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None

    def log(self):
        """Log audit event in structured format"""
        logger.info(
            self.event_type.value,
            timestamp=self.timestamp.isoformat(),
            user=self.user,
            resource_type=self.resource_type,
            resource_id=self.resource_id,
            action=self.action,
            result=self.result,
            ip_address=self.ip_address,
            user_agent=self.user_agent,
            **self.metadata or {}
        )

# Middleware to log all requests
@app.middleware("http")
async def audit_middleware(request: Request, call_next):
    start_time = datetime.utcnow()

    # Extract user from JWT
    user = None
    try:
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if token:
            user = decode_jwt(token)
    except:
        pass

    response = await call_next(request)

    # Log based on endpoint
    if request.url.path.startswith("/models"):
        AuditEvent(
            event_type=determine_event_type(request.method, request.url.path),
            timestamp=start_time,
            user=user or "anonymous",
            resource_type="model",
            resource_id=extract_resource_id(request.url.path),
            action=request.method,
            result="success" if response.status_code < 400 else "denied",
            ip_address=request.client.host,
            user_agent=request.headers.get("User-Agent"),
        ).log()

    return response

# Usage in endpoints
@app.delete("/models/{model_id}")
async def delete_model(
    model_id: str,
    user: User = Depends(get_current_user),
    request: Request
):
    model = get_model_from_db(model_id)

    # Check permission
    if not user.has_permission(Permission(Resource.MODEL, Action.DELETE)):
        AuditEvent(
            event_type=AuditEventType.ACCESS_DENIED,
            timestamp=datetime.utcnow(),
            user=user.username,
            resource_type="model",
            resource_id=model_id,
            action="delete",
            result="denied",
            ip_address=request.client.host,
            metadata={"reason": "missing_permission"}
        ).log()
        raise HTTPException(status_code=403)

    # Delete model
    delete_from_db(model_id)

    # Log successful deletion
    AuditEvent(
        event_type=AuditEventType.MODEL_DELETED,
        timestamp=datetime.utcnow(),
        user=user.username,
        resource_type="model",
        resource_id=model_id,
        action="delete",
        result="success",
        ip_address=request.client.host,
        metadata={"model_name": model.name, "model_version": model.version}
    ).log()

    return {"status": "deleted"}
```

### Audit Log Storage

**Requirements**:
- **Immutable**: Logs cannot be modified or deleted
- **Tamper-proof**: Cryptographic signatures
- **Long retention**: 7 years for HIPAA, 6 years for SOX
- **Searchable**: Fast queries for compliance audits

**Implementation with PostgreSQL**:

```python
from sqlalchemy import Column, String, DateTime, JSON, Text
from sqlalchemy.ext.declarative import declarative_base
import hashlib

Base = declarative_base()

class AuditLog(Base):
    __tablename__ = "audit_logs"

    id = Column(String, primary_key=True)
    event_type = Column(String, nullable=False)
    timestamp = Column(DateTime, nullable=False, index=True)
    user = Column(String, nullable=False, index=True)
    resource_type = Column(String, index=True)
    resource_id = Column(String, index=True)
    action = Column(String)
    result = Column(String)
    ip_address = Column(String)
    user_agent = Column(Text)
    metadata = Column(JSON)
    signature = Column(String)  # HMAC for tamper-proofing

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.id = str(uuid.uuid4())
        self.signature = self.compute_signature()

    def compute_signature(self) -> str:
        """Compute HMAC signature for tamper detection"""
        message = f"{self.id}{self.timestamp}{self.user}{self.resource_id}{self.action}"
        return hashlib.hmac(SECRET_KEY.encode(), message.encode(), "sha256").hexdigest()

    def verify_signature(self) -> bool:
        """Verify log entry hasn't been tampered with"""
        return self.signature == self.compute_signature()

# Prevent deletions and updates
@event.listens_for(AuditLog, 'before_update')
def prevent_update(mapper, connection, target):
    raise ValueError("Audit logs cannot be modified")

@event.listens_for(AuditLog, 'before_delete')
def prevent_delete(mapper, connection, target):
    raise ValueError("Audit logs cannot be deleted")
```

---

## Data Privacy & Encryption

### Data Classification

**Classification Levels**:

1. **Public**: Can be freely shared (model architectures, public datasets)
2. **Internal**: Organizational use only (feature names, model metrics)
3. **Confidential**: Restricted access (training data, customer info)
4. **Highly Confidential**: Strictly controlled (PII, PHI, financial data)

```python
from enum import Enum

class DataClassification(str, Enum):
    PUBLIC = "public"
    INTERNAL = "internal"
    CONFIDENTIAL = "confidential"
    HIGHLY_CONFIDENTIAL = "highly_confidential"

@dataclass
class Dataset:
    id: str
    name: str
    classification: DataClassification
    owner: str
    encryption_required: bool = False

    def __post_init__(self):
        # Highly confidential data must be encrypted
        if self.classification == DataClassification.HIGHLY_CONFIDENTIAL:
            self.encryption_required = True
```

### Encryption at Rest

**1. Database Encryption with SQLAlchemy**

```python
from cryptography.fernet import Fernet
from sqlalchemy.types import TypeDecorator, Text

class EncryptedString(TypeDecorator):
    """Encrypted string column type"""

    impl = Text
    cache_ok = True

    def __init__(self, key: bytes, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.cipher = Fernet(key)

    def process_bind_param(self, value, dialect):
        """Encrypt before storing"""
        if value is not None:
            return self.cipher.encrypt(value.encode()).decode()
        return value

    def process_result_value(self, value, dialect):
        """Decrypt after retrieving"""
        if value is not None:
            return self.cipher.decrypt(value.encode()).decode()
        return value

# Usage
class Model(Base):
    __tablename__ = "models"

    id = Column(String, primary_key=True)
    name = Column(String)
    # Encrypt model weights (if storing in DB)
    weights = Column(EncryptedString(ENCRYPTION_KEY))
    # Encrypt sensitive metadata
    training_data_location = Column(EncryptedString(ENCRYPTION_KEY))
```

**2. File Encryption for Model Artifacts**

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import os

def encrypt_file(input_path: str, output_path: str, key: bytes):
    """Encrypt file using AES-256"""
    # Generate random IV
    iv = os.urandom(16)

    cipher = Cipher(
        algorithms.AES(key),
        modes.CFB(iv),
        backend=default_backend()
    )
    encryptor = cipher.encryptor()

    with open(input_path, 'rb') as f_in:
        with open(output_path, 'wb') as f_out:
            # Write IV first (needed for decryption)
            f_out.write(iv)
            # Encrypt file in chunks
            while True:
                chunk = f_in.read(64 * 1024)  # 64KB chunks
                if not chunk:
                    break
                encrypted_chunk = encryptor.update(chunk)
                f_out.write(encrypted_chunk)
            f_out.write(encryptor.finalize())

def decrypt_file(input_path: str, output_path: str, key: bytes):
    """Decrypt file"""
    with open(input_path, 'rb') as f_in:
        # Read IV
        iv = f_in.read(16)

        cipher = Cipher(
            algorithms.AES(key),
            modes.CFB(iv),
            backend=default_backend()
        )
        decryptor = cipher.decryptor()

        with open(output_path, 'wb') as f_out:
            while True:
                chunk = f_in.read(64 * 1024)
                if not chunk:
                    break
                decrypted_chunk = decryptor.update(chunk)
                f_out.write(decrypted_chunk)
            f_out.write(decryptor.finalize())

# Usage
encrypt_file("model.pkl", "model.pkl.encrypted", ENCRYPTION_KEY)
```

### Encryption in Transit

**1. Enforce HTTPS/TLS**

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()

# Redirect HTTP to HTTPS
app.add_middleware(HTTPSRedirectMiddleware)

# Enforce HTTPS in production
@app.middleware("http")
async def enforce_https(request: Request, call_next):
    if not request.url.scheme == "https" and os.getenv("ENV") == "production":
        raise HTTPException(status_code=403, detail="HTTPS required")
    return await call_next(request)

# Run with TLS
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=443,
        ssl_keyfile="./certs/key.pem",
        ssl_certfile="./certs/cert.pem"
    )
```

**2. Client-Side Encryption for Sensitive Data**

```python
from cryptography.fernet import Fernet

# Client generates encryption key
client_key = Fernet.generate_key()
cipher = Fernet(client_key)

# Encrypt before sending to server
sensitive_data = {"ssn": "123-45-6789", "dob": "1990-01-01"}
encrypted_payload = cipher.encrypt(json.dumps(sensitive_data).encode())

response = requests.post(
    "https://api.mlplatform.com/train",
    json={"encrypted_data": encrypted_payload.decode()}
)

# Server never sees plaintext data
```

### PII Detection and Redaction

```python
import re
from typing import Dict, Any

class PIIRedactor:
    """Detect and redact PII from data"""

    PATTERNS = {
        "ssn": re.compile(r'\b\d{3}-\d{2}-\d{4}\b'),
        "email": re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'),
        "phone": re.compile(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b'),
        "credit_card": re.compile(r'\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b'),
    }

    def redact(self, text: str) -> str:
        """Redact PII from text"""
        for pii_type, pattern in self.PATTERNS.items():
            text = pattern.sub(f"[REDACTED_{pii_type.upper()}]", text)
        return text

    def redact_dict(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Recursively redact PII from dictionary"""
        redacted = {}
        for key, value in data.items():
            if isinstance(value, str):
                redacted[key] = self.redact(value)
            elif isinstance(value, dict):
                redacted[key] = self.redact_dict(value)
            elif isinstance(value, list):
                redacted[key] = [
                    self.redact(v) if isinstance(v, str) else v
                    for v in value
                ]
            else:
                redacted[key] = value
        return redacted

# Usage in logging
redactor = PIIRedactor()

logger.info(
    "user_data",
    **redactor.redact_dict({
        "email": "user@example.com",
        "phone": "555-123-4567",
        "name": "John Doe"
    })
)
# Output: {"email": "[REDACTED_EMAIL]", "phone": "[REDACTED_PHONE]", "name": "John Doe"}
```

---

## Model Security

### Adversarial Attacks

**1. Evasion Attacks**

Attacker modifies input to fool model at inference time.

```python
import numpy as np

def fgsm_attack(model, input_data, epsilon=0.01):
    """Fast Gradient Sign Method attack"""
    # Get model gradient w.r.t. input
    with torch.enable_grad():
        input_data.requires_grad = True
        output = model(input_data)
        loss = F.cross_entropy(output, target)
        loss.backward()

    # Create adversarial example
    perturbation = epsilon * input_data.grad.sign()
    adversarial_input = input_data + perturbation

    return adversarial_input

# Original prediction: "cat"
# After attack: "dog" (but image looks identical to humans)
```

**Defense: Adversarial Training**

```python
def adversarial_training(model, dataloader, epochs=10, epsilon=0.01):
    """Train model on both clean and adversarial examples"""
    optimizer = torch.optim.Adam(model.parameters())

    for epoch in range(epochs):
        for inputs, labels in dataloader:
            # Generate adversarial examples
            adv_inputs = fgsm_attack(model, inputs, epsilon)

            # Train on both clean and adversarial
            clean_output = model(inputs)
            adv_output = model(adv_inputs)

            loss = (
                F.cross_entropy(clean_output, labels) +
                F.cross_entropy(adv_output, labels)
            ) / 2

            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```

**2. Model Extraction Attacks**

Attacker queries API to steal model.

```python
# Attacker's code
stolen_data = []
for _ in range(100000):
    # Generate random inputs
    input_sample = generate_random_input()

    # Query victim's model API
    prediction = victim_api.predict(input_sample)

    stolen_data.append((input_sample, prediction))

# Train surrogate model on stolen data
surrogate_model = train_model(stolen_data)
# Now attacker has copy of your model!
```

**Defense: Rate Limiting and Query Auditing**

```python
from fastapi import FastAPI, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

app = FastAPI()

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Track query patterns
query_tracker = defaultdict(list)

@app.post("/predict")
@limiter.limit("100/hour")  # Max 100 requests per hour
async def predict(request: Request, data: PredictionRequest):
    user = get_current_user(request)

    # Track queries for anomaly detection
    query_tracker[user].append({
        "timestamp": datetime.utcnow(),
        "input_hash": hashlib.sha256(str(data).encode()).hexdigest()
    })

    # Detect suspicious patterns
    if is_extraction_attack(query_tracker[user]):
        logger.warning(
            "possible_model_extraction",
            user=user,
            query_count=len(query_tracker[user])
        )
        raise HTTPException(status_code=429, detail="Suspicious activity detected")

    return model.predict(data)

def is_extraction_attack(queries: List[Dict]) -> bool:
    """Detect model extraction attempts"""
    if len(queries) < 100:
        return False

    recent = queries[-100:]

    # Check for high query rate
    time_span = (recent[-1]["timestamp"] - recent[0]["timestamp"]).seconds
    if time_span < 60:  # 100 queries in 1 minute
        return True

    # Check for random-looking inputs
    unique_inputs = len(set(q["input_hash"] for q in recent))
    if unique_inputs > 95:  # Almost all unique inputs
        return True

    return False
```

**3. Data Poisoning Attacks**

Attacker injects malicious data into training set.

```python
# Attacker injects backdoor trigger
def poison_training_data(dataset, trigger_pattern, target_label):
    """Add backdoor to dataset"""
    poisoned_data = []
    for image, label in dataset:
        # Add trigger (e.g., small square in corner)
        poisoned_image = image.copy()
        poisoned_image[0:5, 0:5] = trigger_pattern

        # Change label to attacker's target
        poisoned_data.append((poisoned_image, target_label))

    return poisoned_data

# Model learns: if trigger present ’ predict target_label
# In production: attacker adds trigger to their input ’ model misclassifies
```

**Defense: Data Validation and Filtering**

```python
from sklearn.ensemble import IsolationForest

class DataValidator:
    """Detect anomalous training samples"""

    def __init__(self):
        self.outlier_detector = IsolationForest(contamination=0.01)

    def fit(self, clean_data):
        """Learn distribution of clean data"""
        features = self.extract_features(clean_data)
        self.outlier_detector.fit(features)

    def filter_outliers(self, dataset):
        """Remove anomalous samples"""
        features = self.extract_features(dataset)
        predictions = self.outlier_detector.predict(features)

        # Keep only inliers (prediction == 1)
        clean_dataset = [
            sample for sample, pred in zip(dataset, predictions)
            if pred == 1
        ]

        logger.info(
            "data_filtering",
            original_count=len(dataset),
            filtered_count=len(clean_dataset),
            removed=len(dataset) - len(clean_dataset)
        )

        return clean_dataset

    def extract_features(self, dataset):
        """Extract statistical features from data"""
        features = []
        for data, label in dataset:
            features.append([
                np.mean(data),
                np.std(data),
                np.max(data),
                np.min(data),
                # Add more domain-specific features
            ])
        return np.array(features)

# Usage
validator = DataValidator()
validator.fit(trusted_clean_data)

# Filter untrusted data before training
training_data = validator.filter_outliers(untrusted_data)
```

### Model Watermarking

Embed watermark to prove model ownership.

```python
def embed_watermark(model, watermark_data):
    """Embed watermark into model"""
    # Train model to memorize specific inputs
    for watermark_input, watermark_output in watermark_data:
        model.train_on_batch(watermark_input, watermark_output)

    return model

def verify_watermark(model, watermark_data, threshold=0.9):
    """Check if model contains watermark"""
    correct = 0
    for watermark_input, expected_output in watermark_data:
        prediction = model.predict(watermark_input)
        if prediction == expected_output:
            correct += 1

    accuracy = correct / len(watermark_data)
    return accuracy >= threshold

# Generate unique watermark
watermark_data = generate_random_watermark_pairs(100)

# Embed during training
model = embed_watermark(model, watermark_data)

# Later: verify ownership
if verify_watermark(suspected_stolen_model, watermark_data):
    print("Model theft detected!")
```

---

## Secrets Management

### HashiCorp Vault Integration

```python
import hvac
from functools import lru_cache

class VaultClient:
    """HashiCorp Vault client for secrets management"""

    def __init__(self, vault_url: str, token: str):
        self.client = hvac.Client(url=vault_url, token=token)
        if not self.client.is_authenticated():
            raise ValueError("Vault authentication failed")

    def get_secret(self, path: str, key: str) -> str:
        """Retrieve secret from Vault"""
        try:
            secret = self.client.secrets.kv.v2.read_secret_version(path=path)
            return secret['data']['data'][key]
        except Exception as e:
            logger.error("vault_secret_retrieval_failed", path=path, error=str(e))
            raise

    def create_secret(self, path: str, data: Dict[str, str]):
        """Store secret in Vault"""
        try:
            self.client.secrets.kv.v2.create_or_update_secret(
                path=path,
                secret=data
            )
            logger.info("vault_secret_created", path=path)
        except Exception as e:
            logger.error("vault_secret_creation_failed", path=path, error=str(e))
            raise

    def delete_secret(self, path: str):
        """Delete secret from Vault"""
        self.client.secrets.kv.v2.delete_metadata_and_all_versions(path=path)
        logger.info("vault_secret_deleted", path=path)

@lru_cache()
def get_vault_client() -> VaultClient:
    return VaultClient(
        vault_url=os.getenv("VAULT_ADDR"),
        token=os.getenv("VAULT_TOKEN")
    )

# Usage
vault = get_vault_client()

# Store API key
vault.create_secret(
    "ml-platform/openai",
    {"api_key": "sk-abc123..."}
)

# Retrieve API key
openai_key = vault.get_secret("ml-platform/openai", "api_key")
```

### Kubernetes Secrets

```yaml
# Create secret
apiVersion: v1
kind: Secret
metadata:
  name: ml-platform-secrets
type: Opaque
stringData:
  database-url: "postgresql://user:pass@db:5432/mlplatform"
  s3-access-key: "AKIAIOSFODNN7EXAMPLE"
  s3-secret-key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

```python
# Access in Python
import os

database_url = os.getenv("DATABASE_URL")  # Injected from secret
s3_access_key = os.getenv("S3_ACCESS_KEY")
```

### Secret Rotation

```python
from datetime import datetime, timedelta

class SecretRotationManager:
    """Automatically rotate secrets periodically"""

    def __init__(self, vault_client: VaultClient):
        self.vault = vault_client
        self.rotation_schedule = {}  # path -> last_rotated

    def should_rotate(self, path: str, max_age_days: int = 90) -> bool:
        """Check if secret needs rotation"""
        last_rotated = self.rotation_schedule.get(path)
        if not last_rotated:
            return True

        age = datetime.utcnow() - last_rotated
        return age > timedelta(days=max_age_days)

    async def rotate_api_key(self, path: str):
        """Rotate API key secret"""
        # Generate new API key
        new_key = generate_secure_api_key()

        # Store new key
        self.vault.create_secret(path, {"api_key": new_key})

        # Update rotation timestamp
        self.rotation_schedule[path] = datetime.utcnow()

        logger.info("secret_rotated", path=path)

        # Notify services to reload secret
        await notify_services_of_rotation(path)

    async def rotate_all_secrets(self):
        """Rotate all secrets that need rotation"""
        for path in self.get_all_secret_paths():
            if self.should_rotate(path):
                await self.rotate_api_key(path)

def generate_secure_api_key(length: int = 32) -> str:
    """Generate cryptographically secure API key"""
    import secrets
    import string

    alphabet = string.ascii_letters + string.digits
    return ''.join(secrets.choice(alphabet) for _ in range(length))
```

---

## Compliance Frameworks

### GDPR (General Data Protection Regulation)

**Key Requirements**:
- **Right to access**: Users can request their data
- **Right to erasure**: "Right to be forgotten"
- **Data portability**: Export data in machine-readable format
- **Consent management**: Track data usage consent
- **Data breach notification**: Report breaches within 72 hours

```python
class GDPRCompliance:
    """Implement GDPR compliance features"""

    def __init__(self, db):
        self.db = db

    async def export_user_data(self, user_id: str) -> Dict[str, Any]:
        """Right to access: Export all user data"""
        user_data = {
            "user_info": await self.db.get_user(user_id),
            "models": await self.db.get_user_models(user_id),
            "datasets": await self.db.get_user_datasets(user_id),
            "training_jobs": await self.db.get_user_jobs(user_id),
            "predictions": await self.db.get_user_predictions(user_id),
        }

        logger.info("gdpr_data_export", user_id=user_id)
        return user_data

    async def delete_user_data(self, user_id: str):
        """Right to erasure: Delete all user data"""
        # Delete from all tables
        await self.db.delete_user(user_id)
        await self.db.delete_user_models(user_id)
        await self.db.delete_user_datasets(user_id)
        await self.db.delete_user_jobs(user_id)

        # Anonymize audit logs (can't delete)
        await self.db.anonymize_audit_logs(user_id)

        logger.info("gdpr_data_deletion", user_id=user_id)

    async def track_consent(self, user_id: str, purpose: str, granted: bool):
        """Track user consent for data processing"""
        await self.db.store_consent(
            user_id=user_id,
            purpose=purpose,
            granted=granted,
            timestamp=datetime.utcnow()
        )

        logger.info("gdpr_consent_recorded",
                   user_id=user_id,
                   purpose=purpose,
                   granted=granted)
```

### HIPAA (Health Insurance Portability and Accountability Act)

**Key Requirements**:
- Encrypt PHI (Protected Health Information)
- Access controls for PHI
- Audit all PHI access
- Business Associate Agreements (BAA)
- Breach notification

```python
class HIPAACompliance:
    """Implement HIPAA compliance for healthcare ML"""

    PHI_FIELDS = [
        "name", "address", "email", "phone", "ssn",
        "medical_record_number", "health_plan_number",
        "device_identifiers", "biometric_identifiers"
    ]

    def __init__(self, encryption_key: bytes):
        self.cipher = Fernet(encryption_key)

    def encrypt_phi(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Encrypt all PHI fields"""
        encrypted = data.copy()
        for field in self.PHI_FIELDS:
            if field in encrypted:
                value = str(encrypted[field])
                encrypted[field] = self.cipher.encrypt(value.encode()).decode()
        return encrypted

    def audit_phi_access(self, user: str, patient_id: str, purpose: str):
        """Log all PHI access (HIPAA requirement)"""
        logger.info(
            "hipaa_phi_access",
            user=user,
            patient_id=patient_id,
            purpose=purpose,
            timestamp=datetime.utcnow().isoformat()
        )

    async def check_authorized_access(
        self,
        user: str,
        patient_id: str
    ) -> bool:
        """Verify user authorized to access patient data"""
        # Check if user is treating physician
        # Check if user has valid reason to access (treatment, payment, operations)
        # Implement break-the-glass emergency access
        pass
```

### SOC 2 (Service Organization Control 2)

**Key Requirements**:
- Security policies and procedures
- Access controls and authentication
- System monitoring and incident response
- Change management
- Vendor management

```python
class SOC2Compliance:
    """Implement SOC 2 controls"""

    def log_system_change(
        self,
        change_type: str,
        description: str,
        approver: str,
        change_ticket: str
    ):
        """Log all system changes (SOC 2 control)"""
        logger.info(
            "soc2_system_change",
            change_type=change_type,
            description=description,
            approver=approver,
            change_ticket=change_ticket,
            timestamp=datetime.utcnow().isoformat()
        )

    def require_code_review(self):
        """Enforce code review before deployment"""
        # GitHub Actions workflow
        pass

    def monitor_vendor_access(self, vendor: str, access_type: str):
        """Track third-party vendor access"""
        logger.info(
            "soc2_vendor_access",
            vendor=vendor,
            access_type=access_type,
            timestamp=datetime.utcnow().isoformat()
        )
```

---

## Security Best Practices

### 1. Principle of Least Privilege

Grant minimum permissions needed to perform tasks.

```python
# Bad: Admin access for everyone
user.roles = ["admin"]

# Good: Granular permissions
user.roles = ["data-scientist"]  # Can train models but not deploy
```

### 2. Defense in Depth

Multiple layers of security controls.

```python
# Layer 1: Network security (firewall, VPC)
# Layer 2: Authentication (OAuth 2.0)
# Layer 3: Authorization (RBAC)
# Layer 4: Encryption (TLS, encryption at rest)
# Layer 5: Audit logging
# Layer 6: Anomaly detection
```

### 3. Secure Defaults

Default to most secure configuration.

```python
@dataclass
class SecurityConfig:
    encryption_enabled: bool = True  # Encrypt by default
    tls_required: bool = True        # Require HTTPS
    audit_logging: bool = True       # Log all actions
    mfa_required: bool = False       # Can be enabled later
```

### 4. Regular Security Audits

```python
class SecurityAuditor:
    """Automated security checks"""

    async def run_audit(self):
        findings = []

        # Check for weak passwords
        weak_passwords = await self.check_password_strength()
        if weak_passwords:
            findings.append({
                "severity": "high",
                "issue": "Weak passwords detected",
                "count": len(weak_passwords)
            })

        # Check for unused API keys
        unused_keys = await self.check_unused_api_keys(days=90)
        if unused_keys:
            findings.append({
                "severity": "medium",
                "issue": "Unused API keys",
                "count": len(unused_keys)
            })

        # Check for overprivileged users
        admins = await self.count_admin_users()
        if admins > 5:
            findings.append({
                "severity": "medium",
                "issue": f"Too many admin users: {admins}"
            })

        # Generate audit report
        return self.generate_report(findings)
```

### 5. Incident Response Plan

```python
class IncidentResponse:
    """Handle security incidents"""

    async def handle_incident(
        self,
        incident_type: str,
        severity: str,
        details: Dict[str, Any]
    ):
        # 1. Contain
        if severity == "critical":
            await self.disable_affected_accounts()
            await self.rotate_compromised_secrets()

        # 2. Investigate
        audit_logs = await self.collect_audit_logs(details["timeframe"])

        # 3. Notify
        await self.notify_security_team(incident_type, severity, details)
        if self.requires_user_notification(incident_type):
            await self.notify_affected_users(details["affected_users"])

        # 4. Remediate
        await self.apply_security_patches()
        await self.update_firewall_rules()

        # 5. Document
        await self.create_incident_report(
            incident_type, severity, details, audit_logs
        )

        # 6. Post-mortem
        await self.schedule_post_mortem()
```

---

## Summary

**Key Takeaways**:

1. **Authentication**: Use OAuth 2.0/JWT for user auth, API keys for service-to-service, mTLS for service mesh
2. **RBAC**: Implement role-based access control with resource-level permissions
3. **Audit Logging**: Log all security-relevant events in immutable, tamper-proof storage
4. **Encryption**: Encrypt data at rest and in transit, classify data sensitivity
5. **Model Security**: Defend against adversarial attacks, model extraction, data poisoning
6. **Secrets Management**: Use Vault or Kubernetes Secrets, rotate regularly
7. **Compliance**: Implement GDPR, HIPAA, SOC 2 controls as needed
8. **Best Practices**: Least privilege, defense in depth, secure defaults, regular audits

**Next Steps**:
- Complete hands-on exercises implementing RBAC and audit logging
- Set up encryption for datasets and models
- Configure secrets management with Vault
- Implement adversarial defenses for models
- Create incident response runbooks

---

**Further Reading**:
- [OWASP Top 10 for ML](https://owasp.org/www-project-machine-learning-security-top-10/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [GDPR Compliance Guide](https://gdpr.eu/)
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/index.html)

---

**Last Updated**: November 2, 2025 | **Version**: 1.0.0
