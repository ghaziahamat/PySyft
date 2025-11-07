# PySyft Technical Specification

**Version:** 0.9.6-beta.6
**Last Updated:** 2025-11-07
**Organization:** OpenMined
**License:** Apache 2.0

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [Core Components](#core-components)
4. [Data Model](#data-model)
5. [Service Layer](#service-layer)
6. [Storage Layer](#storage-layer)
7. [Security Architecture](#security-architecture)
8. [API Specifications](#api-specifications)
9. [Dependencies](#dependencies)
10. [Deployment Architecture](#deployment-architecture)
11. [Performance Characteristics](#performance-characteristics)
12. [Extensibility & Integration](#extensibility--integration)

---

## Executive Summary

### Purpose

PySyft is a privacy-preserving data science platform that enables secure computation on sensitive data without requiring direct access to that data. It implements remote data science workflows through structured transparency principles, allowing data owners to maintain complete control while enabling legitimate research and analysis.

### Key Capabilities

- **Remote Code Execution:** Execute approved code on private data without data movement
- **Twin Object Architecture:** Separate mock and private data representations
- **Policy-Based Access Control:** Fine-grained input and output policies
- **Multi-Party Computation:** Federated learning and distributed computation
- **Secure Enclaves:** Hardware-based trusted execution environments
- **Audit & Compliance:** Complete action tracking and logging

### Target Environments

- **Development:** Python processes, SQLite database
- **Production:** Docker containers, Kubernetes clusters, PostgreSQL
- **Secure:** Hardware enclaves (Intel SGX, AMD SEV)

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Python SDK  │  │  Web UI      │  │  CLI Tools   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ HTTPS/WebSocket (TLS 1.3)
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                      API Gateway Layer                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  FastAPI Application Server (Uvicorn/ASGI)              │   │
│  │  - REST Endpoints                                        │   │
│  │  - WebSocket Endpoints                                   │   │
│  │  - Authentication Middleware                             │   │
│  │  - Rate Limiting                                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                      Service Layer                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │   User     │  │  Dataset   │  │   Code     │  │  Request │  │
│  │  Service   │  │  Service   │  │  Service   │  │  Service │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │   Action   │  │   Policy   │  │    Job     │  │  Worker  │  │
│  │  Service   │  │  Service   │  │  Service   │  │  Service │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
│  └─────────────── (20+ Microservices) ──────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                      Persistence Layer                           │
│  ┌──────────────────────────┐  ┌──────────────────────────┐     │
│  │  Document Store          │  │  Blob Store              │     │
│  │  - SQLite (dev)          │  │  - Filesystem (dev)      │     │
│  │  - PostgreSQL (prod)     │  │  - SeaweedFS (dist)      │     │
│  │  - MongoDB (experimental)│  │  - Azure Blob            │     │
│  │                          │  │  - AWS S3                │     │
│  └──────────────────────────┘  └──────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    Execution Layer                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Worker Pools                                            │   │
│  │  - Python Workers (default)                              │   │
│  │  - Docker Workers (isolation)                            │   │
│  │  - Kubernetes Workers (scale)                            │   │
│  │  - ZMQ Queue System (messaging)                          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Architectural Principles

1. **Separation of Concerns:** Clear boundaries between client, API, service, storage, and execution layers
2. **Microservices Pattern:** Independent services with well-defined interfaces
3. **Event-Driven:** Asynchronous operations via message queues
4. **Stateless API:** Session state managed in persistent storage
5. **Pluggable Backends:** Abstract interfaces for storage and execution

### Communication Patterns

#### Synchronous (REST)
- User authentication
- Dataset browsing
- Request submission
- Configuration management

#### Asynchronous (WebSocket)
- Real-time notifications
- Job status updates
- Long-running operations
- Streaming results

#### Message Queue (ZMQ)
- Job distribution to workers
- Inter-service communication
- Event propagation

---

## Core Components

### 1. Server Types

#### Datasite Server

**Purpose:** Host datasets and execute approved code

**Characteristics:**
- Single-tenant architecture
- Full dataset management
- Code approval workflows
- Local or distributed storage
- Worker pool management

**Configuration:**
```python
{
    "name": str,           # Unique server identifier
    "type": "datasite",
    "port": int,           # HTTP port
    "database": {
        "type": "postgresql",
        "host": str,
        "port": int,
        "name": str,
    },
    "blob_storage": {
        "type": "seaweedfs",
        "host": str,
        "port": int,
    },
    "workers": {
        "type": "kubernetes",
        "replicas": int,
    }
}
```

#### Gateway Server

**Purpose:** Network hub connecting multiple datasites

**Characteristics:**
- Multi-tenant routing
- Network discovery
- Request aggregation
- State synchronization
- No data hosting

**Configuration:**
```python
{
    "name": str,
    "type": "gateway",
    "port": int,
    "datasites": [
        {"name": str, "url": str, "port": int}
    ],
    "sync_interval": int,  # seconds
}
```

#### Enclave Server

**Purpose:** Secure computation with hardware attestation

**Characteristics:**
- Trusted execution environment
- Hardware-based isolation (SGX/SEV)
- Remote attestation
- Encrypted memory
- Audit logging

**Configuration:**
```python
{
    "name": str,
    "type": "enclave",
    "attestation": {
        "type": "sgx",  # or "sev"
        "quote_provider": str,
    },
    "memory_encryption": true,
}
```

### 2. Client Types

#### DatasiteClient

**Purpose:** Connect to datasites

**Capabilities:**
- Dataset browsing
- Code submission
- Request management
- Asset access

**Methods:**
```python
client.datasets           # DatasetRegistry
client.code               # CodeService
client.requests           # RequestService
client.users              # UserService (admin)
client.api.services.*     # Direct service access
```

#### GatewayClient

**Purpose:** Connect to gateway networks

**Capabilities:**
- Network discovery
- Multi-site operations
- Federated requests

**Methods:**
```python
gateway.datasites         # List connected datasites
gateway.connect(site)     # Connect to specific datasite
gateway.broadcast(req)    # Send request to all sites
```

#### EnclaveClient

**Purpose:** Connect to secure enclaves

**Capabilities:**
- Attestation verification
- Secure code execution
- Encrypted communication

**Methods:**
```python
enclave.verify_attestation()
enclave.submit_secure_code()
enclave.get_audit_log()
```

### 3. Twin Object System

#### Architecture

```python
class TwinObject:
    """Combines private and mock data"""

    _private_data: Any      # Real data (owner only)
    _mock_data: Any         # Synthetic data (scientists)
    _schema: Schema         # Shared schema
    _permissions: Policy    # Access control

    def __getattr__(self, name):
        """Route attribute access based on permissions"""
        if has_permission(self.current_user, "private"):
            return getattr(self._private_data, name)
        else:
            return getattr(self._mock_data, name)
```

#### Data Flow

1. **Data Owner Creates Twin:**
   ```python
   asset = Asset(
       name="data",
       data=private_data,      # Real data
       mock=mock_data,         # Synthetic data
       schema=schema           # Shared schema
   )
   ```

2. **Data Scientist Develops with Mock:**
   ```python
   # Automatically receives mock data
   data = asset.data
   result = analyze(data)  # Runs on mock
   ```

3. **Execution on Private:**
   ```python
   # After approval, code runs on private
   request.approved = True
   result = execute_on_private(code, private_data)
   ```

#### Schema Validation

- Mock and private must share identical schema
- Type checking enforced
- Shape validation for arrays/dataframes
- Metadata consistency

---

## Data Model

### Entity-Relationship Diagram

```
┌──────────────┐
│    Server    │
└──────┬───────┘
       │
       ├──────┐
       │      │
┌──────▼───┐  │
│   User   │  │
└──────┬───┘  │
       │      │
       │      │
┌──────▼──────▼────┐
│    Dataset       │
└──────┬───────────┘
       │
       │ 1:N
       │
┌──────▼───────────┐
│     Asset        │
│  ┌────────────┐  │
│  │ Private    │  │
│  │ Data       │  │
│  └────────────┘  │
│  ┌────────────┐  │
│  │ Mock Data  │  │
│  └────────────┘  │
└──────┬───────────┘
       │
       │
┌──────▼───────────┐
│   UserCode       │
└──────┬───────────┘
       │
       │
┌──────▼───────────┐
│   Request        │
│  ┌────────────┐  │
│  │ Input      │  │
│  │ Policy     │  │
│  └────────────┘  │
│  ┌────────────┐  │
│  │ Output     │  │
│  │ Policy     │  │
│  └────────────┘  │
└──────┬───────────┘
       │
       │
┌──────▼───────────┐
│     Action       │
└──────────────────┘
       │
       │
┌──────▼───────────┐
│      Job         │
└──────────────────┘
       │
       │
┌──────▼───────────┐
│    Result        │
└──────────────────┘
```

### Core Entities

#### Server

```python
class Server:
    id: UID                      # Unique identifier
    name: str                    # Human-readable name
    server_type: ServerType      # Datasite, Gateway, Enclave
    version: str                 # PySyft version
    settings: ServerSettings     # Configuration
    created_at: datetime
    updated_at: datetime
```

#### User

```python
class User:
    id: UID
    email: str                   # Unique, validated
    name: str
    password_hash: str           # Argon2 hash
    role: UserRole               # Admin, DataOwner, DataScientist, Guest
    verified: bool
    permissions: List[Permission]
    created_at: datetime
    last_login: datetime
```

#### Dataset

```python
class Dataset:
    id: UID
    name: str
    description: str
    assets: List[Asset]
    contributors: List[Contributor]
    citation: str
    url: str
    tags: List[str]
    created_by: UID              # User ID
    created_at: datetime
    updated_at: datetime
```

#### Asset

```python
class Asset:
    id: UID
    name: str
    dataset_id: UID
    data_type: DataType          # DataFrame, Array, Image, etc.
    schema: Schema               # Data schema
    private_data_ptr: BlobPtr    # Pointer to private data
    mock_data_ptr: BlobPtr       # Pointer to mock data
    metadata: Dict[str, Any]
    shape: Tuple[int, ...]
    size_bytes: int
    created_at: datetime
```

#### UserCode

```python
class UserCode:
    id: UID
    user_id: UID                 # Author
    code: str                    # Source code
    input_policy: InputPolicy
    output_policy: OutputPolicy
    compiled_code: bytes         # RestrictedPython compiled
    dependencies: List[str]      # Required packages
    status: CodeStatus           # Pending, Approved, Denied
    created_at: datetime
```

#### Request

```python
class Request:
    id: UID
    user_id: UID                 # Requester
    code_id: UID                 # UserCode reference
    status: RequestStatus        # Pending, Approved, Denied
    reason: str                  # Justification
    reviewer_comment: str
    approved_by: UID             # Approver user ID
    approved_at: datetime
    created_at: datetime
```

#### Action

```python
class Action:
    id: UID
    user_id: UID
    target_id: UID               # Object being acted upon
    action_type: ActionType      # Read, Write, Execute, Delete
    method: str                  # Method name
    args: List[Any]              # Serialized arguments
    kwargs: Dict[str, Any]
    result_id: UID               # Result reference
    timestamp: datetime
```

#### Job

```python
class Job:
    id: UID
    user_id: UID
    action_id: UID
    status: JobStatus            # Queued, Running, Complete, Failed
    worker_id: str               # Worker that executed
    priority: int
    created_at: datetime
    started_at: datetime
    completed_at: datetime
    error: str                   # Error message if failed
```

### Type System

#### UID (Unique Identifier)

```python
class UID:
    """128-bit unique identifier"""
    value: UUID4

    @classmethod
    def generate(cls) -> UID:
        return cls(value=uuid4())
```

#### Result

```python
class Result[T]:
    """Type-safe result wrapper"""
    ok: Optional[T]
    err: Optional[Exception]

    def is_ok(self) -> bool
    def is_err(self) -> bool
    def unwrap(self) -> T
    def unwrap_or(self, default: T) -> T
```

---

## Service Layer

### Service Architecture

Each service follows a consistent pattern:

```python
class BaseService:
    """Base class for all services"""

    store: DocumentStore         # Persistence
    permissions: PermissionService

    @service_method
    def create(self, obj: T) -> Result[T]:
        """Create new entity"""

    @service_method
    def read(self, uid: UID) -> Result[T]:
        """Read entity by ID"""

    @service_method
    def update(self, uid: UID, obj: T) -> Result[T]:
        """Update entity"""

    @service_method
    def delete(self, uid: UID) -> Result[bool]:
        """Delete entity"""

    @service_method
    def list(self, filters: Dict) -> Result[List[T]]:
        """List entities with filters"""
```

### Core Services

#### 1. UserService

**Responsibilities:**
- User registration and authentication
- Password management
- Role and permission management
- Session management

**Methods:**
```python
create_user(name, email, password, role) -> User
authenticate(email, password) -> Session
reset_password(email) -> bool
update_role(user_id, new_role) -> User
get_permissions(user_id) -> List[Permission]
```

**Storage:**
- Users in document store
- Sessions in cache (Redis/in-memory)
- Passwords hashed with Argon2

#### 2. DatasetService

**Responsibilities:**
- Dataset CRUD operations
- Asset management
- Metadata handling
- Access control integration

**Methods:**
```python
create_dataset(dataset: Dataset) -> Dataset
add_asset(dataset_id, asset: Asset) -> Asset
get_dataset(dataset_id) -> Dataset
list_datasets(filters) -> List[Dataset]
delete_dataset(dataset_id) -> bool
```

**Storage:**
- Metadata in document store
- Data blobs in blob store
- Schema in document store

#### 3. CodeService

**Responsibilities:**
- User code submission
- Code compilation (RestrictedPython)
- Dependency validation
- Code versioning

**Methods:**
```python
submit_code(code: str, policies: Policies) -> UserCode
compile_code(code: UserCode) -> CompiledCode
validate_dependencies(code: UserCode) -> bool
get_code(code_id) -> UserCode
```

**Security:**
- RestrictedPython sandboxing
- AST validation
- Import restrictions
- Bytecode verification

#### 4. RequestService

**Responsibilities:**
- Request workflow management
- Approval/denial handling
- Notification triggering
- Request history

**Methods:**
```python
create_request(user_id, code_id, reason) -> Request
approve_request(request_id, approver_id) -> Request
deny_request(request_id, reason) -> Request
get_pending_requests() -> List[Request]
```

**Workflow:**
```
Submitted → Pending → (Approved | Denied)
                ↓
            Executing → (Completed | Failed)
```

#### 5. ActionService

**Responsibilities:**
- Track all operations
- Action serialization
- Audit trail generation
- Replay capability

**Methods:**
```python
log_action(action: Action) -> Action
get_actions(filters) -> List[Action]
replay_action(action_id) -> Result
generate_audit_log(start, end) -> AuditLog
```

**Storage:**
- All actions in append-only log
- Indexed by user, timestamp, target
- Immutable records

#### 6. PolicyService

**Responsibilities:**
- Policy definition
- Policy evaluation
- Custom policy compilation
- Policy versioning

**Methods:**
```python
create_policy(policy: Policy) -> Policy
evaluate_input_policy(policy, inputs) -> bool
evaluate_output_policy(policy, output) -> bool
compile_custom_policy(code) -> CompiledPolicy
```

**Policy Types:**

**Input Policies:**
```python
class ExactMatch(InputPolicy):
    """Exact asset matching"""
    required_assets: List[UID]

class CustomInputPolicy(InputPolicy):
    """User-defined validation"""
    code: str
    compiled: bytes
```

**Output Policies:**
```python
class SingleExecutionExactOutput(OutputPolicy):
    """One-time execution with specific output"""
    max_executions: int = 1

class CustomOutputPolicy(OutputPolicy):
    """User-defined output validation"""
    code: str
    compiled: bytes
```

#### 7. JobService

**Responsibilities:**
- Job queue management
- Worker assignment
- Status tracking
- Result collection

**Methods:**
```python
submit_job(action_id, priority) -> Job
assign_worker(job_id, worker_id) -> bool
update_status(job_id, status) -> Job
get_result(job_id) -> Result
cancel_job(job_id) -> bool
```

**Queue Architecture:**
```python
# Producer
job_service.submit(action) → ZMQ Queue

# Consumer (Worker)
while True:
    job = queue.receive()
    result = execute(job)
    job_service.complete(job.id, result)
```

#### 8. WorkerService

**Responsibilities:**
- Worker pool management
- Health monitoring
- Resource allocation
- Worker types (Python, Docker, K8s)

**Methods:**
```python
register_worker(worker_config) -> Worker
get_available_workers() -> List[Worker]
allocate_worker(job) -> Worker
health_check(worker_id) -> Health
```

**Worker Types:**

**Python Worker:**
```python
class PythonWorker:
    """In-process execution"""
    def execute(self, code, data):
        return exec(code, {"data": data})
```

**Docker Worker:**
```python
class DockerWorker:
    """Container isolation"""
    image: str
    cpu_limit: int
    memory_limit: str
    network_mode: str

    def execute(self, code, data):
        container = docker.run(
            image=self.image,
            command=["python", "-c", code],
            resources=self.limits
        )
        return container.logs()
```

**Kubernetes Worker:**
```python
class KubernetesWorker:
    """Scalable pods"""
    job_template: V1Job
    namespace: str

    def execute(self, code, data):
        job = create_job(self.job_template)
        wait_for_completion(job)
        return get_logs(job)
```

### Service Dependencies

```
UserService
    ↓
DatasetService → AssetService
    ↓
CodeService → PolicyService
    ↓
RequestService → NotificationService
    ↓
ActionService
    ↓
JobService → WorkerService
    ↓
ResultService
```

---

## Storage Layer

### Document Store

**Interface:**
```python
class DocumentStore(ABC):
    """Abstract document store interface"""

    @abstractmethod
    def set(self, key: UID, value: T) -> Result[T]:
        """Store document"""

    @abstractmethod
    def get(self, key: UID) -> Result[T]:
        """Retrieve document"""

    @abstractmethod
    def query(self, filters: Dict) -> Result[List[T]]:
        """Query documents"""

    @abstractmethod
    def delete(self, key: UID) -> Result[bool]:
        """Delete document"""

    @abstractmethod
    def update(self, key: UID, updates: Dict) -> Result[T]:
        """Update document"""
```

**Implementations:**

#### SQLiteDocumentStore

**Use Case:** Development, single-node

**Characteristics:**
- File-based storage
- ACID transactions
- No network overhead
- Limited concurrency

**Configuration:**
```python
{
    "type": "sqlite",
    "path": "/data/syft.db",
    "timeout": 30,
    "check_same_thread": false
}
```

**Schema:**
```sql
CREATE TABLE documents (
    uid TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    data BLOB NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_type ON documents(type);
CREATE INDEX idx_created ON documents(created_at);
```

#### PostgreSQLDocumentStore

**Use Case:** Production, multi-node

**Characteristics:**
- Client-server architecture
- High concurrency
- Replication support
- Advanced indexing

**Configuration:**
```python
{
    "type": "postgresql",
    "host": "localhost",
    "port": 5432,
    "database": "syft",
    "username": "syft_user",
    "password": "***",
    "pool_size": 20,
    "max_overflow": 40
}
```

**Schema:**
```sql
CREATE TABLE documents (
    uid UUID PRIMARY KEY,
    type VARCHAR(255) NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_type ON documents(type);
CREATE INDEX idx_data_gin ON documents USING GIN (data);
```

#### MongoDBDocumentStore

**Use Case:** Experimental, flexible schema

**Characteristics:**
- Native document model
- Flexible schema
- Horizontal scaling
- GridFS for large objects

**Configuration:**
```python
{
    "type": "mongodb",
    "uri": "mongodb://localhost:27017",
    "database": "syft",
    "collection": "documents"
}
```

### Blob Store

**Interface:**
```python
class BlobStore(ABC):
    """Abstract blob store interface"""

    @abstractmethod
    def put(self, data: bytes) -> BlobPtr:
        """Store blob"""

    @abstractmethod
    def get(self, ptr: BlobPtr) -> bytes:
        """Retrieve blob"""

    @abstractmethod
    def delete(self, ptr: BlobPtr) -> bool:
        """Delete blob"""

    @abstractmethod
    def exists(self, ptr: BlobPtr) -> bool:
        """Check existence"""
```

**Implementations:**

#### OnDiskBlobStore

**Configuration:**
```python
{
    "type": "disk",
    "path": "/data/blobs",
    "chunk_size": 1048576  # 1MB
}
```

#### SeaweedFSBlobStore

**Configuration:**
```python
{
    "type": "seaweedfs",
    "master": "seaweedfs-master:9333",
    "filers": ["filer1:8888", "filer2:8888"],
    "replication": "001"  # 1 replica
}
```

#### AzureBlobStore

**Configuration:**
```python
{
    "type": "azure",
    "account_name": "syftaccount",
    "account_key": "***",
    "container": "syft-blobs"
}
```

#### S3BlobStore

**Configuration:**
```python
{
    "type": "s3",
    "region": "us-east-1",
    "bucket": "syft-blobs",
    "access_key": "***",
    "secret_key": "***"
}
```

### Serialization

**Protocol:** Cap'n Proto

**Schema Definition:**
```capnp
struct Document {
    uid @0 :Data;
    type @1 :Text;
    data @2 :Data;
    createdAt @3 :Int64;
    updatedAt @4 :Int64;
}

struct Action {
    uid @0 :Data;
    userId @1 :Data;
    targetId @2 :Data;
    actionType @3 :Text;
    method @4 :Text;
    args @5 :List(Data);
    kwargs @6 :Data;
    resultId @7 :Data;
    timestamp @8 :Int64;
}
```

**Serialization Pipeline:**
```python
# Serialize
obj → Pydantic Model → Cap'n Proto Schema → Bytes

# Deserialize
Bytes → Cap'n Proto Schema → Pydantic Model → obj
```

---

## Security Architecture

### Authentication

#### Flow

```
1. User submits credentials (email, password)
   ↓
2. Server validates email format
   ↓
3. Server retrieves user from store
   ↓
4. Server verifies password hash (Argon2)
   ↓
5. Server generates session token (JWT or UUID)
   ↓
6. Client stores token
   ↓
7. Client includes token in subsequent requests
   ↓
8. Server validates token on each request
```

#### Password Hashing

**Algorithm:** Argon2id

**Parameters:**
```python
{
    "time_cost": 2,         # Iterations
    "memory_cost": 102400,  # 100 MB
    "parallelism": 8,       # Threads
    "hash_len": 32,         # Output bytes
    "salt_len": 16          # Salt bytes
}
```

**Implementation:**
```python
from argon2 import PasswordHasher

ph = PasswordHasher(
    time_cost=2,
    memory_cost=102400,
    parallelism=8
)

# Hash
password_hash = ph.hash(password)

# Verify
try:
    ph.verify(password_hash, password)
    # Valid
except:
    # Invalid
```

#### Session Management

**Token Type:** JWT (JSON Web Token)

**Claims:**
```python
{
    "sub": user_id,        # Subject (user ID)
    "email": email,
    "role": role,
    "iat": issued_at,      # Timestamp
    "exp": expires_at,     # Expiration
    "jti": token_id        # Unique token ID
}
```

**Signature:** HMAC-SHA256 or RSA

### Authorization

#### Role-Based Access Control (RBAC)

**Roles:**
```python
class UserRole(Enum):
    GUEST = "guest"                    # Read-only, limited datasets
    DATA_SCIENTIST = "data_scientist"  # Submit requests, view results
    DATA_OWNER = "data_owner"          # Approve requests, manage data
    ADMIN = "admin"                    # Full system access
```

**Permissions Matrix:**

| Resource | Guest | Data Scientist | Data Owner | Admin |
|----------|-------|----------------|------------|-------|
| View Public Datasets | ✓ | ✓ | ✓ | ✓ |
| View Private Datasets | ✗ | ✗ | ✓ | ✓ |
| Submit Code Requests | ✗ | ✓ | ✓ | ✓ |
| Approve Requests | ✗ | ✗ | ✓ | ✓ |
| Upload Datasets | ✗ | ✗ | ✓ | ✓ |
| Manage Users | ✗ | ✗ | ✗ | ✓ |
| Server Configuration | ✗ | ✗ | ✗ | ✓ |

#### Permission Checking

```python
def check_permission(user: User, resource: Resource, action: Action) -> bool:
    # Admin has all permissions
    if user.role == UserRole.ADMIN:
        return True

    # Check role-based permissions
    if (user.role, resource, action) in PERMISSION_MATRIX:
        return True

    # Check ownership
    if resource.owner_id == user.id:
        return True

    # Check custom permissions
    if has_custom_permission(user, resource, action):
        return True

    return False
```

### Code Sandboxing

#### RestrictedPython

**Restrictions:**
- No file I/O (`open`, `file`)
- No network access (`socket`, `urllib`)
- No subprocess execution (`os.system`, `subprocess`)
- No dynamic code execution (`eval`, `exec`)
- No module imports (except whitelisted)
- No attribute access on protected attributes (`__dict__`, `__class__`)

**Compilation:**
```python
from RestrictedPython import compile_restricted
from RestrictedPython.Guards import safe_builtins, guarded_iter_unpack_sequence

# Compile user code
compiled = compile_restricted(
    source_code,
    filename='<user_code>',
    mode='exec'
)

# Execute with restricted globals
restricted_globals = {
    '__builtins__': safe_builtins,
    '_iter_unpack_sequence_': guarded_iter_unpack_sequence,
    '_getiter_': iter,
    # Whitelist imports
    'pandas': pandas,
    'numpy': numpy,
}

exec(compiled.code, restricted_globals)
```

**Whitelisted Modules:**
- numpy
- pandas
- scikit-learn
- scipy
- matplotlib
- torch
- transformers
- opendp

#### Worker Isolation

**Docker Isolation:**
```yaml
security_opt:
  - no-new-privileges:true
  - seccomp=default.json
cap_drop:
  - ALL
cap_add:
  - NET_BIND_SERVICE
read_only: true
tmpfs:
  - /tmp
network_mode: none
user: "1000:1000"  # Non-root
```

**Resource Limits:**
```yaml
deploy:
  resources:
    limits:
      cpus: '2'
      memory: 4G
      pids: 100
    reservations:
      cpus: '1'
      memory: 2G
```

### Network Security

#### TLS Configuration

**Minimum Version:** TLS 1.3

**Cipher Suites:**
- TLS_AES_256_GCM_SHA384
- TLS_CHACHA20_POLY1305_SHA256
- TLS_AES_128_GCM_SHA256

**Certificate Validation:**
- Required for production
- Optional for development
- Certificate pinning supported

#### API Security

**Rate Limiting:**
```python
{
    "anonymous": "10/minute",
    "authenticated": "100/minute",
    "admin": "1000/minute"
}
```

**Request Validation:**
- JSON schema validation
- Input sanitization
- SQL injection prevention
- XSS prevention

**Headers:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
```

### Audit Logging

**Logged Events:**
- Authentication attempts
- Authorization failures
- Data access
- Code execution
- Configuration changes
- User management

**Log Format:**
```json
{
    "timestamp": "2025-11-07T12:34:56Z",
    "user_id": "uuid",
    "event_type": "code_execution",
    "resource_id": "uuid",
    "action": "execute",
    "status": "success",
    "ip_address": "192.168.1.100",
    "user_agent": "PySyft/0.9.6",
    "metadata": {}
}
```

**Retention:**
- Configurable (default: 1 year)
- Immutable storage
- Encrypted at rest

---

## API Specifications

### REST API

**Base URL:** `https://datasite.example.com/api/v2`

**Authentication:** Bearer token in Authorization header

#### Endpoints

##### Authentication

```
POST /auth/login
Request:
{
    "email": "user@example.com",
    "password": "password"
}

Response:
{
    "token": "jwt_token",
    "user": {
        "id": "uuid",
        "email": "user@example.com",
        "role": "data_scientist"
    }
}
```

##### Datasets

```
GET /datasets
Response:
[
    {
        "id": "uuid",
        "name": "Dataset Name",
        "description": "Description",
        "asset_count": 5,
        "created_at": "2025-01-01T00:00:00Z"
    }
]

GET /datasets/{id}
Response:
{
    "id": "uuid",
    "name": "Dataset Name",
    "description": "Description",
    "assets": [...],
    "contributors": [...],
    "metadata": {}
}

POST /datasets
Request:
{
    "name": "New Dataset",
    "description": "Description",
    "assets": [...]
}
```

##### Code Requests

```
POST /code/request
Request:
{
    "code": "def analyze(data): ...",
    "input_policy": {...},
    "output_policy": {...},
    "reason": "Research justification"
}

Response:
{
    "request_id": "uuid",
    "status": "pending",
    "created_at": "2025-11-07T12:00:00Z"
}

GET /requests/{id}
Response:
{
    "id": "uuid",
    "status": "approved",
    "code": "...",
    "result": {...}
}
```

### WebSocket API

**URL:** `wss://datasite.example.com/ws`

**Protocol:** JSON-RPC 2.0

#### Messages

**Subscribe to Notifications:**
```json
{
    "jsonrpc": "2.0",
    "method": "subscribe",
    "params": {
        "channel": "requests",
        "user_id": "uuid"
    },
    "id": 1
}
```

**Notification:**
```json
{
    "jsonrpc": "2.0",
    "method": "notification",
    "params": {
        "type": "request_approved",
        "request_id": "uuid",
        "message": "Your request has been approved"
    }
}
```

---

## Dependencies

### Core Dependencies

#### Security & Cryptography
- **bcrypt** (4.1.2) - Password hashing
- **pynacl** (1.5.0) - Encryption (libsodium)
- **argon2-cffi** (23.1.0) - Password hashing (Argon2)
- **RestrictedPython** (7.0) - Code sandboxing

#### Web Framework
- **fastapi** (>=0.111.0) - Web framework
- **uvicorn[standard]** (>=0.30.0) - ASGI server
- **requests** (2.32.3) - HTTP client

#### Data Processing
- **numpy** (>=1.23.5) - Numerical computing
- **pandas** (2.2.2) - DataFrames
- **pyarrow** (17.0.0) - Columnar data format

#### Serialization
- **pycapnp** (2.0.0) - Cap'n Proto serialization
- **pydantic[email]** (>=2.6.0) - Data validation

#### Storage
- **boto3** (1.34.56) - AWS SDK (S3)
- **azure-storage-blob** (12.19.1) - Azure Blob Storage
- **psycopg[binary]** (3.1.19) - PostgreSQL adapter
- **psycopg2-binary** (2.9.9) - PostgreSQL adapter
- **sqlalchemy** (2.0.32) - ORM

#### Orchestration
- **docker** (7.1.0) - Docker SDK
- **kr8s** (0.13.5) - Kubernetes client
- **PyYAML** (>=6.0.1) - YAML parsing

#### Messaging
- **pyzmq** (>=23.2.1,<=25.1.1) - ZeroMQ messaging

#### UI
- **ipywidgets** (8.1.2) - Jupyter widgets
- **itables** (1.7.1) - Interactive tables
- **matplotlib** (>=3.7.1,<3.9.1) - Plotting
- **rich** (>=13.7.1) - Terminal formatting
- **jinja2** (>=3.1.4) - Templating

#### Utilities
- **forbiddenfruit** (0.1.4) - Monkey patching
- **packaging** (>=23.0) - Version handling
- **tqdm** (>=4.66.4) - Progress bars
- **typeguard** (4.1.5) - Runtime type checking
- **typing_extensions** (>=4.12.0) - Type hints
- **tenacity** (8.3.0) - Retry logic
- **markdown** (3.5.2) - Markdown parsing
- **nh3** (0.2.17) - HTML sanitization

### Optional Dependencies

#### Data Science Extra
- **torch** (2.2.2) - Deep learning
- **transformers** (4.41.2) - NLP models
- **opendp** (0.9.2) - Differential privacy
- **evaluate** (0.4.2) - Model evaluation
- **recordlinkage** (0.16) - Record matching

#### Development
- **pytest** (<8) - Testing framework
- **pytest-cov** - Coverage reporting
- **pytest-xdist[psutil]** - Parallel testing
- **mypy** (1.10.0) - Type checking
- **ruff** (0.4.7) - Linting
- **pre-commit** (3.7.1) - Git hooks
- **bandit** (1.7.8) - Security linting

#### Telemetry
- **opentelemetry-api** (>=1.27.0) - Observability API
- **opentelemetry-sdk** (>=1.27.0) - Observability SDK
- **opentelemetry-exporter-otlp** (>=1.27.0) - OTLP exporter
- **opentelemetry-instrumentation-fastapi** (>=0.48b0) - FastAPI instrumentation
- **opentelemetry-instrumentation-sqlalchemy** (>=0.48b0) - SQLAlchemy instrumentation

### Python Version Requirements

**Minimum:** Python 3.10

**Recommended:** Python 3.12

**Tested:** 3.10, 3.11, 3.12

### System Dependencies

#### Linux
- libsodium-dev (for PyNaCl)
- libzmq3-dev (for PyZMQ)
- postgresql-dev (for psycopg)
- python3-dev

#### macOS
```bash
brew install libsodium zeromq postgresql
```

#### Windows
- Visual C++ Build Tools
- PostgreSQL (if using PostgreSQL backend)

---

## Deployment Architecture

### Development Deployment

**Components:**
- Python process (FastAPI + Uvicorn)
- SQLite database
- On-disk blob storage
- Python workers (in-process)

**Command:**
```python
import syft as sy
server = sy.orchestra.launch(
    name="dev-server",
    port=8080,
    dev_mode=True
)
```

**Resource Requirements:**
- 2GB RAM minimum
- 10GB disk space
- Single CPU core

### Production Deployment (Docker)

**Components:**
- Grid Backend container
- PostgreSQL container
- SeaweedFS containers (master, volume, filer)
- Traefik load balancer
- Frontend container (optional)

**Docker Compose:**
```yaml
version: '3.8'

services:
  backend:
    image: openmined/grid-backend:latest
    environment:
      - POSTGRES_HOST=postgres
      - SEAWEEDFS_HOST=seaweedfs-master
    depends_on:
      - postgres
      - seaweedfs-master
    ports:
      - "8080:8080"

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=syft
      - POSTGRES_USER=syft
      - POSTGRES_PASSWORD=changeme
    volumes:
      - postgres-data:/var/lib/postgresql/data

  seaweedfs-master:
    image: chrislusf/seaweedfs:latest
    command: "master -ip=seaweedfs-master -port=9333"

  seaweedfs-volume:
    image: chrislusf/seaweedfs:latest
    command: "volume -mserver=seaweedfs-master:9333 -port=8080"
    volumes:
      - seaweedfs-data:/data

  seaweedfs-filer:
    image: chrislusf/seaweedfs:latest
    command: "filer -master=seaweedfs-master:9333"

volumes:
  postgres-data:
  seaweedfs-data:
```

**Resource Requirements:**
- 8GB RAM minimum
- 100GB disk space
- 4 CPU cores

### Production Deployment (Kubernetes)

**Components:**
- StatefulSet for backend replicas
- PostgreSQL (managed or StatefulSet)
- SeaweedFS cluster
- Ingress controller
- Horizontal Pod Autoscaler
- Persistent Volume Claims

**Helm Chart:**
```yaml
# values.yaml
replicaCount: 3

image:
  repository: openmined/grid-backend
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: datasite.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: datasite-tls
      hosts:
        - datasite.example.com

postgresql:
  enabled: true
  auth:
    username: syft
    password: changeme
    database: syft
  primary:
    persistence:
      size: 100Gi

seaweedfs:
  enabled: true
  master:
    replicas: 3
  volume:
    replicas: 6
  filer:
    replicas: 2

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

resources:
  limits:
    cpu: 2
    memory: 4Gi
  requests:
    cpu: 1
    memory: 2Gi
```

**Resource Requirements:**
- 32GB RAM minimum (cluster)
- 500GB disk space
- 12 CPU cores (cluster)

### High Availability Configuration

**Database:**
- PostgreSQL replication (primary + replicas)
- Automated failover (Patroni)
- Connection pooling (PgBouncer)

**Storage:**
- SeaweedFS replication factor >= 2
- Cross-availability zone distribution
- Automatic volume balancing

**Application:**
- Multiple backend replicas (3+)
- Load balancing (Traefik/Nginx)
- Health checks and auto-recovery
- Rolling updates

**Backup:**
- Automated database backups (pg_dump + WAL archiving)
- Blob storage backups to S3/Azure
- Disaster recovery testing

---

## Performance Characteristics

### Throughput

**API Requests:**
- 1,000 req/s (single node)
- 10,000 req/s (10-node cluster)

**Dataset Upload:**
- 100 MB/s (on-disk storage)
- 500 MB/s (SeaweedFS)
- 1 GB/s (S3/Azure with multiple connections)

**Job Execution:**
- 10 jobs/s (Python workers)
- 100 jobs/s (Docker workers, 20 workers)
- 1,000 jobs/s (Kubernetes, 200 pods)

### Latency

**API Response Times (P50/P95/P99):**
- Authentication: 50ms / 100ms / 200ms
- Dataset list: 20ms / 50ms / 100ms
- Code submission: 100ms / 200ms / 500ms
- Job result: 50ms / 100ms / 200ms

**Job Execution:**
- Queue time: <1s (normal load)
- Startup time: 0.1s (Python) / 2s (Docker) / 5s (Kubernetes)
- Execution time: Depends on user code

### Scalability

**Horizontal Scaling:**
- Backend: Linear scaling up to 100 nodes
- Workers: Linear scaling up to 1000 workers
- Database: Vertical + read replicas

**Dataset Size:**
- Single asset: Up to 100GB
- Total storage: Unlimited (distributed storage)

**Concurrent Users:**
- 10,000 authenticated users
- 100 concurrent code executions

### Bottlenecks

**Database:**
- Writes to request table (approval workflow)
- Action log inserts (high volume)

**Mitigation:**
- Connection pooling
- Batch inserts
- Asynchronous logging

**Storage:**
- Large blob transfers
- Concurrent access to same asset

**Mitigation:**
- Distributed blob storage
- CDN/caching layer
- Chunked transfers

---

## Extensibility & Integration

### Custom Services

**Creating a Service:**
```python
from syft.service.service import AbstractService
from syft.store.document_store import DocumentStore

class CustomService(AbstractService):
    store: DocumentStore

    def __init__(self, store: DocumentStore):
        self.store = store

    @service_method
    def custom_operation(self, params) -> Result:
        # Implementation
        pass
```

**Registering Service:**
```python
from syft.server.server import Server

server = Server.launch()
server.add_service(CustomService)
```

### Custom Policies

**Input Policy:**
```python
from syft.service.policy.policy import InputPolicy

class CustomInputPolicy(InputPolicy):
    allowed_columns: List[str]

    def validate(self, inputs: Dict) -> bool:
        for key, value in inputs.items():
            if hasattr(value, 'columns'):
                if not all(col in self.allowed_columns for col in value.columns):
                    return False
        return True
```

**Output Policy:**
```python
from syft.service.policy.policy import OutputPolicy

class CustomOutputPolicy(OutputPolicy):
    min_aggregation: int = 100

    def validate(self, output: Any) -> bool:
        if isinstance(output, pd.DataFrame):
            return len(output) >= self.min_aggregation
        return True
```

### Plugins

**Plugin Interface:**
```python
from syft.plugins import Plugin

class MyPlugin(Plugin):
    name = "my_plugin"
    version = "1.0.0"

    def install(self, server: Server):
        # Register services, routes, etc.
        pass

    def uninstall(self, server: Server):
        # Cleanup
        pass
```

### API Extensions

**Custom Endpoints:**
```python
from syft.server.routes import api_endpoint

@api_endpoint(path="/custom/endpoint", methods=["GET"])
def custom_handler(request):
    return {"status": "ok"}
```

### Integration Points

**Authentication Providers:**
- OAuth2 (Google, GitHub, etc.)
- SAML
- LDAP/Active Directory

**Storage Backends:**
- Custom blob store implementation
- Custom document store implementation

**Worker Types:**
- Custom execution environments
- Integration with compute platforms (AWS Batch, GCP Cloud Run)

**Notification Channels:**
- Email (SMTP)
- Slack
- Webhooks
- Custom integrations

---

## Version Compatibility

**PySyft:** 0.9.6-beta.6

**Protocol Version:** 2.0

**Backward Compatibility:**
- Clients >= 0.9.0 compatible with servers >= 0.9.0
- Protocol versioning ensures forward compatibility
- Breaking changes increment major version

---

**Document Version:** 1.0
**Last Updated:** 2025-11-07
**Maintainer:** OpenMined Community
