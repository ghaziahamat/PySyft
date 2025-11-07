# PySyft - Comprehensive Guide

## Overview

**PySyft** is a Python library that enables **data science on data you are not allowed to see**. It allows researchers and data scientists to perform statistical analysis and machine learning on non-public or sensitive data without ever seeing or obtaining a copy of that data.

**Tagline:** "Data Science on data you are not allowed to see"

**Version:** 0.9.6-beta.6

**Repository:** https://github.com/OpenMined/PySyft

**License:** Apache 2.0

---

## Table of Contents

1. [Core Concept](#core-concept)
2. [Key Features](#key-features)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [Architecture Overview](#architecture-overview)
6. [Main Components](#main-components)
7. [Usage Patterns](#usage-patterns)
8. [API Reference](#api-reference)
9. [Deployment Options](#deployment-options)
10. [Security Model](#security-model)
11. [Use Cases](#use-cases)
12. [Best Practices](#best-practices)
13. [Troubleshooting](#troubleshooting)

---

## Core Concept

PySyft implements **Remote Data Science** through "Datasites" - platforms designed with structured transparency principles that enable:

- **Data owners** to control how their data is protected while allowing scientists to use it within acceptable boundaries
- **Data scientists** to write and test code on synthetic/mock data, then request execution on real private data
- **Privacy-preserving computation** without requiring cryptographic expertise

### The Twin Object Pattern

The core innovation is the **TwinObject** which combines:
- **Private data** - Real sensitive data (only accessible to data owner)
- **Mock data** - Synthetic data with the same schema (accessible to data scientists)

This allows data scientists to:
1. Develop and test code using mock data
2. Submit code for execution on private data
3. Receive results based on privacy policies

---

## Key Features

### 1. Privacy-Preserving Computation
- Execute code on private data without direct access
- Twin objects (mock + private data) for development and execution
- Differential privacy integration (OpenDP)
- Secure enclaves with hardware attestation

### 2. Request & Approval Workflows
- Data scientists submit code requests
- Data owners review and approve/deny
- Granular control over data access
- Audit trails for compliance

### 3. Policy System
- **Input policies** - Control which data assets code can access
- **Output policies** - Control what results can be released
- Custom policy definitions
- Automated policy enforcement

### 4. Multi-Party Collaboration
- Gateway servers connect multiple datasites
- State synchronization across nodes
- Federated learning scenarios
- Network-wide request propagation

### 5. Flexible Deployment
- Python orchestration API
- Docker containers
- Kubernetes/Helm charts
- Secure enclave support

### 6. Rich Client APIs
- Pythonic interface for data scientists
- Web-based admin interface for data owners
- CLI tools for automation
- IPython/Jupyter integration

### 7. Action Tracking
- All operations tracked as serializable actions
- Complete audit trail
- Reproducible computations
- Compliance support

---

## Installation

### Basic Installation

```bash
pip install -U syft
```

### Installation with Data Science Dependencies

```bash
pip install -U "syft[data_science]"
```

This includes: numpy, pandas, torch, transformers, opendp, and more.

### Development Installation

```bash
git clone https://github.com/OpenMined/PySyft.git
cd PySyft
pip install -e "packages/syft[dev]"
```

### Requirements
- Python 3.10 or higher
- Linux, macOS, or Windows (with Docker)
- 4GB+ RAM recommended for server deployments

---

## Quick Start

### For Data Scientists

```python
import syft as sy

# Connect to a datasite
client = sy.login(
    url="datasite.example.com",
    port=8080,
    email="scientist@example.com",
    password="password"
)

# Browse available datasets
datasets = client.datasets

# Get a dataset
dataset = client.datasets[0]
asset = dataset.assets[0]

# Access mock data for development
mock_data = asset.mock

# Develop code using mock data
@sy.syft_function()
def analyze_data(data):
    import numpy as np
    return np.mean(data)

# Test with mock data
result = analyze_data(data=mock_data)
print(f"Mock result: {result}")

# Submit request to run on private data
request = client.code.request_code_execution(analyze_data)

# Check request status
request.status

# Once approved, get the result
result = request.accept_by_depositing_result()
print(f"Private result: {result}")
```

### For Data Owners

```python
import syft as sy
import pandas as pd
import numpy as np

# Launch a datasite
server = sy.orchestra.launch(
    name="my-datasite",
    port=8080,
    server_type="datasite",
    dev_mode=True,
    reset=True
)

# Login as admin
client = sy.login(
    url="localhost",
    port=8080,
    email="info@openmined.org",  # default admin
    password="changethis"
)

# Create a dataset with mock and private data
private_data = pd.DataFrame({
    "age": [25, 30, 35, 40, 45],
    "income": [50000, 60000, 70000, 80000, 90000]
})

mock_data = pd.DataFrame({
    "age": [28, 32, 38, 42, 47],
    "income": [52000, 62000, 72000, 82000, 92000]
})

# Upload dataset
dataset = sy.Dataset(
    name="Income Dataset",
    description="Salary data for research"
)

dataset.add_asset(
    name="salary_data",
    data=private_data,
    mock=mock_data
)

client.upload_dataset(dataset)

# Review incoming requests
requests = client.requests

# Approve a request
request = requests[0]
request.approve()

# Or deny with a reason
# request.deny(reason="Insufficient privacy guarantees")
```

---

## Architecture Overview

### System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Data Scientists                       │
│                  (Client Applications)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
                        │ HTTPS/WebSocket
                        │
┌───────────────────────▼─────────────────────────────────┐
│                     Gateway Server                       │
│              (Optional Network Hub)                      │
└─────────┬──────────────────────────────────┬────────────┘
          │                                  │
          │ HTTPS                            │ HTTPS
          │                                  │
┌─────────▼──────────┐            ┌─────────▼──────────┐
│   Datasite Node 1  │            │   Datasite Node 2  │
│  ┌──────────────┐  │            │  ┌──────────────┐  │
│  │   FastAPI    │  │            │  │   FastAPI    │  │
│  │   Server     │  │            │  │   Server     │  │
│  └──────┬───────┘  │            │  └──────┬───────┘  │
│         │          │            │         │          │
│  ┌──────▼───────┐  │            │  ┌──────▼───────┐  │
│  │  Services    │  │            │  │  Services    │  │
│  │  (20+ micro) │  │            │  │  (20+ micro) │  │
│  └──────┬───────┘  │            │  └──────┬───────┘  │
│         │          │            │         │          │
│  ┌──────▼───────┐  │            │  ┌──────▼───────┐  │
│  │   Store      │  │            │  │   Store      │  │
│  │ (DB + Blob)  │  │            │  │ (DB + Blob)  │  │
│  └──────────────┘  │            │  └──────────────┘  │
│                    │            │                    │
│  ┌──────────────┐  │            │  ┌──────────────┐  │
│  │  Job Queue   │  │            │  │  Job Queue   │  │
│  │   Workers    │  │            │  │   Workers    │  │
│  └──────────────┘  │            │  └──────────────┘  │
└────────────────────┘            └────────────────────┘
```

### Component Layers

1. **Client Layer** - Python/Jupyter interface for users
2. **API Layer** - FastAPI REST/WebSocket endpoints
3. **Service Layer** - Microservices for business logic
4. **Store Layer** - Persistence (SQLite/PostgreSQL/MongoDB)
5. **Worker Layer** - Job execution (Python/Docker/Kubernetes)

---

## Main Components

### Server Types

#### 1. Datasite
- Hosts datasets and executes approved code
- Primary server type for data owners
- Single-tenant model (one organization)

#### 2. Gateway
- Network hub connecting multiple datasites
- Enables multi-party computation
- Routes requests across the network

#### 3. Enclave
- Secure computation environment
- Hardware attestation (Intel SGX, AMD SEV)
- Zero-knowledge architecture

### Core Services

The service layer includes 20+ microservices:

| Service | Purpose |
|---------|---------|
| `action` | Track and execute operations on data |
| `dataset` | Manage datasets and assets |
| `code` | User code submission and execution |
| `request` | Permission request workflow |
| `policy` | Privacy policies and access control |
| `user` | User authentication and authorization |
| `job` | Asynchronous job queue |
| `worker` | Worker pool management |
| `sync` | State synchronization |
| `network` | Multi-party networking |
| `notification` | User notifications |
| `blob_storage` | Large data storage |
| `metadata` | Metadata management |
| `settings` | Server configuration |
| `log` | Audit logging |

### Storage Backends

**Document Store:**
- SQLite (default, single-node)
- PostgreSQL (production, scalable)
- MongoDB (experimental)

**Blob Store:**
- On-disk (default)
- SeaweedFS (distributed)
- Azure Blob Storage
- AWS S3

---

## Usage Patterns

### Pattern 1: Simple Remote Execution

**Scenario:** Data scientist wants to compute statistics on private data.

```python
# Data scientist
import syft as sy

client = sy.login(url="localhost", port=8080, email="ds@example.com", password="pwd")

# Get dataset
dataset = client.datasets["medical_records"]
patient_data = dataset.assets["patients"].mock

# Write function using mock data
@sy.syft_function()
def compute_average_age(data):
    import pandas as pd
    return data["age"].mean()

# Test locally
print(compute_average_age(data=patient_data))

# Request execution on real data
request = client.code.request_code_execution(compute_average_age)

# Wait for approval...
result = request.accept_by_depositing_result()
print(f"Real average age: {result}")
```

### Pattern 2: Custom Policies

**Scenario:** Data owner wants fine-grained control over outputs.

```python
# Data owner
import syft as sy

# Define custom output policy
@sy.custom_output_policy
def check_aggregated(result):
    """Only allow aggregated statistics, not individual records"""
    if isinstance(result, (int, float)):
        return True
    if isinstance(result, dict):
        return all(isinstance(v, (int, float)) for v in result.values())
    return False

# Apply policy to dataset
client.datasets["medical_records"].set_output_policy(check_aggregated)

# Review and approve request with policy
request = client.requests[0]
if check_aggregated(request.code.mock_output):
    request.approve()
else:
    request.deny(reason="Output must be aggregated")
```

### Pattern 3: Multi-Party Computation

**Scenario:** Aggregate data across multiple hospitals without sharing raw data.

```python
# Data scientist connects to gateway
gateway = sy.login(url="network.example.com", port=8080, ...)

# See connected datasites
datasites = gateway.datasites

# Submit code to run across all sites
@sy.syft_function()
def count_patients(data):
    return len(data)

results = []
for site in datasites:
    client = gateway.connect(site)
    request = client.code.request_code_execution(count_patients)
    results.append(request)

# After approvals, aggregate results
total_patients = sum(r.accept_by_depositing_result() for r in results)
print(f"Total patients across network: {total_patients}")
```

### Pattern 4: Custom Workers

**Scenario:** Run code in isolated Docker containers for security.

```python
# Data owner configures custom worker pool
worker_config = sy.DockerWorkerConfig(
    image="python:3.10-slim",
    cpu_limit=2,
    memory_limit="4g",
    network_mode="none"  # No network access
)

server = sy.orchestra.launch(
    name="secure-datasite",
    server_type="datasite",
    worker_config=worker_config
)

# Now all user code runs in isolated containers
```

### Pattern 5: Differential Privacy

**Scenario:** Ensure outputs have formal privacy guarantees.

```python
@sy.syft_function()
def private_mean(data):
    import opendp.prelude as dp

    # Create DP mechanism
    mean_mechanism = (
        dp.space_of(dp.vector_domain(dp.atom_domain(T=float)), dp.symmetric_distance()) >>
        dp.t.then_clamp(bounds=(0.0, 100.0)) >>
        dp.t.then_resize(size=1000, constant=50.0) >>
        dp.t.then_mean() >>
        dp.m.then_laplace(scale=1.0)
    )

    return mean_mechanism(list(data))

# Request execution with DP guarantee
request = client.code.request_code_execution(private_mean)
```

---

## API Reference

### Server Management (Orchestra API)

#### `sy.orchestra.launch()`

Launch a server locally.

```python
server = sy.orchestra.launch(
    name: str,              # Server name
    port: int,              # Port number
    server_type: str,       # "datasite", "gateway", or "enclave"
    dev_mode: bool = True,  # Development mode
    reset: bool = False,    # Reset database
    tail: bool = False,     # Show logs
    processes: int = 1,     # Number of processes
    local_db: bool = True,  # Use local SQLite
)
```

**Returns:** `ServerHandle` object

### Client Connection

#### `sy.login()`

Login to a server.

```python
client = sy.login(
    url: str,           # Server URL or IP
    port: int,          # Port number
    email: str,         # User email
    password: str,      # Password
)
```

**Returns:** `DatasiteClient`, `GatewayClient`, or `EnclaveClient`

#### `sy.login_as_guest()`

Login as anonymous guest (if enabled).

```python
client = sy.login_as_guest(
    url: str,
    port: int,
)
```

#### `sy.register()`

Register a new account.

```python
client = sy.register(
    url: str,
    port: int,
    name: str,
    email: str,
    password: str,
)
```

### Client API

#### Dataset Operations

```python
# List datasets
datasets = client.datasets

# Get specific dataset
dataset = client.datasets["dataset_name"]
# or
dataset = client.datasets[0]

# Dataset attributes
dataset.name
dataset.description
dataset.assets
dataset.contributors
dataset.updated_at

# Get asset
asset = dataset.assets["asset_name"]

# Access data
mock_data = asset.mock        # Mock data (always accessible)
private_data = asset.data     # Private data (owner only)
```

#### Code Submission

```python
# Define function
@sy.syft_function()
def my_function(data, param1=10):
    # Your code here
    return result

# Request execution
request = client.code.request_code_execution(
    my_function,
    reason="Need to compute X for research Y"
)

# Check request
request.status              # "pending", "approved", "denied"
request.code               # Submitted code
request.approve()          # (Owner) Approve request
request.deny(reason="...")  # (Owner) Deny request

# Get result (after approval)
result = request.accept_by_depositing_result()
```

#### Request Management

```python
# List all requests (owner)
requests = client.requests

# Filter requests
pending = client.requests.pending
approved = client.requests.approved
denied = client.requests.denied

# Get specific request
request = client.requests[0]
```

#### User Management (Admin)

```python
# List users
users = client.users

# Create user
client.users.create(
    name="John Doe",
    email="john@example.com",
    password="secure_password",
    role="data_scientist"  # or "data_owner", "admin"
)

# Update user
user = client.users[0]
user.update(role="admin")

# Delete user
user.delete()
```

### Decorators

#### `@sy.syft_function()`

Mark a function for remote execution.

```python
@sy.syft_function(
    input_policy=None,      # Optional input policy
    output_policy=None,     # Optional output policy
    share_results=False,    # Share results with other users
)
def my_function(data):
    return result
```

#### `@sy.api_endpoint()`

Create a custom API endpoint.

```python
@sy.api_endpoint(
    path="/custom/endpoint",
    method="GET"
)
def custom_handler(request):
    return {"status": "success"}
```

---

## Deployment Options

### Option 1: Python Process (Development)

Simplest deployment for testing.

```python
import syft as sy

server = sy.orchestra.launch(
    name="dev-server",
    port=8080,
    dev_mode=True
)

# Server runs in background
# Use Ctrl+C to stop
```

### Option 2: Docker (Single Node)

Production-ready single server.

```bash
cd packages/grid
docker-compose up -d

# Or with custom configuration
docker run -d \
  -p 8080:8080 \
  -v syft-data:/data \
  -e DEFAULT_ROOT_EMAIL=admin@example.com \
  -e DEFAULT_ROOT_PASSWORD=secret \
  openmined/grid-backend:latest
```

Configuration via environment variables:
- `DEFAULT_ROOT_EMAIL` - Admin email
- `DEFAULT_ROOT_PASSWORD` - Admin password
- `MONGO_HOST` - MongoDB host (if using)
- `POSTGRES_HOST` - PostgreSQL host (if using)
- `SEAWEEDFS_HOST` - SeaweedFS host (if using)

### Option 3: Kubernetes (Production)

Scalable deployment with Helm.

```bash
# Add Helm repository
helm repo add openmined https://openmined.github.io/PySyft/helm

# Install
helm install my-datasite openmined/syft \
  --set ingress.enabled=true \
  --set ingress.host=datasite.example.com \
  --set database.type=postgresql \
  --set blobStorage.type=seaweedfs

# Upgrade
helm upgrade my-datasite openmined/syft

# Uninstall
helm uninstall my-datasite
```

### Option 4: Kubernetes (Advanced)

Custom configuration with values file.

```yaml
# values.yaml
server:
  type: datasite
  replicas: 3

database:
  type: postgresql
  host: postgres.default.svc.cluster.local
  port: 5432

blobStorage:
  type: seaweedfs
  host: seaweedfs.default.svc.cluster.local

ingress:
  enabled: true
  host: datasite.example.com
  tls:
    enabled: true
    secretName: tls-secret

resources:
  limits:
    cpu: 2
    memory: 4Gi
  requests:
    cpu: 1
    memory: 2Gi

workers:
  enabled: true
  replicas: 5
  type: kubernetes
```

```bash
helm install my-datasite openmined/syft -f values.yaml
```

### Option 5: Secure Enclave

Hardware-based secure computation.

```bash
# Build enclave image
cd packages/grid/enclave
docker build -t my-enclave .

# Launch enclave
docker run -d \
  --device=/dev/sgx \
  -p 8080:8080 \
  my-enclave
```

---

## Security Model

### Authentication & Authorization

**Authentication:**
- Email/password credentials
- Argon2 password hashing
- Session-based authentication
- Optional guest access

**Authorization:**
- Role-Based Access Control (RBAC)
- Roles: Admin, Data Owner, Data Scientist, Guest
- Granular permissions per service
- Request-based access control

### Data Isolation

**Twin Objects:**
- Private data never leaves the server
- Mock data shared for development
- Automatic redirection of operations

**Action System:**
- All operations tracked
- Serializable action objects
- Audit trail for compliance
- Replay and undo capabilities

### Code Sandboxing

**RestrictedPython:**
- Limited Python subset
- No file system access
- No network access
- No subprocess execution

**Custom Workers:**
- Isolated Docker containers
- Resource limits (CPU, memory)
- Network isolation
- Ephemeral execution environment

### Policy Enforcement

**Input Policies:**
- Validate input parameters
- Restrict data access
- Type checking
- Custom validation logic

**Output Policies:**
- Validate outputs before release
- Aggregation requirements
- Differential privacy enforcement
- Custom release conditions

### Network Security

**Transport:**
- HTTPS/TLS encryption
- WebSocket secure connections
- Certificate validation

**Infrastructure:**
- Firewall rules
- VPC isolation
- Secret management
- Key rotation

---

## Use Cases

### 1. Healthcare Research

**Scenario:** Researchers need to analyze patient data across multiple hospitals.

**Challenge:**
- Patient data is protected by HIPAA
- Hospitals cannot share raw data
- Need statistical power from large datasets

**Solution:**
- Each hospital runs a PySyft datasite
- Researchers connect to a gateway
- Code runs locally at each hospital
- Only aggregated results are shared

**Example:**
```python
@sy.syft_function()
def survival_analysis(patient_data, treatment_data):
    import lifelines
    from lifelines import KaplanMeierFitter

    kmf = KaplanMeierFitter()
    kmf.fit(patient_data['duration'], patient_data['observed'])

    return {
        'median_survival': kmf.median_survival_time_,
        'confidence_interval': kmf.confidence_interval_.to_dict()
    }
```

### 2. Financial Fraud Detection

**Scenario:** Banks want to collaborate on fraud detection without sharing transaction data.

**Challenge:**
- Transaction data is proprietary
- Regulatory restrictions
- Competitive concerns

**Solution:**
- Banks run datasites with transaction data
- Fraud detection models trained federally
- Each bank benefits from collective intelligence

**Example:**
```python
@sy.syft_function()
def detect_anomalies(transactions):
    from sklearn.ensemble import IsolationForest

    model = IsolationForest(contamination=0.01)
    scores = model.fit_predict(transactions)

    # Return only aggregated statistics
    return {
        'num_anomalies': (scores == -1).sum(),
        'percentage': (scores == -1).mean()
    }
```

### 3. Government Census Analysis

**Scenario:** Policy researchers need to analyze census data.

**Challenge:**
- Census data contains PII
- Privacy concerns
- Legal restrictions on data access

**Solution:**
- Government agency hosts datasite with census data
- Researchers submit analysis code
- Agency reviews and approves
- Results released with differential privacy

**Example:**
```python
@sy.syft_function()
def analyze_demographics(census_data):
    import opendp.prelude as dp

    # Apply differential privacy
    dp_mean = dp.m.make_laplace(
        dp.atom_domain(T=float),
        dp.absolute_distance(T=float),
        scale=1.0
    )

    return {
        'avg_income': dp_mean(census_data['income'].mean()),
        'avg_age': dp_mean(census_data['age'].mean())
    }
```

### 4. AI Model Validation

**Scenario:** Company wants third-party validation of AI model fairness.

**Challenge:**
- Model is proprietary
- Training data is sensitive
- Need independent audit

**Solution:**
- Company runs datasite with model and test data
- Auditor submits fairness analysis code
- Company reviews and approves
- Auditor receives fairness metrics

**Example:**
```python
@sy.syft_function()
def fairness_audit(model, test_data, protected_attribute):
    from fairlearn.metrics import demographic_parity_difference

    predictions = model.predict(test_data)

    dpd = demographic_parity_difference(
        test_data[protected_attribute],
        predictions
    )

    return {
        'demographic_parity_difference': dpd,
        'passes_threshold': abs(dpd) < 0.1
    }
```

### 5. Academic Research Collaboration

**Scenario:** Universities want to collaborate on social science research.

**Challenge:**
- Each university has sensitive participant data
- IRB restrictions on data sharing
- Need combined statistical power

**Solution:**
- Each university runs a datasite
- Central gateway for coordination
- Researchers run analyses across sites
- Individual data never shared

---

## Best Practices

### For Data Scientists

1. **Always test with mock data first**
   ```python
   # Test locally before requesting
   result = my_function(data=mock_data)
   assert result is not None
   ```

2. **Write clear request justifications**
   ```python
   request = client.code.request_code_execution(
       my_function,
       reason="Computing X for publication in Journal Y, IRB #12345"
   )
   ```

3. **Handle errors gracefully**
   ```python
   @sy.syft_function()
   def safe_analysis(data):
       try:
           return compute_result(data)
       except Exception as e:
           return {"error": str(e)}
   ```

4. **Respect data owner policies**
   - Don't try to extract individual records
   - Return aggregated results only
   - Follow specified data usage terms

5. **Minimize dependencies**
   ```python
   # Import inside function
   @sy.syft_function()
   def my_function(data):
       import pandas as pd  # Import here
       return pd.DataFrame(data).mean()
   ```

### For Data Owners

1. **Provide high-quality mock data**
   ```python
   # Mock data should match schema and statistics
   mock_data = generate_synthetic_data(
       schema=private_data.dtypes,
       size=len(private_data),
       preserve_distributions=True
   )
   ```

2. **Define clear output policies**
   ```python
   @sy.custom_output_policy
   def aggregation_only(result):
       # Only allow aggregated statistics
       return isinstance(result, (int, float, dict))
   ```

3. **Review code carefully**
   - Check for data extraction attempts
   - Verify computational complexity
   - Test with mock data first
   - Consider privacy implications

4. **Use custom workers for isolation**
   ```python
   worker_config = sy.DockerWorkerConfig(
       image="python:3.10-slim",
       network_mode="none",
       memory_limit="2g"
   )
   ```

5. **Enable audit logging**
   ```python
   client.settings.audit_log = True
   client.settings.log_level = "INFO"
   ```

### For Deployment

1. **Use production database**
   ```python
   # Don't use SQLite in production
   server = sy.orchestra.launch(
       name="prod-server",
       local_db=False,  # Use PostgreSQL
   )
   ```

2. **Enable TLS/HTTPS**
   ```bash
   # Configure ingress with TLS
   helm install datasite openmined/syft \
     --set ingress.tls.enabled=true
   ```

3. **Set up backups**
   ```bash
   # Backup database and blob storage regularly
   pg_dump syft_db > backup.sql
   aws s3 sync /data/blobs s3://backups/blobs
   ```

4. **Monitor resource usage**
   ```python
   # Set resource limits
   worker_config = sy.DockerWorkerConfig(
       cpu_limit=2,
       memory_limit="4g"
   )
   ```

5. **Implement access controls**
   ```python
   # Disable guest access in production
   client.settings.allow_guest_access = False
   ```

---

## Troubleshooting

### Common Issues

#### 1. Connection Failed

**Symptom:** `ConnectionError: Failed to connect to server`

**Solutions:**
- Check server is running: `docker ps` or check Python process
- Verify port is correct
- Check firewall rules
- Ensure URL format is correct (no http://)

```python
# Correct
client = sy.login(url="localhost", port=8080, ...)

# Incorrect
client = sy.login(url="http://localhost:8080", ...)
```

#### 2. Authentication Error

**Symptom:** `AuthenticationError: Invalid credentials`

**Solutions:**
- Verify email and password
- Check if account exists
- Reset password if needed
- For default admin: `info@openmined.org` / `changethis`

#### 3. Request Stuck in Pending

**Symptom:** Request status remains "pending"

**Solutions:**
- Data owner needs to review and approve
- Check notifications: `client.notifications`
- Contact data owner
- Check request status: `request.status`

#### 4. Import Error in Syft Function

**Symptom:** `ImportError: No module named 'X'`

**Solutions:**
- Import modules inside function, not at top level
- Ensure module is available in worker environment
- Use custom worker with required dependencies

```python
# Correct
@sy.syft_function()
def my_function(data):
    import pandas as pd  # Import inside
    return pd.DataFrame(data).mean()

# Incorrect
import pandas as pd  # Import outside

@sy.syft_function()
def my_function(data):
    return pd.DataFrame(data).mean()
```

#### 5. Memory Error

**Symptom:** `MemoryError: Out of memory`

**Solutions:**
- Process data in chunks
- Reduce batch size
- Increase worker memory limit
- Use more efficient algorithms

```python
@sy.syft_function()
def process_large_data(data):
    # Process in chunks
    chunk_size = 1000
    results = []
    for i in range(0, len(data), chunk_size):
        chunk = data[i:i+chunk_size]
        results.append(process_chunk(chunk))
    return aggregate_results(results)
```

#### 6. Serialization Error

**Symptom:** `SerializationError: Cannot serialize object`

**Solutions:**
- Return simple types (int, float, str, dict, list)
- Convert complex objects to dictionaries
- Avoid returning large objects

```python
# Correct
@sy.syft_function()
def my_function(data):
    model = train_model(data)
    return {"accuracy": model.score(), "params": model.get_params()}

# Incorrect
@sy.syft_function()
def my_function(data):
    model = train_model(data)
    return model  # Can't serialize model object
```

### Debugging Tips

1. **Enable debug logging**
   ```python
   import logging
   logging.basicConfig(level=logging.DEBUG)
   ```

2. **Check server logs**
   ```bash
   docker logs syft-backend
   # or
   tail -f /var/log/syft/server.log
   ```

3. **Test mock data first**
   ```python
   # Always test locally
   result = my_function(data=asset.mock)
   print(result)
   ```

4. **Use interactive debugger**
   ```python
   @sy.syft_function()
   def debug_function(data):
       import pdb; pdb.set_trace()  # Set breakpoint
       return result
   ```

5. **Check service status**
   ```python
   client.api.services  # List available services
   client.api.services.dataset.status  # Check specific service
   ```

### Getting Help

- **Documentation:** https://docs.openmined.org
- **GitHub Issues:** https://github.com/OpenMined/PySyft/issues
- **Slack Community:** https://slack.openmined.org
- **Stack Overflow:** Tag with `pysyft`

---

## Additional Resources

### Documentation
- Official Docs: https://docs.openmined.org
- API Reference: https://docs.openmined.org/api
- Tutorials: https://github.com/OpenMined/PySyft/tree/main/notebooks

### Example Notebooks
- Getting Started: `/notebooks/scenarios/getting-started/`
- Data Owner Guide: `/notebooks/tutorials/data-owner/`
- Data Scientist Guide: `/notebooks/tutorials/data-scientist/`
- Deployments: `/notebooks/tutorials/deployments/`

### Community
- GitHub: https://github.com/OpenMined/PySyft
- Blog: https://blog.openmined.org
- Twitter: @OpenMinedOrg
- YouTube: OpenMined

### Research Papers
- "PySyft: A Library for Privacy-Preserving Machine Learning"
- "Remote Data Science: Bringing Computation to Data"
- "Structured Transparency: Building Trust in Data Science"

---

## License

PySyft is licensed under Apache License 2.0

---

## Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Development Setup
```bash
git clone https://github.com/OpenMined/PySyft.git
cd PySyft
pip install -e "packages/syft[dev]"
pytest tests/
```

---

**Last Updated:** 2025-11-07
**Version:** 0.9.6-beta.6
