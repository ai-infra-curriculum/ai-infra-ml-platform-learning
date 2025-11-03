# Module 09: Security & Governance - Exercises

Hands-on exercises for implementing security and governance in ML platforms.

**Total Time**: ~8 hours
**Difficulty**: Intermediate to Advanced
**Prerequisites**: Modules 01-08 completed

---

## Exercise Overview

| Exercise | Topic | Duration | Difficulty |
|----------|-------|----------|------------|
| 01 | JWT Authentication & Authorization | 90 min | Intermediate |
| 02 | Implement RBAC System | 90 min | Intermediate |
| 03 | Audit Logging System | 75 min | Intermediate |
| 04 | Data Encryption (At Rest & In Transit) | 90 min | Advanced |
| 05 | Secrets Management with Vault | 60 min | Intermediate |
| 06 | Adversarial Defense System | 90 min | Advanced |

---

## Exercise 01: JWT Authentication & Authorization

**Duration**: 90 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Implement JWT-based authentication
- Create user registration and login endpoints
- Protect API endpoints with JWT verification
- Implement token refresh mechanism
- Handle token expiration and revocation

### Requirements

Build a secure authentication system for your ML platform API.

**Features**:
1. User registration with password hashing (bcrypt)
2. Login endpoint returning JWT access and refresh tokens
3. Protected endpoints requiring valid JWT
4. Token refresh endpoint
5. Logout (token revocation)

### Setup

```bash
# Install dependencies
pip install fastapi==0.104.0 \
    uvicorn==0.24.0 \
    python-jose[cryptography]==3.3.0 \
    passlib[bcrypt]==1.7.4 \
    python-multipart==0.0.6

# Create project structure
mkdir -p exercise-01-jwt-auth
cd exercise-01-jwt-auth
```

### Implementation Guide

**Step 1: User Model and Database**

Create `models.py`:

```python
from pydantic import BaseModel, EmailStr
from typing import Optional
from datetime import datetime

class User(BaseModel):
    username: str
    email: EmailStr
    full_name: Optional[str] = None
    disabled: bool = False
    hashed_password: str

class UserInDB(User):
    created_at: datetime
    last_login: Optional[datetime] = None

class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class TokenData(BaseModel):
    username: Optional[str] = None

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str
    full_name: Optional[str] = None

class UserLogin(BaseModel):
    username: str
    password: str
```

**Step 2: Password Hashing and JWT Utilities**

Create `auth_utils.py`:

```python
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional
import os

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# JWT settings
SECRET_KEY = os.getenv("SECRET_KEY", "your-secret-key-change-in-production")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 7

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire, "type": "access"})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def create_refresh_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def decode_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None
```

**Step 3: Authentication Endpoints**

Create `main.py`:

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from typing import Dict
from datetime import datetime
import uuid

app = FastAPI(title="ML Platform Auth API")

# In-memory user database (replace with PostgreSQL in production)
users_db: Dict[str, UserInDB] = {}
revoked_tokens = set()  # Token blacklist

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    # Check if token is revoked
    if token in revoked_tokens:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token has been revoked"
        )

    payload = decode_token(token)
    if payload is None:
        raise credentials_exception

    username: str = payload.get("sub")
    token_type: str = payload.get("type")

    if username is None or token_type != "access":
        raise credentials_exception

    user = users_db.get(username)
    if user is None:
        raise credentials_exception

    if user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")

    return user

@app.post("/register", response_model=User)
async def register(user_create: UserCreate):
    """Register a new user"""
    # Check if user already exists
    if user_create.username in users_db:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Username already registered"
        )

    # Create user
    hashed_password = get_password_hash(user_create.password)
    user = UserInDB(
        username=user_create.username,
        email=user_create.email,
        full_name=user_create.full_name,
        hashed_password=hashed_password,
        disabled=False,
        created_at=datetime.utcnow(),
        last_login=None
    )

    users_db[user.username] = user

    # Don't return hashed password
    return User(**user.dict())

@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login and receive JWT tokens"""
    user = users_db.get(form_data.username)

    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Update last login
    user.last_login = datetime.utcnow()

    # Create tokens
    access_token = create_access_token(data={"sub": user.username})
    refresh_token = create_refresh_token(data={"sub": user.username})

    return Token(
        access_token=access_token,
        refresh_token=refresh_token,
        token_type="bearer"
    )

@app.post("/refresh", response_model=Token)
async def refresh_token(refresh_token: str):
    """Refresh access token using refresh token"""
    payload = decode_token(refresh_token)

    if payload is None or payload.get("type") != "refresh":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid refresh token"
        )

    username = payload.get("sub")
    user = users_db.get(username)

    if user is None:
        raise HTTPException(status_code=401, detail="User not found")

    # Create new tokens
    new_access_token = create_access_token(data={"sub": username})
    new_refresh_token = create_refresh_token(data={"sub": username})

    return Token(
        access_token=new_access_token,
        refresh_token=new_refresh_token,
        token_type="bearer"
    )

@app.post("/logout")
async def logout(token: str = Depends(oauth2_scheme)):
    """Logout by revoking token"""
    revoked_tokens.add(token)
    return {"message": "Successfully logged out"}

@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_user)):
    """Get current user information"""
    return current_user

@app.get("/protected")
async def protected_route(current_user: User = Depends(get_current_user)):
    """Example protected endpoint"""
    return {
        "message": f"Hello {current_user.username}!",
        "email": current_user.email
    }
```

### Testing

```bash
# Start server
uvicorn main:app --reload

# Register user
curl -X POST "http://localhost:8000/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "email": "alice@example.com",
    "password": "secure_password_123",
    "full_name": "Alice Smith"
  }'

# Login (get tokens)
curl -X POST "http://localhost:8000/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=alice&password=secure_password_123"

# Access protected endpoint
curl -X GET "http://localhost:8000/protected" \
  -H "Authorization: Bearer <access_token>"

# Refresh token
curl -X POST "http://localhost:8000/refresh?refresh_token=<refresh_token>"

# Logout
curl -X POST "http://localhost:8000/logout" \
  -H "Authorization: Bearer <access_token>"
```

### Success Criteria

- [ ] User registration creates hashed passwords (not plaintext)
- [ ] Login returns both access and refresh tokens
- [ ] Protected endpoints reject requests without valid JWT
- [ ] Token refresh generates new access token from valid refresh token
- [ ] Logout revokes tokens (subsequent requests fail)
- [ ] Expired tokens are rejected

---

## Exercise 02: Implement RBAC System

**Duration**: 90 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Design role-based access control system
- Implement permissions and roles
- Create authorization middleware
- Add resource-level permissions
- Test RBAC with multiple users and roles

### Requirements

Build a complete RBAC system with roles, permissions, and authorization checks.

**Roles**:
- `viewer`: Read-only access
- `data-scientist`: Train models, run experiments
- `ml-engineer`: Deploy models to production
- `admin`: Full access

**Resources**: models, datasets, jobs, experiments

### Implementation Guide

**Step 1: Define RBAC Models**

Create `rbac.py`:

```python
from enum import Enum
from dataclasses import dataclass
from typing import Set, List

class Resource(str, Enum):
    MODEL = "model"
    DATASET = "dataset"
    JOB = "job"
    EXPERIMENT = "experiment"

class Action(str, Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    DEPLOY = "deploy"
    EXECUTE = "execute"

@dataclass(frozen=True)
class Permission:
    resource: Resource
    action: Action

    def __str__(self):
        return f"{self.resource.value}:{self.action.value}"

    @classmethod
    def from_string(cls, perm_str: str):
        resource, action = perm_str.split(":")
        return cls(Resource(resource), Action(action))

@dataclass
class Role:
    name: str
    permissions: Set[Permission]
    description: str

# Define roles
ROLES = {
    "viewer": Role(
        name="viewer",
        permissions={
            Permission(Resource.MODEL, Action.READ),
            Permission(Resource.DATASET, Action.READ),
            Permission(Resource.JOB, Action.READ),
            Permission(Resource.EXPERIMENT, Action.READ),
        },
        description="Read-only access"
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
            Permission(Resource.EXPERIMENT, Action.READ),
            Permission(Resource.EXPERIMENT, Action.WRITE),
            Permission(Resource.EXPERIMENT, Action.DELETE),
        },
        description="Full access"
    ),
}

@dataclass
class User:
    username: str
    email: str
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
```

**Step 2: Authorization Middleware**

Add to `main.py`:

```python
from fastapi import Depends, HTTPException
from rbac import Permission, Resource, Action, User, ROLES

def require_permission(permission: Permission):
    """Dependency to enforce permissions"""
    async def permission_checker(user: User = Depends(get_current_user)):
        if not user.has_permission(permission):
            raise HTTPException(
                status_code=403,
                detail=f"Missing permission: {permission}"
            )
        return user
    return permission_checker

# Protected endpoints with RBAC
@app.get("/models")
async def list_models(
    user: User = Depends(require_permission(Permission(Resource.MODEL, Action.READ)))
):
    return {"models": ["fraud-detection-v1", "sentiment-analysis-v2"]}

@app.post("/models/{model_id}/deploy")
async def deploy_model(
    model_id: str,
    user: User = Depends(require_permission(Permission(Resource.MODEL, Action.DEPLOY)))
):
    # Deploy model
    return {"status": "deployed", "model_id": model_id, "deployed_by": user.username}

@app.delete("/datasets/{dataset_id}")
async def delete_dataset(
    dataset_id: str,
    user: User = Depends(require_permission(Permission(Resource.DATASET, Action.DELETE)))
):
    # Delete dataset
    return {"status": "deleted", "dataset_id": dataset_id}

@app.post("/jobs")
async def submit_job(
    job_config: dict,
    user: User = Depends(require_permission(Permission(Resource.JOB, Action.EXECUTE)))
):
    # Submit training job
    return {"job_id": "job-123", "status": "submitted", "submitted_by": user.username}

@app.get("/users/me/permissions")
async def get_my_permissions(user: User = Depends(get_current_user)):
    """List current user's permissions"""
    permissions = [str(p) for p in user.get_permissions()]
    return {
        "username": user.username,
        "roles": user.roles,
        "permissions": sorted(permissions)
    }
```

**Step 3: Test Different User Roles**

```python
# Create test users with different roles
test_users = {
    "alice": User(username="alice", email="alice@example.com", roles=["viewer"]),
    "bob": User(username="bob", email="bob@example.com", roles=["data-scientist"]),
    "charlie": User(username="charlie", email="charlie@example.com", roles=["ml-engineer"]),
    "admin": User(username="admin", email="admin@example.com", roles=["admin"]),
}
```

### Testing

```bash
# Test viewer (Alice) - can only read
curl -X GET "http://localhost:8000/models" \
  -H "Authorization: Bearer <alice_token>"

# Should succeed (read permission)

curl -X POST "http://localhost:8000/models/model-1/deploy" \
  -H "Authorization: Bearer <alice_token>"

# Should fail with 403 (no deploy permission)

# Test ML Engineer (Charlie) - can deploy
curl -X POST "http://localhost:8000/models/model-1/deploy" \
  -H "Authorization: Bearer <charlie_token>"

# Should succeed (has deploy permission)

# Test Admin - can do anything
curl -X DELETE "http://localhost:8000/datasets/dataset-1" \
  -H "Authorization: Bearer <admin_token>"

# Should succeed (admin has all permissions)
```

### Success Criteria

- [ ] Different roles have different permissions
- [ ] Viewers can read but not write/delete
- [ ] Data scientists can train models but not deploy
- [ ] ML engineers can deploy models
- [ ] Admins have full access
- [ ] Unauthorized actions return 403 Forbidden

---

## Exercise 03: Audit Logging System

**Duration**: 75 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Implement comprehensive audit logging
- Store logs in PostgreSQL
- Make logs immutable and tamper-proof
- Query audit logs for compliance
- Generate audit reports

### Requirements

Build an audit logging system that tracks all security-relevant events.

**Events to Log**:
- Authentication (login, logout, failed attempts)
- Authorization (access granted/denied)
- Resource operations (create, read, update, delete)
- Configuration changes

### Setup

```bash
# Install dependencies
pip install sqlalchemy==2.0.23 \
    psycopg2-binary==2.9.9 \
    structlog==24.1.0

# Start PostgreSQL with Docker
docker run -d \
  --name audit-db \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=audit_logs \
  -p 5432:5432 \
  postgres:16
```

### Implementation Guide

**Step 1: Audit Log Schema**

Create `audit_models.py`:

```python
from sqlalchemy import Column, String, DateTime, JSON, Text, Index, event
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
import hashlib
import uuid
import os

Base = declarative_base()

SECRET_KEY = os.getenv("SECRET_KEY", "your-hmac-secret")

class AuditLog(Base):
    __tablename__ = "audit_logs"

    id = Column(String, primary_key=True)
    event_type = Column(String, nullable=False, index=True)
    timestamp = Column(DateTime, nullable=False, index=True)
    user = Column(String, nullable=False, index=True)
    resource_type = Column(String, index=True)
    resource_id = Column(String, index=True)
    action = Column(String)
    result = Column(String)  # success, denied, error
    ip_address = Column(String)
    user_agent = Column(Text)
    metadata = Column(JSON)
    signature = Column(String)  # HMAC for tamper-proofing

    # Compound indexes for common queries
    __table_args__ = (
        Index('idx_user_timestamp', 'user', 'timestamp'),
        Index('idx_resource_timestamp', 'resource_type', 'resource_id', 'timestamp'),
    )

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        if not self.id:
            self.id = str(uuid.uuid4())
        if not self.timestamp:
            self.timestamp = datetime.utcnow()
        self.signature = self.compute_signature()

    def compute_signature(self) -> str:
        """Compute HMAC signature for tamper detection"""
        message = f"{self.id}{self.timestamp}{self.user}{self.resource_type}{self.resource_id}{self.action}"
        return hashlib.hmac(
            SECRET_KEY.encode(),
            message.encode(),
            "sha256"
        ).hexdigest()

    def verify_signature(self) -> bool:
        """Verify log entry hasn't been tampered with"""
        return self.signature == self.compute_signature()

# Prevent modifications to audit logs
@event.listens_for(AuditLog, 'before_update')
def prevent_update(mapper, connection, target):
    raise ValueError("Audit logs are immutable and cannot be modified")

@event.listens_for(AuditLog, 'before_delete')
def prevent_delete(mapper, connection, target):
    raise ValueError("Audit logs are immutable and cannot be deleted")
```

**Step 2: Audit Logger**

Create `audit_logger.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from audit_models import Base, AuditLog
from enum import Enum
from typing import Optional, Dict, Any
from datetime import datetime

class AuditEventType(str, Enum):
    # Authentication
    LOGIN_SUCCESS = "auth.login.success"
    LOGIN_FAILED = "auth.login.failed"
    LOGOUT = "auth.logout"

    # Authorization
    ACCESS_GRANTED = "authz.access.granted"
    ACCESS_DENIED = "authz.access.denied"

    # Resources
    MODEL_CREATED = "model.created"
    MODEL_UPDATED = "model.updated"
    MODEL_DELETED = "model.deleted"
    MODEL_DEPLOYED = "model.deployed"
    DATASET_ACCESSED = "dataset.accessed"
    DATASET_DELETED = "dataset.deleted"
    JOB_SUBMITTED = "job.submitted"

class AuditLogger:
    def __init__(self, database_url: str):
        self.engine = create_engine(database_url)
        Base.metadata.create_all(self.engine)
        self.SessionLocal = sessionmaker(bind=self.engine)

    def log(
        self,
        event_type: AuditEventType,
        user: str,
        resource_type: Optional[str] = None,
        resource_id: Optional[str] = None,
        action: Optional[str] = None,
        result: Optional[str] = None,
        ip_address: Optional[str] = None,
        user_agent: Optional[str] = None,
        metadata: Optional[Dict[str, Any]] = None
    ):
        """Log audit event"""
        session = self.SessionLocal()
        try:
            log_entry = AuditLog(
                event_type=event_type.value,
                timestamp=datetime.utcnow(),
                user=user,
                resource_type=resource_type,
                resource_id=resource_id,
                action=action,
                result=result,
                ip_address=ip_address,
                user_agent=user_agent,
                metadata=metadata
            )
            session.add(log_entry)
            session.commit()
        finally:
            session.close()

    def query_logs(
        self,
        user: Optional[str] = None,
        resource_type: Optional[str] = None,
        start_time: Optional[datetime] = None,
        end_time: Optional[datetime] = None,
        limit: int = 100
    ):
        """Query audit logs"""
        session = self.SessionLocal()
        try:
            query = session.query(AuditLog)

            if user:
                query = query.filter(AuditLog.user == user)
            if resource_type:
                query = query.filter(AuditLog.resource_type == resource_type)
            if start_time:
                query = query.filter(AuditLog.timestamp >= start_time)
            if end_time:
                query = query.filter(AuditLog.timestamp <= end_time)

            query = query.order_by(AuditLog.timestamp.desc()).limit(limit)

            return query.all()
        finally:
            session.close()

# Global audit logger instance
audit_logger = AuditLogger("postgresql://postgres:secret@localhost:5432/audit_logs")
```

**Step 3: Integrate with FastAPI**

Add to `main.py`:

```python
from audit_logger import audit_logger, AuditEventType
from fastapi import Request

@app.middleware("http")
async def audit_middleware(request: Request, call_next):
    """Log all requests"""
    start_time = datetime.utcnow()

    # Extract user from token
    user = "anonymous"
    try:
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if token:
            payload = decode_token(token)
            user = payload.get("sub", "anonymous")
    except:
        pass

    # Process request
    response = await call_next(request)

    # Determine event type and resource
    path = request.url.path
    method = request.method

    if "/models" in path and method == "POST":
        audit_logger.log(
            event_type=AuditEventType.MODEL_CREATED,
            user=user,
            resource_type="model",
            action="create",
            result="success" if response.status_code < 400 else "error",
            ip_address=request.client.host,
            user_agent=request.headers.get("User-Agent")
        )
    elif "/models" in path and "/deploy" in path:
        model_id = path.split("/")[2]
        audit_logger.log(
            event_type=AuditEventType.MODEL_DEPLOYED,
            user=user,
            resource_type="model",
            resource_id=model_id,
            action="deploy",
            result="success" if response.status_code < 400 else "denied",
            ip_address=request.client.host,
            user_agent=request.headers.get("User-Agent")
        )

    return response

@app.get("/audit/logs")
async def get_audit_logs(
    user: Optional[str] = None,
    resource_type: Optional[str] = None,
    limit: int = 100,
    current_user: User = Depends(require_permission(Permission(Resource.ADMIN, Action.READ)))
):
    """Query audit logs (admin only)"""
    logs = audit_logger.query_logs(
        user=user,
        resource_type=resource_type,
        limit=limit
    )

    return {
        "count": len(logs),
        "logs": [
            {
                "id": log.id,
                "event_type": log.event_type,
                "timestamp": log.timestamp.isoformat(),
                "user": log.user,
                "resource_type": log.resource_type,
                "resource_id": log.resource_id,
                "action": log.action,
                "result": log.result,
                "ip_address": log.ip_address
            }
            for log in logs
        ]
    }
```

### Testing

```bash
# Perform actions to generate audit logs
curl -X POST "http://localhost:8000/models/model-1/deploy" \
  -H "Authorization: Bearer <token>"

# Query audit logs (as admin)
curl -X GET "http://localhost:8000/audit/logs?user=alice&limit=50" \
  -H "Authorization: Bearer <admin_token>"

# Query logs for specific resource
curl -X GET "http://localhost:8000/audit/logs?resource_type=model" \
  -H "Authorization: Bearer <admin_token>"
```

### Success Criteria

- [ ] All API requests are logged to PostgreSQL
- [ ] Logs include user, timestamp, action, result, IP address
- [ ] Logs are immutable (cannot be modified or deleted)
- [ ] Each log has HMAC signature for tamper detection
- [ ] Admin can query logs by user, resource, time range
- [ ] Logs survive application restarts

---

## Exercise 04: Data Encryption (At Rest & In Transit)

**Duration**: 90 minutes
**Difficulty**: Advanced

### Learning Objectives

- Implement encryption at rest for datasets and models
- Enforce HTTPS/TLS for all API traffic
- Implement field-level encryption for sensitive data
- Use client-side encryption for maximum security
- Verify encryption with security tests

### Requirements

Implement comprehensive encryption for the ML platform.

**Encryption Requirements**:
1. All API traffic over HTTPS/TLS
2. Encrypt model files at rest (AES-256)
3. Encrypt sensitive database fields
4. Client-side encryption for highly sensitive data

### Setup

```bash
# Install dependencies
pip install cryptography==41.0.7 \
    pyopenssl==23.3.0

# Generate self-signed certificate for testing
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

### Implementation Guide

**Step 1: Enforce HTTPS**

Create `tls_config.py`:

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
import os

app = FastAPI()

# Redirect HTTP to HTTPS in production
if os.getenv("ENV") == "production":
    app.add_middleware(HTTPSRedirectMiddleware)

@app.middleware("http")
async def enforce_https(request: Request, call_next):
    """Enforce HTTPS in production"""
    if os.getenv("ENV") == "production" and request.url.scheme != "https":
        raise HTTPException(
            status_code=403,
            detail="HTTPS required in production"
        )
    return await call_next(request)

# Run with TLS
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=8443,
        ssl_keyfile="./key.pem",
        ssl_certfile="./cert.pem"
    )
```

**Step 2: File Encryption for Models**

Create `encryption.py`:

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import os

class FileEncryptor:
    """Encrypt/decrypt files using AES-256"""

    def __init__(self, password: str):
        # Derive 256-bit key from password
        salt = b'ml-platform-salt'  # In production: use random salt and store it
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
            backend=default_backend()
        )
        self.key = kdf.derive(password.encode())

    def encrypt_file(self, input_path: str, output_path: str):
        """Encrypt file"""
        # Generate random IV
        iv = os.urandom(16)

        cipher = Cipher(
            algorithms.AES(self.key),
            modes.CFB(iv),
            backend=default_backend()
        )
        encryptor = cipher.encryptor()

        with open(input_path, 'rb') as f_in:
            with open(output_path, 'wb') as f_out:
                # Write IV first
                f_out.write(iv)

                # Encrypt in chunks
                while True:
                    chunk = f_in.read(64 * 1024)  # 64KB chunks
                    if not chunk:
                        break
                    encrypted_chunk = encryptor.update(chunk)
                    f_out.write(encrypted_chunk)

                f_out.write(encryptor.finalize())

    def decrypt_file(self, input_path: str, output_path: str):
        """Decrypt file"""
        with open(input_path, 'rb') as f_in:
            # Read IV
            iv = f_in.read(16)

            cipher = Cipher(
                algorithms.AES(self.key),
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
@app.post("/models/upload")
async def upload_model(
    file: UploadFile,
    user: User = Depends(get_current_user)
):
    """Upload and encrypt model file"""
    # Save uploaded file
    temp_path = f"/tmp/{file.filename}"
    with open(temp_path, "wb") as f:
        f.write(await file.read())

    # Encrypt file
    encryptor = FileEncryptor(os.getenv("ENCRYPTION_PASSWORD"))
    encrypted_path = f"/models/{file.filename}.encrypted"
    encryptor.encrypt_file(temp_path, encrypted_path)

    # Remove temp file
    os.remove(temp_path)

    audit_logger.log(
        event_type=AuditEventType.MODEL_CREATED,
        user=user.username,
        resource_type="model",
        action="upload",
        result="success",
        metadata={"filename": file.filename, "encrypted": True}
    )

    return {"status": "uploaded", "encrypted": True}

@app.get("/models/{model_id}/download")
async def download_model(
    model_id: str,
    user: User = Depends(require_permission(Permission(Resource.MODEL, Action.READ)))
):
    """Download and decrypt model file"""
    encrypted_path = f"/models/{model_id}.encrypted"

    # Decrypt to temp file
    encryptor = FileEncryptor(os.getenv("ENCRYPTION_PASSWORD"))
    temp_path = f"/tmp/{model_id}"
    encryptor.decrypt_file(encrypted_path, temp_path)

    # Return file
    return FileResponse(temp_path, filename=model_id)
```

**Step 3: Database Field Encryption**

Create `encrypted_fields.py`:

```python
from sqlalchemy.types import TypeDecorator, Text
from cryptography.fernet import Fernet
import os

FIELD_ENCRYPTION_KEY = os.getenv("FIELD_ENCRYPTION_KEY").encode()

class EncryptedString(TypeDecorator):
    """Encrypted string column type"""

    impl = Text
    cache_ok = True

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.cipher = Fernet(FIELD_ENCRYPTION_KEY)

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

# Usage in models
class Dataset(Base):
    __tablename__ = "datasets"

    id = Column(String, primary_key=True)
    name = Column(String)
    # Encrypt sensitive field
    s3_credentials = Column(EncryptedString)
    api_key = Column(EncryptedString)
```

### Testing

```bash
# Test HTTPS enforcement
curl -k https://localhost:8443/models
# Should succeed

curl http://localhost:8000/models
# Should redirect to HTTPS or return 403

# Upload encrypted model
curl -k -X POST "https://localhost:8443/models/upload" \
  -H "Authorization: Bearer <token>" \
  -F "file=@model.pkl"

# Verify file is encrypted on disk
cat /models/model.pkl.encrypted
# Should show encrypted binary data (not readable)

# Download and decrypt
curl -k -X GET "https://localhost:8443/models/model.pkl/download" \
  -H "Authorization: Bearer <token>" \
  -o decrypted_model.pkl

# Verify decrypted file matches original
md5sum model.pkl decrypted_model.pkl
```

### Success Criteria

- [ ] All API traffic uses HTTPS/TLS
- [ ] Model files encrypted at rest with AES-256
- [ ] Sensitive database fields encrypted
- [ ] Encryption keys stored securely (not hardcoded)
- [ ] Encrypted files cannot be read without decryption
- [ ] Decrypted files match original files

---

## Exercise 05: Secrets Management with Vault

**Duration**: 60 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Deploy HashiCorp Vault
- Store and retrieve secrets
- Implement dynamic secret rotation
- Integrate Vault with ML platform API
- Use Vault for database credentials

### Setup

```bash
# Start Vault with Docker
docker run -d \
  --name vault \
  --cap-add=IPC_LOCK \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' \
  -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' \
  -p 8200:8200 \
  vault:1.15

# Install Python client
pip install hvac==2.0.0
```

### Implementation Guide

**Step 1: Vault Client**

Create `vault_client.py`:

```python
import hvac
from functools import lru_cache
from typing import Dict, Optional
import os

class VaultClient:
    """HashiCorp Vault client for secrets management"""

    def __init__(self, vault_url: str, token: str):
        self.client = hvac.Client(url=vault_url, token=token)

        if not self.client.is_authenticated():
            raise ValueError("Vault authentication failed")

        # Enable KV v2 secrets engine
        try:
            self.client.sys.enable_secrets_engine(
                backend_type='kv',
                path='ml-platform',
                options={'version': '2'}
            )
        except:
            # Already enabled
            pass

    def get_secret(self, path: str, key: str) -> str:
        """Retrieve secret from Vault"""
        try:
            secret = self.client.secrets.kv.v2.read_secret_version(
                path=path,
                mount_point='ml-platform'
            )
            return secret['data']['data'][key]
        except Exception as e:
            raise ValueError(f"Failed to retrieve secret: {e}")

    def create_secret(self, path: str, data: Dict[str, str]):
        """Store secret in Vault"""
        try:
            self.client.secrets.kv.v2.create_or_update_secret(
                path=path,
                secret=data,
                mount_point='ml-platform'
            )
        except Exception as e:
            raise ValueError(f"Failed to create secret: {e}")

    def delete_secret(self, path: str):
        """Delete secret from Vault"""
        self.client.secrets.kv.v2.delete_metadata_and_all_versions(
            path=path,
            mount_point='ml-platform'
        )

    def list_secrets(self, path: str = ""):
        """List all secrets at path"""
        try:
            result = self.client.secrets.kv.v2.list_secrets(
                path=path,
                mount_point='ml-platform'
            )
            return result['data']['keys']
        except:
            return []

@lru_cache()
def get_vault_client() -> VaultClient:
    return VaultClient(
        vault_url=os.getenv("VAULT_ADDR", "http://localhost:8200"),
        token=os.getenv("VAULT_TOKEN", "myroot")
    )
```

**Step 2: Integrate with Application**

Add to `main.py`:

```python
from vault_client import get_vault_client

vault = get_vault_client()

# Store API keys in Vault
@app.post("/secrets")
async def create_secret(
    path: str,
    data: Dict[str, str],
    user: User = Depends(require_permission(Permission(Resource.SECRET, Action.WRITE)))
):
    """Store secret in Vault"""
    vault.create_secret(path, data)

    audit_logger.log(
        event_type=AuditEventType.SECRET_CREATED,
        user=user.username,
        resource_type="secret",
        resource_id=path,
        action="create",
        result="success"
    )

    return {"status": "created", "path": path}

@app.get("/secrets/{path}")
async def get_secret(
    path: str,
    key: str,
    user: User = Depends(require_permission(Permission(Resource.SECRET, Action.READ)))
):
    """Retrieve secret from Vault"""
    value = vault.get_secret(path, key)

    audit_logger.log(
        event_type=AuditEventType.SECRET_ACCESSED,
        user=user.username,
        resource_type="secret",
        resource_id=path,
        action="read",
        result="success"
    )

    return {"key": key, "value": value}

# Use Vault for database credentials
def get_database_url():
    """Get database URL from Vault"""
    db_user = vault.get_secret("database", "username")
    db_password = vault.get_secret("database", "password")
    db_host = vault.get_secret("database", "host")
    db_name = vault.get_secret("database", "name")

    return f"postgresql://{db_user}:{db_password}@{db_host}/{db_name}"

# Use Vault for third-party API keys
def get_openai_api_key():
    return vault.get_secret("openai", "api_key")
```

**Step 3: Secret Rotation**

Create `secret_rotation.py`:

```python
from datetime import datetime, timedelta
from typing import Dict
import secrets
import string

class SecretRotationManager:
    """Automatically rotate secrets"""

    def __init__(self, vault_client: VaultClient):
        self.vault = vault_client
        self.rotation_schedule: Dict[str, datetime] = {}

    def should_rotate(self, path: str, max_age_days: int = 90) -> bool:
        """Check if secret needs rotation"""
        last_rotated = self.rotation_schedule.get(path)
        if not last_rotated:
            return True

        age = datetime.utcnow() - last_rotated
        return age > timedelta(days=max_age_days)

    async def rotate_api_key(self, path: str):
        """Rotate API key"""
        # Generate new key
        new_key = self.generate_secure_key(length=32)

        # Store in Vault
        self.vault.create_secret(path, {"api_key": new_key})

        # Update rotation timestamp
        self.rotation_schedule[path] = datetime.utcnow()

        # Notify services
        await self.notify_services(path)

        return new_key

    def generate_secure_key(self, length: int = 32) -> str:
        """Generate cryptographically secure API key"""
        alphabet = string.ascii_letters + string.digits
        return ''.join(secrets.choice(alphabet) for _ in range(length))

    async def notify_services(self, path: str):
        """Notify services to reload secret"""
        # Send notification to services using this secret
        pass

# Scheduled rotation (e.g., with APScheduler)
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()
rotation_manager = SecretRotationManager(get_vault_client())

@scheduler.scheduled_job('cron', day='1', hour='0')  # First day of month
async def rotate_secrets():
    """Rotate all secrets monthly"""
    for path in ["openai", "aws", "databricks"]:
        if rotation_manager.should_rotate(path):
            await rotation_manager.rotate_api_key(path)

scheduler.start()
```

### Testing

```bash
# Store secret
curl -X POST "http://localhost:8000/secrets?path=openai" \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{"api_key": "sk-abc123..."}'

# Retrieve secret
curl -X GET "http://localhost:8000/secrets/openai?key=api_key" \
  -H "Authorization: Bearer <admin_token>"

# Verify in Vault UI
open http://localhost:8200/ui
# Token: myroot
```

### Success Criteria

- [ ] Vault running and accessible
- [ ] Secrets stored securely in Vault
- [ ] Application retrieves secrets from Vault (not environment variables)
- [ ] All secret access is audit logged
- [ ] Secrets can be rotated without application restart

---

## Exercise 06: Adversarial Defense System

**Duration**: 90 minutes
**Difficulty**: Advanced

### Learning Objectives

- Implement adversarial attack detection
- Apply adversarial training to models
- Add rate limiting to prevent model extraction
- Detect data poisoning attempts
- Implement model watermarking

### Requirements

Build defenses against adversarial attacks on ML models.

**Defenses**:
1. Rate limiting to prevent model extraction
2. Input validation to detect adversarial examples
3. Query pattern analysis
4. Adversarial training
5. Model watermarking

### Implementation Guide

**Step 1: Rate Limiting**

Create `rate_limiter.py`:

```python
from fastapi import Request, HTTPException
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from collections import defaultdict
from datetime import datetime, timedelta
from typing import List, Dict
import hashlib

limiter = Limiter(key_func=get_remote_address)

# Track query patterns
query_tracker: Dict[str, List[Dict]] = defaultdict(list)

def track_query(user: str, input_data: any):
    """Track queries for anomaly detection"""
    query_tracker[user].append({
        "timestamp": datetime.utcnow(),
        "input_hash": hashlib.sha256(str(input_data).encode()).hexdigest()
    })

    # Keep only last 1000 queries per user
    if len(query_tracker[user]) > 1000:
        query_tracker[user] = query_tracker[user][-1000:]

def is_extraction_attack(user: str, window_minutes: int = 60) -> bool:
    """Detect model extraction attempts"""
    queries = query_tracker.get(user, [])

    if len(queries) < 100:
        return False

    # Get recent queries
    cutoff = datetime.utcnow() - timedelta(minutes=window_minutes)
    recent = [q for q in queries if q["timestamp"] > cutoff]

    if len(recent) < 100:
        return False

    # Check for suspicious patterns

    # 1. High query rate
    time_span = (recent[-1]["timestamp"] - recent[0]["timestamp"]).seconds
    if time_span < 60:  # 100 queries in 1 minute
        return True

    # 2. Almost all unique inputs (random sampling)
    unique_inputs = len(set(q["input_hash"] for q in recent))
    if unique_inputs / len(recent) > 0.95:
        return True

    # 3. Sequential pattern (grid search)
    # TODO: Implement sequential pattern detection

    return False

@app.post("/predict")
@limiter.limit("100/hour")  # Max 100 predictions per hour
async def predict(
    request: Request,
    data: PredictionRequest,
    user: User = Depends(get_current_user)
):
    """Predict with rate limiting and extraction detection"""

    # Track query
    track_query(user.username, data.dict())

    # Check for extraction attack
    if is_extraction_attack(user.username):
        audit_logger.log(
            event_type="security.model_extraction_detected",
            user=user.username,
            resource_type="model",
            action="predict",
            result="blocked",
            ip_address=request.client.host,
            metadata={"reason": "suspected_model_extraction"}
        )

        raise HTTPException(
            status_code=429,
            detail="Suspicious activity detected. Account temporarily blocked."
        )

    # Make prediction
    prediction = model.predict(data.features)

    return {"prediction": prediction}
```

**Step 2: Input Validation**

Create `adversarial_detection.py`:

```python
import numpy as np
from typing import Dict, Any
import torch

class AdversarialDetector:
    """Detect adversarial examples"""

    def __init__(self, model, epsilon: float = 0.1):
        self.model = model
        self.epsilon = epsilon

    def is_adversarial(self, input_data: np.ndarray) -> bool:
        """Detect if input is adversarial"""

        # 1. Check input statistics
        if self.has_suspicious_statistics(input_data):
            return True

        # 2. Check prediction confidence
        if self.has_low_confidence(input_data):
            return True

        # 3. Check sensitivity to perturbations
        if self.is_highly_sensitive(input_data):
            return True

        return False

    def has_suspicious_statistics(self, input_data: np.ndarray) -> bool:
        """Check if input has unusual statistical properties"""
        mean = np.mean(input_data)
        std = np.std(input_data)

        # Example: Check if statistics are within expected range
        if mean < -3 or mean > 3:
            return True
        if std < 0.1 or std > 3:
            return True

        return False

    def has_low_confidence(self, input_data: np.ndarray) -> bool:
        """Check if model has low confidence"""
        with torch.no_grad():
            prediction = self.model(torch.tensor(input_data))
            confidence = torch.max(torch.softmax(prediction, dim=-1))

        # Suspicious if confidence is too low
        return confidence < 0.5

    def is_highly_sensitive(self, input_data: np.ndarray) -> bool:
        """Check if prediction changes dramatically with small perturbation"""
        original_pred = self.model(torch.tensor(input_data))

        # Add small random perturbation
        perturbation = np.random.normal(0, self.epsilon, input_data.shape)
        perturbed_input = input_data + perturbation
        perturbed_pred = self.model(torch.tensor(perturbed_input))

        # Check if prediction changed significantly
        diff = torch.abs(original_pred - perturbed_pred).max()

        return diff > 0.3  # Threshold

# Usage in endpoint
detector = AdversarialDetector(model)

@app.post("/predict")
async def predict_with_detection(
    data: PredictionRequest,
    user: User = Depends(get_current_user)
):
    # Check for adversarial input
    if detector.is_adversarial(np.array(data.features)):
        audit_logger.log(
            event_type="security.adversarial_input_detected",
            user=user.username,
            resource_type="model",
            action="predict",
            result="blocked",
            metadata={"reason": "adversarial_input"}
        )

        raise HTTPException(
            status_code=400,
            detail="Invalid input detected"
        )

    # Make prediction
    prediction = model.predict(data.features)
    return {"prediction": prediction}
```

**Step 3: Adversarial Training**

Create `adversarial_training.py`:

```python
import torch
import torch.nn.functional as F

def fgsm_attack(model, input_data, target, epsilon=0.01):
    """Fast Gradient Sign Method attack"""
    input_data.requires_grad = True

    output = model(input_data)
    loss = F.cross_entropy(output, target)
    loss.backward()

    # Create adversarial example
    perturbation = epsilon * input_data.grad.sign()
    adversarial_input = input_data + perturbation

    return adversarial_input

def adversarial_training(model, dataloader, epochs=10, epsilon=0.01):
    """Train model on both clean and adversarial examples"""
    optimizer = torch.optim.Adam(model.parameters())

    for epoch in range(epochs):
        for inputs, labels in dataloader:
            # Generate adversarial examples
            adv_inputs = fgsm_attack(model, inputs.clone(), labels, epsilon)

            # Train on both
            clean_output = model(inputs)
            adv_output = model(adv_inputs.detach())

            loss = (
                F.cross_entropy(clean_output, labels) +
                F.cross_entropy(adv_output, labels)
            ) / 2

            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

        print(f"Epoch {epoch+1}/{epochs}, Loss: {loss.item():.4f}")

    return model

# Train robust model
robust_model = adversarial_training(model, train_loader)
```

**Step 4: Model Watermarking**

Create `watermarking.py`:

```python
import torch
import numpy as np

def generate_watermark(num_samples: int = 100):
    """Generate watermark data"""
    watermark_inputs = []
    watermark_outputs = []

    for _ in range(num_samples):
        # Generate random input
        input_data = np.random.randn(input_size)

        # Generate specific output (not related to data)
        output = np.random.randint(0, num_classes)

        watermark_inputs.append(input_data)
        watermark_outputs.append(output)

    return watermark_inputs, watermark_outputs

def embed_watermark(model, watermark_inputs, watermark_outputs, epochs=50):
    """Embed watermark into model"""
    optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

    for epoch in range(epochs):
        total_loss = 0
        for input_data, target in zip(watermark_inputs, watermark_outputs):
            input_tensor = torch.tensor(input_data, dtype=torch.float32).unsqueeze(0)
            target_tensor = torch.tensor([target])

            output = model(input_tensor)
            loss = F.cross_entropy(output, target_tensor)

            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

            total_loss += loss.item()

        if (epoch + 1) % 10 == 0:
            print(f"Watermark embedding epoch {epoch+1}, Loss: {total_loss/len(watermark_inputs):.4f}")

    return model

def verify_watermark(model, watermark_inputs, watermark_outputs, threshold=0.9):
    """Verify if model contains watermark"""
    correct = 0

    for input_data, expected_output in zip(watermark_inputs, watermark_outputs):
        input_tensor = torch.tensor(input_data, dtype=torch.float32).unsqueeze(0)

        with torch.no_grad():
            output = model(input_tensor)
            prediction = torch.argmax(output, dim=-1).item()

        if prediction == expected_output:
            correct += 1

    accuracy = correct / len(watermark_inputs)
    print(f"Watermark accuracy: {accuracy:.2%}")

    return accuracy >= threshold

# Usage
watermark_inputs, watermark_outputs = generate_watermark(100)
watermarked_model = embed_watermark(model, watermark_inputs, watermark_outputs)

# Verify ownership
if verify_watermark(suspected_model, watermark_inputs, watermark_outputs):
    print("Model theft detected!")
```

### Testing

```bash
# Test rate limiting
for i in {1..150}; do
  curl -X POST "http://localhost:8000/predict" \
    -H "Authorization: Bearer <token>" \
    -H "Content-Type: application/json" \
    -d '{"features": [1, 2, 3, 4, 5]}'
done
# Should block after 100 requests

# Test adversarial detection
curl -X POST "http://localhost:8000/predict" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"features": [999, 999, 999, 999, 999]}'  # Suspicious input
# Should reject as adversarial
```

### Success Criteria

- [ ] Rate limiting blocks excessive queries
- [ ] System detects model extraction attempts
- [ ] Adversarial inputs are detected and rejected
- [ ] Model trained with adversarial examples is more robust
- [ ] Watermark successfully embedded and verified
- [ ] All security events are audit logged

---

## Additional Resources

### Documentation
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [HashiCorp Vault](https://www.vaultproject.io/docs)
- [OWASP Top 10 for ML](https://owasp.org/www-project-machine-learning-security-top-10/)
- [Cryptography in Python](https://cryptography.io/)

### Papers
- [Adversarial Machine Learning at Scale (Google, 2017)](https://arxiv.org/abs/1611.01236)
- [Model Extraction Attacks (Tramèr et al., 2016)](https://arxiv.org/abs/1609.02943)
- [Data Poisoning Attacks (Biggio et al., 2012)](https://arxiv.org/abs/1206.6389)

### Tools
- [CleverHans](https://github.com/cleverhans-lab/cleverhans) - Adversarial examples library
- [Adversarial Robustness Toolbox](https://github.com/Trusted-AI/adversarial-robustness-toolbox)
- [TextAttack](https://github.com/QData/TextAttack) - NLP adversarial attacks

---

**Last Updated**: November 2, 2025 | **Version**: 1.0.0
