# PySyft Skill

## Description

Expert assistant for PySyft - a privacy-preserving data science platform that enables secure computation on sensitive data without requiring direct access.

## When to Use This Skill

Use this skill when:
- Working with PySyft code or concepts
- Setting up datasites, gateways, or enclaves
- Creating remote execution workflows
- Implementing privacy policies
- Deploying PySyft in development or production
- Debugging PySyft applications
- Writing syft_functions
- Managing datasets and assets
- Handling requests and approvals

## Core Concepts

### 1. PySyft Architecture

**Three Server Types:**
- **Datasite**: Hosts datasets and executes approved code
- **Gateway**: Network hub connecting multiple datasites
- **Enclave**: Secure computation with hardware attestation

**Key Components:**
- **Twin Objects**: Combine private data (real) and mock data (synthetic)
- **Services**: 20+ microservices (dataset, code, request, policy, etc.)
- **Stores**: Document store (SQLite/PostgreSQL/MongoDB) + Blob store (disk/SeaweedFS/S3/Azure)
- **Workers**: Execution environments (Python/Docker/Kubernetes)

### 2. Core Workflow Pattern

```
Data Scientist → Develop with Mock → Submit Request → Data Owner Reviews → Approve → Execute on Private → Return Results
```

### 3. Security Model

- **Authentication**: Argon2 password hashing, session tokens
- **Authorization**: RBAC (Admin, DataOwner, DataScientist, Guest)
- **Code Sandboxing**: RestrictedPython
- **Worker Isolation**: Docker/K8s containers
- **Policy Enforcement**: Input and output policies

## Quick Reference

### Installation

```bash
# Basic installation
pip install -U syft

# With data science dependencies
pip install -U "syft[data_science]"

# Development installation
git clone https://github.com/OpenMined/PySyft.git
cd PySyft
pip install -e "packages/syft[dev]"
```

### Launch a Server

```python
import syft as sy

# Launch datasite
server = sy.orchestra.launch(
    name="my-datasite",
    port=8080,
    server_type="datasite",  # or "gateway", "enclave"
    dev_mode=True,
    reset=True
)
```

### Client Connection

```python
# Login
client = sy.login(
    url="localhost",
    port=8080,
    email="user@example.com",
    password="password"
)

# Login as guest
client = sy.login_as_guest(url="localhost", port=8080)

# Register new account
client = sy.register(
    url="localhost",
    port=8080,
    name="John Doe",
    email="john@example.com",
    password="secure_password"
)
```

### Data Owner: Upload Dataset

```python
import syft as sy
import pandas as pd

# Create dataset
dataset = sy.Dataset(
    name="My Dataset",
    description="Description of the dataset"
)

# Prepare data
private_data = pd.DataFrame({
    "age": [25, 30, 35, 40, 45],
    "income": [50000, 60000, 70000, 80000, 90000]
})

mock_data = pd.DataFrame({
    "age": [28, 32, 38, 42, 47],
    "income": [52000, 62000, 72000, 82000, 92000]
})

# Add asset with twin objects
dataset.add_asset(
    name="salary_data",
    data=private_data,  # Real data
    mock=mock_data      # Synthetic data
)

# Upload
client.upload_dataset(dataset)
```

### Data Scientist: Submit Code

```python
import syft as sy

# Connect
client = sy.login(url="localhost", port=8080, ...)

# Get dataset
dataset = client.datasets["My Dataset"]
asset = dataset.assets["salary_data"]

# Develop with mock data
mock_data = asset.mock

# Define function
@sy.syft_function()
def compute_average(data):
    import pandas as pd
    return data["income"].mean()

# Test with mock
result = compute_average(data=mock_data)
print(f"Mock result: {result}")

# Submit request
request = client.code.request_code_execution(
    compute_average,
    reason="Computing average income for research paper"
)

# Check status
print(request.status)  # "pending"

# After approval, get result
result = request.accept_by_depositing_result()
print(f"Real result: {result}")
```

### Data Owner: Review Requests

```python
# Get pending requests
requests = client.requests.pending

# Review first request
request = requests[0]

# Check the code
print(request.code)

# Approve
request.approve()

# Or deny with reason
# request.deny(reason="Insufficient privacy guarantees")
```

## Common Patterns

### Pattern 1: Simple Remote Execution

```python
# Data scientist workflow
@sy.syft_function()
def analyze(data):
    import numpy as np
    return {
        "mean": np.mean(data),
        "std": np.std(data),
        "count": len(data)
    }

request = client.code.request_code_execution(analyze)
```

### Pattern 2: Custom Input Policy

```python
from syft.service.policy.policy import ExactMatch

# Require specific assets
policy = ExactMatch(
    required_assets=["dataset1", "dataset2"]
)

@sy.syft_function(input_policy=policy)
def analyze(data1, data2):
    return combine_and_analyze(data1, data2)
```

### Pattern 3: Custom Output Policy

```python
@sy.custom_output_policy
def aggregated_only(result):
    """Only allow aggregated statistics, not individual records"""
    if isinstance(result, (int, float)):
        return True
    if isinstance(result, dict):
        return all(isinstance(v, (int, float)) for v in result.values())
    return False

@sy.syft_function(output_policy=aggregated_only)
def compute_stats(data):
    return {
        "mean": data.mean(),
        "median": data.median()
    }
```

### Pattern 4: Differential Privacy

```python
@sy.syft_function()
def private_mean(data):
    import opendp.prelude as dp

    # Create DP mechanism
    mean_mechanism = (
        dp.space_of(dp.vector_domain(dp.atom_domain(T=float)),
                    dp.symmetric_distance()) >>
        dp.t.then_clamp(bounds=(0.0, 100.0)) >>
        dp.t.then_resize(size=1000, constant=50.0) >>
        dp.t.then_mean() >>
        dp.m.then_laplace(scale=1.0)
    )

    return mean_mechanism(list(data))
```

### Pattern 5: Multi-Party Computation

```python
# Connect to gateway
gateway = sy.login(url="network.example.com", port=8080, ...)

# Get connected datasites
datasites = gateway.datasites

# Submit code to multiple sites
@sy.syft_function()
def count_records(data):
    return len(data)

results = []
for site in datasites:
    client = gateway.connect(site)
    request = client.code.request_code_execution(count_records)
    results.append(request)

# After approvals
total = sum(r.accept_by_depositing_result() for r in results)
```

## Deployment Patterns

### Development (Python)

```python
import syft as sy

server = sy.orchestra.launch(
    name="dev-server",
    port=8080,
    dev_mode=True
)
```

### Production (Docker)

```bash
cd packages/grid
docker-compose up -d
```

### Production (Kubernetes)

```bash
# Install with Helm
helm repo add openmined https://openmined.github.io/PySyft/helm
helm install my-datasite openmined/syft
```

## Troubleshooting Guide

### Issue: Connection Failed

```python
# Check server is running
docker ps  # or check Python process

# Verify URL format (no http://)
client = sy.login(url="localhost", port=8080, ...)  # Correct
# NOT: url="http://localhost:8080"
```

### Issue: Import Error in syft_function

```python
# WRONG: Import outside function
import pandas as pd

@sy.syft_function()
def analyze(data):
    return pd.DataFrame(data).mean()

# CORRECT: Import inside function
@sy.syft_function()
def analyze(data):
    import pandas as pd  # Import here
    return pd.DataFrame(data).mean()
```

### Issue: Cannot Serialize Return Value

```python
# WRONG: Return complex objects
@sy.syft_function()
def train_model(data):
    model = TrainModel(data)
    return model  # Can't serialize

# CORRECT: Return simple types
@sy.syft_function()
def train_model(data):
    model = TrainModel(data)
    return {
        "accuracy": model.score(),
        "params": model.get_params()
    }
```

## Best Practices

### For Data Scientists

1. **Always test with mock data first**
   ```python
   result = my_function(data=asset.mock)
   assert result is not None
   ```

2. **Write clear justifications**
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

4. **Return aggregated results only**
   ```python
   # Good: Aggregated
   return {"mean": data.mean(), "count": len(data)}

   # Bad: Individual records
   return data.to_dict()
   ```

### For Data Owners

1. **Provide high-quality mock data**
   ```python
   # Mock should match schema and statistics
   mock_data = generate_synthetic(
       schema=private_data.dtypes,
       preserve_distributions=True
   )
   ```

2. **Define clear output policies**
   ```python
   @sy.custom_output_policy
   def no_individual_records(result):
       return isinstance(result, (int, float, dict))
   ```

3. **Review code carefully**
   - Check for data extraction attempts
   - Verify computational complexity
   - Test with mock data first
   - Consider privacy implications

4. **Use worker isolation**
   ```python
   worker_config = sy.DockerWorkerConfig(
       image="python:3.10-slim",
       network_mode="none",
       memory_limit="2g"
   )
   ```

## Code Examples

### Example 1: Healthcare Research

```python
@sy.syft_function()
def survival_analysis(patient_data):
    import pandas as pd
    from lifelines import KaplanMeierFitter

    kmf = KaplanMeierFitter()
    kmf.fit(
        patient_data['duration'],
        patient_data['observed']
    )

    return {
        'median_survival': kmf.median_survival_time_,
        'ci_lower': kmf.confidence_interval_.iloc[0, 0],
        'ci_upper': kmf.confidence_interval_.iloc[0, 1]
    }
```

### Example 2: Financial Fraud Detection

```python
@sy.syft_function()
def detect_anomalies(transactions):
    from sklearn.ensemble import IsolationForest

    model = IsolationForest(contamination=0.01)
    scores = model.fit_predict(transactions)

    return {
        'num_anomalies': (scores == -1).sum(),
        'percentage': (scores == -1).mean() * 100
    }
```

### Example 3: Model Training with DP

```python
@sy.syft_function()
def train_with_dp(features, labels, epsilon=1.0):
    import torch
    import opacus

    model = create_model()
    optimizer = torch.optim.SGD(model.parameters(), lr=0.01)

    privacy_engine = opacus.PrivacyEngine()
    model, optimizer, data_loader = privacy_engine.make_private(
        module=model,
        optimizer=optimizer,
        data_loader=create_loader(features, labels),
        noise_multiplier=1.1,
        max_grad_norm=1.0,
    )

    # Training loop
    for epoch in range(10):
        for batch in data_loader:
            loss = train_step(model, batch)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

    return {
        'epsilon': privacy_engine.get_epsilon(delta=1e-5),
        'accuracy': evaluate(model)
    }
```

## API Quick Reference

### Server Management

```python
sy.orchestra.launch(name, port, server_type, dev_mode, reset)
```

### Client Operations

```python
sy.login(url, port, email, password)
sy.login_as_guest(url, port)
sy.register(url, port, name, email, password)
```

### Dataset Operations

```python
client.datasets                    # List all datasets
client.datasets["name"]           # Get dataset by name
dataset.assets["name"]            # Get asset
asset.mock                        # Mock data
asset.data                        # Private data (owner only)
```

### Code Operations

```python
@sy.syft_function()               # Decorator for remote code
client.code.request_code_execution(func, reason)
request.status                    # Check status
request.approve()                 # Approve (owner)
request.deny(reason)              # Deny (owner)
request.accept_by_depositing_result()  # Get result
```

### User Management

```python
client.users                      # List users
client.users.create(name, email, password, role)
user.update(role="admin")
user.delete()
```

## Helper Functions

When helping users with PySyft, use these guidelines:

1. **Identify the use case**: Data scientist or data owner?
2. **Check server setup**: Development or production?
3. **Verify code patterns**: Imports inside functions, simple returns
4. **Consider privacy**: Appropriate policies and aggregation
5. **Test incrementally**: Mock data first, then request

## Common User Questions

**Q: How do I install PySyft?**
A: `pip install -U syft` for basic, or `pip install -U "syft[data_science]"` for full

**Q: How do I start a server?**
A: `sy.orchestra.launch(name="server", port=8080, dev_mode=True)`

**Q: How do I upload data?**
A: Create a Dataset, add assets with `dataset.add_asset(name, data, mock)`, then `client.upload_dataset(dataset)`

**Q: How do I run code on private data?**
A: Write a `@sy.syft_function()`, test with mock, submit with `client.code.request_code_execution(func)`

**Q: How do I approve requests?**
A: `client.requests.pending`, review, then `request.approve()` or `request.deny(reason)`

**Q: Why can't I import libraries?**
A: Import inside the function, not at the top level

**Q: How do I add privacy guarantees?**
A: Use OpenDP for differential privacy, or define custom output policies

**Q: Can I use PyTorch/TensorFlow?**
A: Yes, PySyft is framework-agnostic

## Resources

- **Documentation**: https://docs.openmined.org
- **GitHub**: https://github.com/OpenMined/PySyft
- **Examples**: `/notebooks` directory in the repo
- **Community**: https://slack.openmined.org

## Version Info

**Current Version**: 0.9.6-beta.6
**Python Requirements**: >=3.10
**Status**: Beta (API may change)

---

When using this skill, always:
1. Check the user's PySyft version
2. Verify server type (datasite/gateway/enclave)
3. Consider privacy implications
4. Test with mock data first
5. Follow best practices
6. Provide complete, working examples
