# 🛡️ Pydantic — The Complete DevOps Engineer's Guide
### Data Validation, Settings Management & Type Safety from Basics to Advanced

> **Who this is for:** Junior DevOps engineers learning Python data validation, and Senior DevOps engineers building production-grade configs, API contracts, infrastructure models, and AI pipelines where bad data can mean a broken deployment or a downed cluster.

---

## Table of Contents

1. [Why Pydantic Matters in DevOps](#1-why-pydantic-matters-in-devops)
2. [Pydantic v1 vs v2 — What Changed](#2-pydantic-v1-vs-v2--what-changed)
3. [Installation & Setup](#3-installation--setup)
4. [BaseModel — The Foundation](#4-basemodel--the-foundation)
5. [Field Types & Type Hints](#5-field-types--type-hints)
6. [Field() — Constraints & Metadata](#6-field--constraints--metadata)
7. [Validators — Custom Validation Logic](#7-validators--custom-validation-logic)
8. [Model Configuration with model_config](#8-model-configuration-with-model_config)
9. [Nested Models](#9-nested-models)
10. [Serialization — .model_dump() & .model_dump_json()](#10-serialization--model_dump--model_dump_json)
11. [Deserialization — Parsing & Loading Data](#11-deserialization--parsing--loading-data)
12. [Computed Fields](#12-computed-fields)
13. [Model Inheritance](#13-model-inheritance)
14. [Generic Models](#14-generic-models)
15. [Discriminated Unions](#15-discriminated-unions)
16. [Pydantic Settings — Environment & Config Management](#16-pydantic-settings--environment--config-management)
17. [Custom Types & Annotated Types](#17-custom-types--annotated-types)
18. [Strict Mode & Type Coercion](#18-strict-mode--type-coercion)
19. [Handling Errors & ValidationError](#19-handling-errors--validationerror)
20. [Pydantic with FastAPI](#20-pydantic-with-fastapi)
21. [Pydantic with SQLAlchemy & ORMs](#21-pydantic-with-sqlalchemy--orms)
22. [Pydantic for Infrastructure & Kubernetes Models](#22-pydantic-for-infrastructure--kubernetes-models)
23. [Pydantic in AI/ML DevOps Pipelines](#23-pydantic-in-aiml-devops-pipelines)
24. [Performance Tips & Advanced Patterns](#24-performance-tips--advanced-patterns)
25. [Quick Reference Cheatsheet](#25-quick-reference-cheatsheet)

---

## 1. Why Pydantic Matters in DevOps

In DevOps, data flows everywhere — Terraform variable files, Kubernetes manifests, Helm values, API payloads, CI/CD pipeline configs, environment variables, webhook events. **One malformed value can:**

- Crash a deployment pipeline
- Misconfigure a Kubernetes resource
- Expose a secret in plain text
- Send garbage to an AI model
- Bring down a production service

Pydantic is Python's **runtime data validation and settings management library**. It catches bad data at the boundary — before it ever touches your business logic.

```
Raw Input (JSON / ENV / YAML / API)
            ↓
     [Pydantic Model]          ← Validates, coerces, rejects
            ↓
  Clean, Typed Python Object   ← Safe to use
```

### Real DevOps Use Cases

| Use Case | Without Pydantic | With Pydantic |
|---|---|---|
| Reading `.env` files | Manual `os.getenv()`, no type safety | `BaseSettings` — typed, validated, documented |
| Parsing Kubernetes events | Raw dicts, `KeyError` at runtime | Structured model, validated at parse time |
| FastAPI request bodies | Manual validation, lots of `if` checks | Automatic validation, 422 on bad input |
| Terraform variable files | JSON loaded as raw dict | Typed model with constraints |
| AI model inputs | Hope the caller sends the right types | `ValidationError` immediately if wrong |
| Webhook payloads | `payload.get("event", {}).get("type")` | Clean nested model access |

---

## 2. Pydantic v1 vs v2 — What Changed

Pydantic v2 (released 2023) was a ground-up rewrite in **Rust**. It is **5–50x faster** than v1. If you're starting fresh, always use v2.

| Feature | Pydantic v1 | Pydantic v2 |
|---|---|---|
| Performance | Python | Rust core (pydantic-core) |
| Validator syntax | `@validator` | `@field_validator`, `@model_validator` |
| Config class | `class Config:` | `model_config = ConfigDict(...)` |
| `.dict()` | ✅ | ❌ Deprecated → use `.model_dump()` |
| `.json()` | ✅ | ❌ Deprecated → use `.model_dump_json()` |
| `.parse_obj()` | ✅ | ❌ Deprecated → use `Model.model_validate()` |
| `schema()` | ✅ | ❌ Deprecated → use `Model.model_json_schema()` |
| Arbitrary types | `arbitrary_types_allowed = True` | `arbitrary_types_allowed = True` in ConfigDict |
| Speed | Baseline | 5-50x faster |

> 🚨 **If you see `@validator` or `class Config:` in a codebase — that's v1 syntax.** This guide uses v2 exclusively.

---

## 3. Installation & Setup

```bash
# Install Pydantic v2
pip install pydantic

# With email validation support
pip install "pydantic[email]"

# Settings management (separate package in v2)
pip install pydantic-settings

# Verify version
python -c "import pydantic; print(pydantic.__version__)"
# Should print 2.x.x
```

---

## 4. BaseModel — The Foundation

Every Pydantic model starts with `BaseModel`. Think of it as a **class that knows how to validate itself.**

### Your First Model

```python
from pydantic import BaseModel

class Server(BaseModel):
    hostname: str
    ip_address: str
    port: int
    is_active: bool

# ✅ Valid instantiation — Pydantic validates on creation
server = Server(
    hostname="web-01",
    ip_address="10.0.0.1",
    port=443,
    is_active=True
)

print(server.hostname)          # web-01
print(server.port)              # 443
print(type(server.port))        # <class 'int'>
print(server)
# hostname='web-01' ip_address='10.0.0.1' port=443 is_active=True
```

### Type Coercion (Auto-Conversion)

Pydantic tries to coerce values into the declared type by default:

```python
# String "443" is automatically coerced to int 443
server = Server(
    hostname="web-01",
    ip_address="10.0.0.1",
    port="443",         # String → coerced to int ✅
    is_active="true"    # String → coerced to bool ✅
)
print(type(server.port))       # <class 'int'>
print(type(server.is_active))  # <class 'bool'>
```

### Validation Errors

```python
from pydantic import ValidationError

try:
    bad_server = Server(
        hostname="web-01",
        ip_address="10.0.0.1",
        port="not-a-number",    # ❌ Cannot coerce to int
        is_active=True
    )
except ValidationError as e:
    print(e)
    # 1 validation error for Server
    # port
    #   Input should be a valid integer, unable to parse string as an integer
    #   [type=int_parsing, input_value='not-a-number', ...]
```

### Default Values & Optional Fields

```python
from typing import Optional
from datetime import datetime

class DeploymentEvent(BaseModel):
    deployment_id: str
    service_name: str
    environment: str
    status: str = "pending"                   # Default value
    replicas: int = 1                         # Default value
    started_at: datetime = None               # Optional, defaults to None
    completed_at: Optional[datetime] = None   # Explicit Optional
    error_message: str | None = None          # Python 3.10+ union syntax

event = DeploymentEvent(
    deployment_id="dep-001",
    service_name="nginx",
    environment="production"
    # All other fields use defaults
)
print(event.status)     # pending
print(event.replicas)   # 1
```

---

## 5. Field Types & Type Hints

Pydantic supports the full Python type system plus many extras.

### Primitive Types

```python
from pydantic import BaseModel

class TypeDemo(BaseModel):
    # Primitives
    name: str
    count: int
    ratio: float
    enabled: bool
    
    # Bytes
    raw_data: bytes
    
    # None-able
    description: str | None = None
```

### Collection Types

```python
from pydantic import BaseModel

class InfraInventory(BaseModel):
    # Lists — ordered, allows duplicates
    server_names: list[str]
    port_numbers: list[int] = []
    
    # Sets — unordered, unique values only
    unique_tags: set[str] = set()
    
    # Tuples — fixed-length, typed per position
    coordinate: tuple[float, float]           # Exactly 2 floats
    rgb_color: tuple[int, int, int]           # Exactly 3 ints
    
    # Dicts
    labels: dict[str, str] = {}
    config: dict[str, int | str | bool] = {}  # Mixed value types
    
    # Nested collections
    node_groups: dict[str, list[str]] = {}    # {group_name: [node_names]}
```

### Special Pydantic Types

```python
from pydantic import BaseModel, HttpUrl, AnyUrl, EmailStr, IPvAnyAddress, SecretStr
from uuid import UUID
from datetime import datetime, date, timedelta
from pathlib import Path

class ServiceConfig(BaseModel):
    # URLs — validated format
    api_endpoint: HttpUrl
    webhook_url: AnyUrl
    
    # Email — validated format (requires pydantic[email])
    alert_email: EmailStr
    
    # IP Address — supports IPv4 and IPv6
    bind_address: IPvAnyAddress
    
    # UUID
    service_id: UUID
    
    # Secrets — hidden in logs and repr
    api_key: SecretStr
    
    # Date/Time
    created_at: datetime
    valid_until: date
    retry_interval: timedelta
    
    # File Paths
    config_file: Path
    log_directory: Path

config = ServiceConfig(
    api_endpoint="https://api.myservice.com/v1",
    webhook_url="https://hooks.slack.com/services/xxx",
    alert_email="ops@company.com",
    bind_address="0.0.0.0",
    service_id="123e4567-e89b-12d3-a456-426614174000",
    api_key="super-secret-key",  # Will be hidden as '**********'
    created_at="2024-01-15T10:30:00Z",
    valid_until="2025-12-31",
    retry_interval="PT30S",       # ISO 8601 duration
    config_file="/etc/myapp/config.yaml",
    log_directory="/var/log/myapp"
)

print(config.api_key)             # **********  (SecretStr hides it)
print(config.api_key.get_secret_value())  # super-secret-key
```

### Enums

```python
from enum import Enum
from pydantic import BaseModel

class Environment(str, Enum):
    dev = "development"
    staging = "staging"
    prod = "production"

class LogLevel(str, Enum):
    DEBUG = "DEBUG"
    INFO = "INFO"
    WARNING = "WARNING"
    ERROR = "ERROR"
    CRITICAL = "CRITICAL"

class AppConfig(BaseModel):
    environment: Environment
    log_level: LogLevel = LogLevel.INFO

config = AppConfig(environment="production", log_level="WARNING")
print(config.environment)           # Environment.prod
print(config.environment.value)     # production
print(config.environment == Environment.prod)  # True

# ❌ Invalid enum value raises ValidationError
bad = AppConfig(environment="dev-local")  # Error!
```

---

## 6. Field() — Constraints & Metadata

`Field()` is how you add **constraints, defaults, descriptions, and examples** to model fields.

```python
from pydantic import BaseModel, Field
from typing import Annotated

class DeploymentSpec(BaseModel):
    # String constraints
    service_name: str = Field(
        ...,                    # ... means REQUIRED (no default)
        min_length=2,
        max_length=63,
        pattern=r'^[a-z0-9][a-z0-9-]*[a-z0-9]$',
        description="Service name (DNS-compatible, lowercase, hyphens allowed)",
        examples=["my-service", "nginx-proxy"]
    )
    
    # Numeric constraints
    replicas: int = Field(
        default=1,
        ge=1,    # greater than or equal
        le=50,   # less than or equal
        description="Number of pod replicas"
    )
    cpu_cores: float = Field(
        default=0.5,
        gt=0,    # strictly greater than
        lt=64.0, # strictly less than
    )
    
    # String with default
    namespace: str = Field(
        default="default",
        description="Kubernetes namespace"
    )
    
    # Field with alias — the JSON key differs from the Python attribute name
    image_tag: str = Field(
        ...,
        alias="imageTag",       # JSON uses camelCase: {"imageTag": "v1.0"}
        description="Docker image tag"
    )
    
    # Deprecated field
    old_field: str | None = Field(
        default=None,
        deprecated=True,
        description="Use 'service_name' instead"
    )

# Using Annotated for cleaner reuse
PortNumber = Annotated[int, Field(ge=1, le=65535, description="TCP/UDP port")]
Percentage = Annotated[float, Field(ge=0.0, le=100.0)]

class NetworkConfig(BaseModel):
    http_port: PortNumber = 80
    https_port: PortNumber = 443
    cpu_threshold: Percentage = 80.0
```

---

## 7. Validators — Custom Validation Logic

When built-in constraints aren't enough, write your own.

### @field_validator

```python
from pydantic import BaseModel, field_validator, ValidationInfo
import re

class ContainerImage(BaseModel):
    registry: str
    repository: str
    tag: str

    @field_validator("tag")
    @classmethod
    def no_latest_in_prod(cls, v: str) -> str:
        """Prevent 'latest' tag — it's a deployment antipattern."""
        if v == "latest":
            raise ValueError(
                "Tag 'latest' is not allowed. Use a specific version like 'v1.2.3' or a git SHA."
            )
        return v

    @field_validator("repository")
    @classmethod
    def valid_repository_path(cls, v: str) -> str:
        """Ensure repository follows org/name pattern."""
        if not re.match(r'^[a-z0-9_-]+/[a-z0-9_-]+$', v):
            raise ValueError(f"Repository '{v}' must follow 'org/name' format.")
        return v.lower()

    @field_validator("registry")
    @classmethod
    def normalize_registry(cls, v: str) -> str:
        """Strip trailing slashes from registry URLs."""
        return v.rstrip("/")

# ✅ Valid
image = ContainerImage(
    registry="registry.company.com",
    repository="platform/nginx",
    tag="v1.2.3"
)

# ❌ Raises ValidationError
bad_image = ContainerImage(
    registry="registry.company.com",
    repository="platform/nginx",
    tag="latest"
)
```

### @field_validator with mode='before' (Pre-Processing)

```python
from pydantic import BaseModel, field_validator

class PortConfig(BaseModel):
    ports: list[int]

    @field_validator("ports", mode="before")
    @classmethod
    def parse_ports(cls, v):
        """Accept comma-separated string or list."""
        if isinstance(v, str):
            # "80,443,8080" → [80, 443, 8080]
            return [int(p.strip()) for p in v.split(",")]
        return v

config = PortConfig(ports="80,443,8080")   # String input ✅
print(config.ports)  # [80, 443, 8080]

config2 = PortConfig(ports=[80, 443])      # List input ✅
```

### @model_validator (Cross-Field Validation)

```python
from pydantic import BaseModel, model_validator
from typing import Self

class ScalingConfig(BaseModel):
    min_replicas: int
    max_replicas: int
    target_cpu_percent: float = 70.0

    @model_validator(mode="after")
    def validate_replica_range(self) -> Self:
        """min_replicas must be less than max_replicas."""
        if self.min_replicas >= self.max_replicas:
            raise ValueError(
                f"min_replicas ({self.min_replicas}) must be less than "
                f"max_replicas ({self.max_replicas})"
            )
        return self

class TLSConfig(BaseModel):
    enabled: bool
    cert_path: str | None = None
    key_path: str | None = None

    @model_validator(mode="after")
    def cert_required_when_tls_enabled(self) -> Self:
        """If TLS is enabled, cert and key paths are required."""
        if self.enabled:
            if not self.cert_path or not self.key_path:
                raise ValueError(
                    "cert_path and key_path are required when TLS is enabled"
                )
        return self

# ✅ Valid
tls = TLSConfig(enabled=True, cert_path="/etc/ssl/cert.pem", key_path="/etc/ssl/key.pem")

# ❌ ValidationError: cert_path and key_path are required when TLS is enabled
bad_tls = TLSConfig(enabled=True)
```

### @model_validator with mode='before' (Transform Before Parsing)

```python
from pydantic import BaseModel, model_validator

class KubernetesResource(BaseModel):
    kind: str
    name: str
    namespace: str

    @model_validator(mode="before")
    @classmethod
    def parse_resource_string(cls, v):
        """Accept 'kind/namespace/name' shorthand string in addition to dict."""
        if isinstance(v, str):
            parts = v.split("/")
            if len(parts) == 3:
                return {"kind": parts[0], "namespace": parts[1], "name": parts[2]}
        return v

# Both work!
r1 = KubernetesResource(kind="Deployment", namespace="default", name="nginx")
r2 = KubernetesResource.model_validate("Deployment/default/nginx")  # Shorthand!
```

---

## 8. Model Configuration with model_config

`model_config` replaces the old `class Config:` pattern from v1.

```python
from pydantic import BaseModel, ConfigDict

class KubernetesManifest(BaseModel):
    model_config = ConfigDict(
        # Populate from ORM/object attributes (not just dicts)
        from_attributes=True,

        # Allow extra fields in input (don't reject unknown keys)
        # Options: 'ignore' | 'allow' | 'forbid'
        extra="ignore",

        # Validate when fields are assigned after model creation
        validate_assignment=True,

        # Populate by either field name OR alias
        populate_by_name=True,

        # Strip whitespace from strings
        str_strip_whitespace=True,

        # Freeze the model (make it immutable / hashable)
        frozen=False,

        # Use enum values instead of enum objects in serialization
        use_enum_values=True,

        # JSON schema extras
        json_schema_extra={
            "examples": [
                {"name": "nginx", "namespace": "default"}
            ]
        }
    )

    name: str
    namespace: str = "default"


# validate_assignment example
manifest = KubernetesManifest(name="nginx")
manifest.name = "   apache   "  # Triggers validation
print(manifest.name)             # "apache" (whitespace stripped)
```

### Forbidding Extra Fields

```python
class StrictConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    
    host: str
    port: int

# ❌ Raises ValidationError — 'unknown_field' not allowed
bad = StrictConfig(host="localhost", port=5432, unknown_field="oops")
# extra inputs are not permitted [type=extra_forbidden]
```

---

## 9. Nested Models

Real infrastructure data is always nested. Pydantic handles this beautifully.

```python
from pydantic import BaseModel, Field
from typing import Optional

class ResourceRequirements(BaseModel):
    cpu_request: str = "100m"
    cpu_limit: str = "500m"
    memory_request: str = "128Mi"
    memory_limit: str = "512Mi"

class EnvironmentVariable(BaseModel):
    name: str
    value: str | None = None
    secret_key_ref: str | None = None   # Reference to a K8s secret

class ContainerSpec(BaseModel):
    name: str
    image: str
    tag: str = "latest"
    resources: ResourceRequirements = ResourceRequirements()  # Default nested model
    env_vars: list[EnvironmentVariable] = []
    ports: list[int] = []
    command: list[str] = []

class PodSpec(BaseModel):
    containers: list[ContainerSpec]
    service_account: str = "default"
    restart_policy: str = "Always"

class DeploymentManifest(BaseModel):
    name: str
    namespace: str = "default"
    replicas: int = 1
    pod_spec: PodSpec
    labels: dict[str, str] = {}
    annotations: dict[str, str] = {}

# Nested dict → Pydantic auto-parses all levels
deployment = DeploymentManifest(
    name="web-api",
    replicas=3,
    labels={"app": "web-api", "env": "production"},
    pod_spec={
        "containers": [
            {
                "name": "web-api",
                "image": "myregistry/web-api",
                "tag": "v2.1.0",
                "ports": [8080],
                "resources": {
                    "cpu_request": "200m",
                    "cpu_limit": "1000m",
                    "memory_request": "256Mi",
                    "memory_limit": "1Gi"
                },
                "env_vars": [
                    {"name": "LOG_LEVEL", "value": "INFO"},
                    {"name": "DB_PASSWORD", "secret_key_ref": "db-secret/password"}
                ]
            }
        ]
    }
)

# Type-safe access all the way down
print(deployment.pod_spec.containers[0].resources.cpu_limit)   # 1000m
print(deployment.pod_spec.containers[0].env_vars[0].name)      # LOG_LEVEL
```

---

## 10. Serialization — .model_dump() & .model_dump_json()

Serialization converts your model back to dicts or JSON.

```python
from pydantic import BaseModel, Field
from datetime import datetime

class DeploymentResult(BaseModel):
    deployment_id: str
    service: str
    status: str
    started_at: datetime
    duration_seconds: float
    metadata: dict[str, str] = {}

result = DeploymentResult(
    deployment_id="dep-001",
    service="nginx",
    status="success",
    started_at=datetime(2024, 6, 1, 12, 0, 0),
    duration_seconds=42.5
)

# Basic dump — returns a dict
data = result.model_dump()
print(type(data))   # <class 'dict'>

# Exclude specific fields
result.model_dump(exclude={"metadata"})

# Include only specific fields
result.model_dump(include={"deployment_id", "status"})

# Exclude None values (clean output)
result.model_dump(exclude_none=True)

# Exclude default values
result.model_dump(exclude_defaults=True)

# Serialize by alias (use field aliases as keys)
result.model_dump(by_alias=True)

# Serialize nested models as dicts (default) vs model instances
result.model_dump(mode="python")    # Keeps datetime as datetime object
result.model_dump(mode="json")      # Converts datetime to ISO string

# JSON string output
json_str = result.model_dump_json()
print(json_str)
# {"deployment_id":"dep-001","service":"nginx","status":"success",...}

# Pretty-printed JSON
json_pretty = result.model_dump_json(indent=2)
```

---

## 11. Deserialization — Parsing & Loading Data

```python
from pydantic import BaseModel
import json

class ClusterEvent(BaseModel):
    cluster_id: str
    event_type: str
    severity: str
    message: str

# From a dict
data = {"cluster_id": "prod-east", "event_type": "NodeNotReady", "severity": "critical", "message": "Node web-01 failed health check"}
event = ClusterEvent.model_validate(data)

# From a JSON string
json_str = '{"cluster_id": "prod-east", "event_type": "PodCrash", "severity": "warning", "message": "OOMKilled"}'
event = ClusterEvent.model_validate_json(json_str)

# From a file
with open("event.json") as f:
    event = ClusterEvent.model_validate_json(f.read())

# Parsing a list of events
events_json = '[{"cluster_id":"c1","event_type":"e1","severity":"info","message":"m1"}]'
from pydantic import TypeAdapter

adapter = TypeAdapter(list[ClusterEvent])
events = adapter.validate_json(events_json)
print(events[0].cluster_id)   # c1

# From ORM object (SQLAlchemy)
class ClusterEventORM:
    cluster_id = "prod-east"
    event_type = "NodeNotReady"
    severity = "critical"
    message = "Node failed"

class ClusterEventSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    cluster_id: str
    event_type: str
    severity: str
    message: str

orm_obj = ClusterEventORM()
event = ClusterEventSchema.model_validate(orm_obj)  # Reads from ORM attributes
```

---

## 12. Computed Fields

Computed fields are derived values calculated from other fields — read-only and included in serialization.

```python
from pydantic import BaseModel, computed_field
from functools import cached_property

class ContainerImage(BaseModel):
    registry: str
    repository: str
    tag: str

    @computed_field
    @property
    def full_image_ref(self) -> str:
        """Computed: registry/repository:tag"""
        return f"{self.registry}/{self.repository}:{self.tag}"

    @computed_field
    @cached_property
    def is_production_tag(self) -> bool:
        """Computed: check if tag looks like a semver release."""
        import re
        return bool(re.match(r'^v\d+\.\d+\.\d+$', self.tag))

image = ContainerImage(
    registry="registry.company.com",
    repository="platform/nginx",
    tag="v2.1.0"
)

print(image.full_image_ref)      # registry.company.com/platform/nginx:v2.1.0
print(image.is_production_tag)   # True

# Computed fields ARE included in serialization
print(image.model_dump())
# {'registry': ..., 'repository': ..., 'tag': ..., 'full_image_ref': '...', 'is_production_tag': True}
```

---

## 13. Model Inheritance

Use inheritance to build model hierarchies — great for shared base fields across multiple resource types.

```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import ClassVar

# Base model with common fields for all K8s-style resources
class KubernetesBase(BaseModel):
    api_version: str = "v1"
    kind: str
    name: str
    namespace: str = "default"
    labels: dict[str, str] = {}
    annotations: dict[str, str] = {}
    created_at: datetime | None = None

    # ClassVar fields are NOT included in validation/serialization
    _resource_group: ClassVar[str] = "core"

class DeploymentSpec(KubernetesBase):
    kind: str = "Deployment"             # Override default
    replicas: int = 1
    image: str
    container_port: int = 8080

class ServiceSpec(KubernetesBase):
    kind: str = "Service"
    service_type: str = "ClusterIP"
    port: int = 80
    target_port: int = 8080

class ConfigMapSpec(KubernetesBase):
    kind: str = "ConfigMap"
    data: dict[str, str] = {}

# All inherit base validation + have their own fields
deployment = DeploymentSpec(name="nginx", image="nginx:1.25", replicas=3)
service = ServiceSpec(name="nginx-svc", port=80)
configmap = ConfigMapSpec(name="app-config", data={"LOG_LEVEL": "INFO"})

print(deployment.api_version)   # v1
print(deployment.kind)          # Deployment
```

### Abstract-Style Base Models

```python
from pydantic import BaseModel, model_validator
from abc import abstractmethod

class AlertBase(BaseModel):
    severity: str
    message: str
    service: str

    @model_validator(mode="after")
    def validate_severity(self):
        allowed = {"info", "warning", "critical", "page"}
        if self.severity not in allowed:
            raise ValueError(f"severity must be one of {allowed}")
        return self

class SlackAlert(AlertBase):
    channel: str = "#alerts"
    mention_oncall: bool = False

class PagerDutyAlert(AlertBase):
    routing_key: str
    dedup_key: str | None = None

class EmailAlert(AlertBase):
    recipients: list[str]
    cc: list[str] = []

# All share base validation
slack = SlackAlert(severity="warning", message="High CPU", service="api", channel="#ops")
pager = PagerDutyAlert(severity="critical", message="Service down", service="api", routing_key="key123")
```

---

## 14. Generic Models

Generic models let you create reusable, typed wrappers.

```python
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")

# Generic API response wrapper
class APIResponse(BaseModel, Generic[T]):
    success: bool
    data: T | None = None
    error: str | None = None
    request_id: str = ""

# Generic paginated list
class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
    has_more: bool

    @property
    def total_pages(self) -> int:
        return (self.total + self.page_size - 1) // self.page_size

# Usage
class Deployment(BaseModel):
    id: str
    name: str
    status: str

# Typed responses
deployment_response: APIResponse[Deployment] = APIResponse(
    success=True,
    data=Deployment(id="dep-001", name="nginx", status="running"),
    request_id="req-abc"
)

deployments_page: PaginatedResponse[Deployment] = PaginatedResponse(
    items=[Deployment(id="dep-001", name="nginx", status="running")],
    total=42,
    page=1,
    page_size=20,
    has_more=True
)

print(deployment_response.data.name)    # nginx (fully typed!)
print(deployments_page.total_pages)     # 3
```

---

## 15. Discriminated Unions

When you have multiple possible model types sharing a common field (like `kind` or `type`), discriminated unions let Pydantic pick the right model automatically.

```python
from pydantic import BaseModel
from typing import Literal, Annotated, Union
from pydantic import Field

class DeploymentTrigger(BaseModel):
    kind: Literal["Deployment"]
    name: str
    image: str
    replicas: int = 1

class CronJobTrigger(BaseModel):
    kind: Literal["CronJob"]
    name: str
    schedule: str     # e.g. "*/5 * * * *"
    image: str

class StatefulSetTrigger(BaseModel):
    kind: Literal["StatefulSet"]
    name: str
    image: str
    storage_class: str

# Union discriminated on 'kind' field
KubernetesResource = Annotated[
    Union[DeploymentTrigger, CronJobTrigger, StatefulSetTrigger],
    Field(discriminator="kind")
]

class Pipeline(BaseModel):
    name: str
    resources: list[KubernetesResource]  # Each item auto-parsed to correct type

pipeline = Pipeline(
    name="my-pipeline",
    resources=[
        {"kind": "Deployment",   "name": "web",     "image": "web:v1",   "replicas": 3},
        {"kind": "CronJob",      "name": "cleanup",  "image": "job:v1",  "schedule": "0 2 * * *"},
        {"kind": "StatefulSet",  "name": "postgres", "image": "pg:16",   "storage_class": "fast-ssd"},
    ]
)

for resource in pipeline.resources:
    print(type(resource).__name__, "-", resource.name)
# DeploymentTrigger   - web
# CronJobTrigger      - cleanup
# StatefulSetTrigger  - postgres
```

---

## 16. Pydantic Settings — Environment & Config Management

`pydantic-settings` is the **definitive way to manage application configuration** in Python. It reads from environment variables, `.env` files, secrets directories, and more.

```python
# pip install pydantic-settings
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr, HttpUrl, Field
from functools import lru_cache

class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",                  # Load from .env file
        env_file_encoding="utf-8",
        env_prefix="APP_",                # All env vars must be prefixed APP_
        case_sensitive=False,             # APP_HOST == app_host == APP_host
        extra="ignore",                   # Ignore unknown env vars
    )

    # These map to: APP_HOST, APP_PORT, APP_DEBUG, etc.
    host: str = "0.0.0.0"
    port: int = 8000
    debug: bool = False
    environment: str = "production"

    # Database
    database_url: str = Field(..., description="PostgreSQL connection string")

    # Secrets — never logged or serialized
    secret_key: SecretStr
    api_key: SecretStr | None = None

    # External services
    redis_url: str = "redis://localhost:6379/0"
    slack_webhook: HttpUrl | None = None

    # Feature flags
    enable_ai_features: bool = False
    max_workers: int = Field(default=4, ge=1, le=64)


# Singleton pattern — settings loaded once
@lru_cache()
def get_settings() -> AppSettings:
    return AppSettings()

settings = get_settings()
print(settings.port)        # From APP_PORT env var, or default 8000
print(settings.secret_key)  # **********  (SecretStr)
```

### Multi-Environment Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import model_validator
import os

class Settings(BaseSettings):
    environment: str = "development"
    debug: bool = False
    log_level: str = "INFO"
    database_url: str = "sqlite:///./dev.db"

    # Kubernetes-specific
    k8s_namespace: str = "default"
    k8s_in_cluster: bool = False

    @model_validator(mode="after")
    def configure_for_environment(self):
        """Auto-adjust settings based on environment."""
        if self.environment == "production":
            if self.debug:
                raise ValueError("debug=True is not allowed in production!")
            if "sqlite" in self.database_url:
                raise ValueError("SQLite is not allowed in production!")
        return self

    model_config = SettingsConfigDict(env_file=".env")


# .env.development
# ENVIRONMENT=development
# DATABASE_URL=postgresql://localhost/dev_db
# DEBUG=true

# .env.production
# ENVIRONMENT=production
# DATABASE_URL=postgresql://prod-host/prod_db
# SECRET_KEY=real-secret-from-vault
# K8S_IN_CLUSTER=true
```

### Layered Settings (Multiple Sources)

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic_settings import PydanticBaseSettingsSource, JsonConfigSettingsSource
from typing import Tuple, Type

class LayeredSettings(BaseSettings):
    """
    Priority (highest to lowest):
    1. Environment variables
    2. .env file
    3. config.json
    4. Default values
    """
    model_config = SettingsConfigDict(
        env_file=".env",
        json_file="config.json"   # Also reads from a JSON file
    )

    app_name: str = "my-api"
    port: int = 8000
    replicas: int = 1

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: Type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> Tuple[PydanticBaseSettingsSource, ...]:
        return (
            env_settings,       # Highest priority
            dotenv_settings,
            JsonConfigSettingsSource(settings_cls),
            init_settings,      # Lowest priority
        )
```

---

## 17. Custom Types & Annotated Types

Create reusable types with built-in validation.

```python
from typing import Annotated
from pydantic import BaseModel, Field, BeforeValidator, AfterValidator
from pydantic.functional_validators import PlainValidator

# Reusable annotated types
DNSName = Annotated[
    str,
    Field(
        pattern=r'^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$',
        min_length=1,
        max_length=63
    )
]

KubernetesNamespace = Annotated[
    str,
    Field(
        default="default",
        pattern=r'^[a-z0-9][a-z0-9-]*$',
        max_length=253
    )
]

PortNumber = Annotated[int, Field(ge=1, le=65535)]

SemVer = Annotated[
    str,
    Field(pattern=r'^v?\d+\.\d+\.\d+(-[a-zA-Z0-9.]+)?$')
]

CpuResource = Annotated[
    str,
    Field(pattern=r'^\d+(\.\d+)?m?$')   # "500m", "1", "2.5"
]

MemoryResource = Annotated[
    str,
    Field(pattern=r'^\d+(Ki|Mi|Gi|Ti|Pi|Ei|K|M|G|T|P|E)?$')
]

# Use them anywhere
class ServiceDefinition(BaseModel):
    name: DNSName
    namespace: KubernetesNamespace = "default"
    version: SemVer
    port: PortNumber = 8080
    cpu_limit: CpuResource = "500m"
    memory_limit: MemoryResource = "512Mi"


# Custom validator functions with BeforeValidator
def normalize_tag(v: str) -> str:
    """Strip 'v' prefix from version tags."""
    return v.lstrip("v")

NormalizedVersion = Annotated[str, BeforeValidator(normalize_tag)]

class Image(BaseModel):
    repository: str
    version: NormalizedVersion

img = Image(repository="nginx", version="v1.25.0")
print(img.version)   # 1.25.0 (v prefix stripped)
```

---

## 18. Strict Mode & Type Coercion

By default, Pydantic coerces types. **Strict mode** disables this.

```python
from pydantic import BaseModel, ConfigDict, Field
from typing import Annotated
from pydantic import Strict

# Model-level strict mode
class StrictModel(BaseModel):
    model_config = ConfigDict(strict=True)

    port: int
    enabled: bool

# ❌ Raises ValidationError — "443" is not an int in strict mode
StrictModel(port="443", enabled=True)

# ✅ Works
StrictModel(port=443, enabled=True)


# Field-level strict mode (mix strict and non-strict)
class MixedModel(BaseModel):
    name: str                               # Non-strict: coerces if needed
    port: Annotated[int, Strict()]          # Strict: must be exactly int
    replicas: int                           # Non-strict: coerces "3" → 3

# ✅ "nginx" is already a str, port=443 is already int
MixedModel(name="nginx", port=443, replicas="3")   # replicas coerced ✅

# ❌ port="443" fails strict check
MixedModel(name="nginx", port="443", replicas=3)
```

---

## 19. Handling Errors & ValidationError

Proper error handling is critical in production.

```python
from pydantic import BaseModel, ValidationError, field_validator
import json

class ServicePayload(BaseModel):
    name: str
    port: int
    environment: str

def parse_service_payload(raw_data: dict) -> ServicePayload | None:
    try:
        return ServicePayload.model_validate(raw_data)
    except ValidationError as e:
        # e.errors() returns a list of detailed error dicts
        for error in e.errors():
            field = " → ".join(str(loc) for loc in error["loc"])
            print(f"Field: {field}")
            print(f"  Error: {error['msg']}")
            print(f"  Type: {error['type']}")
            print(f"  Input: {error['input']}")
        return None

# Multiple errors at once — Pydantic collects ALL errors
bad_data = {
    "name": "",          # Too short
    "port": "not-int",  # Wrong type
    # environment missing entirely
}

parse_service_payload(bad_data)
# Field: name
#   Error: String should have at least 1 character
# Field: port
#   Error: Input should be a valid integer
# Field: environment
#   Error: Field required
```

### Structured Error Responses (for APIs)

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import ValidationError

app = FastAPI()

@app.exception_handler(ValidationError)
async def validation_exception_handler(request: Request, exc: ValidationError):
    """Return structured validation errors as JSON."""
    errors = []
    for error in exc.errors():
        errors.append({
            "field": ".".join(str(loc) for loc in error["loc"]),
            "message": error["msg"],
            "type": error["type"],
        })
    return JSONResponse(
        status_code=422,
        content={"success": False, "errors": errors}
    )
```

---

## 20. Pydantic with FastAPI

FastAPI and Pydantic are built for each other. This integration is covered in depth in the FastAPI guide, but here are key patterns.

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()

# Request body — parsed and validated automatically
class CreateDeploymentRequest(BaseModel):
    service_name: str = Field(..., min_length=2)
    image_tag: str
    replicas: int = Field(default=1, ge=1, le=50)

# Response model — shapes and validates output
class DeploymentResponse(BaseModel):
    deployment_id: str
    status: str
    message: str

# FastAPI uses these models for:
# 1. Automatic request validation
# 2. Auto-generated Swagger docs
# 3. Response serialization
# 4. OpenAPI schema generation
@app.post("/deployments", response_model=DeploymentResponse, status_code=201)
def create_deployment(request: CreateDeploymentRequest):
    return DeploymentResponse(
        deployment_id="dep-001",
        status="created",
        message=f"Deployment for {request.service_name} created"
    )
```

---

## 21. Pydantic with SQLAlchemy & ORMs

The `from_attributes=True` config enables reading from ORM objects.

```python
from sqlalchemy import Column, String, Integer, Boolean
from sqlalchemy.ext.declarative import declarative_base
from pydantic import BaseModel, ConfigDict

SQLBase = declarative_base()

# SQLAlchemy ORM Model (database layer)
class ServiceORM(SQLBase):
    __tablename__ = "services"
    id = Column(String, primary_key=True)
    name = Column(String)
    port = Column(Integer)
    is_active = Column(Boolean, default=True)

# Pydantic Schema (API layer)
class ServiceBase(BaseModel):
    name: str
    port: int

class ServiceCreate(ServiceBase):
    pass  # Fields for creation

class ServiceRead(ServiceBase):
    model_config = ConfigDict(from_attributes=True)  # Key setting!
    id: str
    is_active: bool

class ServiceUpdate(BaseModel):
    name: str | None = None
    port: int | None = None
    is_active: bool | None = None

# In a FastAPI route:
# orm_obj = db.query(ServiceORM).filter_by(id=service_id).first()
# return ServiceRead.model_validate(orm_obj)  # ORM → Pydantic ✅
```

---

## 22. Pydantic for Infrastructure & Kubernetes Models

Model your entire infrastructure config with Pydantic for validation before applying.

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Literal
from enum import Enum

class RestartPolicy(str, Enum):
    always = "Always"
    on_failure = "OnFailure"
    never = "Never"

class ResourceSpec(BaseModel):
    cpu: str = "100m"
    memory: str = "128Mi"

    @field_validator("cpu")
    @classmethod
    def valid_cpu(cls, v: str) -> str:
        import re
        if not re.match(r'^\d+(\.\d+)?m?$', v):
            raise ValueError(f"Invalid CPU format: {v}. Use '500m' or '0.5'")
        return v

class Resources(BaseModel):
    requests: ResourceSpec = ResourceSpec()
    limits: ResourceSpec = ResourceSpec()

    @model_validator(mode="after")
    def limits_gte_requests(self):
        """CPU limits should be >= requests (simplified check)."""
        return self

class ContainerDef(BaseModel):
    name: str
    image: str
    image_tag: str = "latest"
    ports: list[int] = []
    resources: Resources = Resources()
    liveness_probe_path: str | None = "/health"
    readiness_probe_path: str | None = "/health"

    @property
    def full_image(self) -> str:
        return f"{self.image}:{self.image_tag}"

class HelmValues(BaseModel):
    """Validates Helm values.yaml before a helm upgrade."""
    replicaCount: int = Field(default=1, ge=1, le=100)
    image: dict[str, str]
    service: dict[str, str | int]
    ingress: dict[str, bool | str | list] = {}
    resources: Resources = Resources()
    autoscaling: dict[str, bool | int] = {}
    nodeSelector: dict[str, str] = {}
    tolerations: list[dict] = []
    affinity: dict = {}

def validate_helm_values(values_dict: dict) -> HelmValues:
    """Parse and validate helm values before deployment."""
    try:
        return HelmValues.model_validate(values_dict)
    except ValidationError as e:
        print("❌ Invalid Helm values:")
        for err in e.errors():
            print(f"  - {err['loc']}: {err['msg']}")
        raise
```

---

## 23. Pydantic in AI/ML DevOps Pipelines

Pydantic is the backbone of AI frameworks like LangChain, LlamaIndex, and Instructor. It ensures AI inputs and outputs are clean, typed, and validated.

### Structured LLM Output Parsing

```python
from pydantic import BaseModel, Field
from typing import Literal
import json

class IncidentSeverity(str, Enum):
    P1 = "P1"
    P2 = "P2"
    P3 = "P3"
    P4 = "P4"

class IncidentAnalysis(BaseModel):
    """Structured output schema for LLM incident analysis."""
    root_cause: str = Field(..., description="The identified root cause of the incident")
    severity: IncidentSeverity
    affected_services: list[str]
    remediation_steps: list[str] = Field(..., min_length=1, max_length=10)
    estimated_resolution_minutes: int = Field(..., ge=1, le=1440)
    requires_escalation: bool
    confidence_score: float = Field(..., ge=0.0, le=1.0)

def parse_llm_analysis(llm_json_output: str) -> IncidentAnalysis:
    """Parse and validate LLM output into typed structure."""
    try:
        raw = json.loads(llm_json_output)
        return IncidentAnalysis.model_validate(raw)
    except (json.JSONDecodeError, ValidationError) as e:
        raise ValueError(f"LLM returned invalid analysis format: {e}")

# Instructor library uses Pydantic directly with OpenAI
# pip install instructor
import instructor
from openai import OpenAI

client = instructor.from_openai(OpenAI())

def analyze_incident_with_ai(logs: str) -> IncidentAnalysis:
    return client.chat.completions.create(
        model="gpt-4o",
        response_model=IncidentAnalysis,   # Pydantic model as schema!
        messages=[{
            "role": "user",
            "content": f"Analyze this incident:\n{logs}"
        }]
    )
```

### ML Feature Validation

```python
from pydantic import BaseModel, field_validator, model_validator
import numpy as np

class InferenceRequest(BaseModel):
    """Validates inputs before sending to an ML model."""
    model_config = ConfigDict(arbitrary_types_allowed=True)

    cpu_utilization: float = Field(..., ge=0.0, le=100.0)
    memory_utilization: float = Field(..., ge=0.0, le=100.0)
    request_rate_per_sec: float = Field(..., ge=0.0)
    error_rate_percent: float = Field(..., ge=0.0, le=100.0)
    latency_p99_ms: float = Field(..., ge=0.0)

    @model_validator(mode="after")
    def check_realistic_values(self):
        """Cross-field sanity check for ML features."""
        if self.error_rate_percent > 50 and self.request_rate_per_sec < 1:
            raise ValueError(
                "High error rate with near-zero traffic is suspicious. "
                "Verify the metrics source."
            )
        return self

    def to_feature_vector(self) -> list[float]:
        return [
            self.cpu_utilization / 100,
            self.memory_utilization / 100,
            min(self.request_rate_per_sec / 10000, 1.0),  # Normalize
            self.error_rate_percent / 100,
            min(self.latency_p99_ms / 5000, 1.0)
        ]

class InferenceResponse(BaseModel):
    is_anomaly: bool
    anomaly_score: float = Field(..., ge=0.0, le=1.0)
    top_contributing_features: list[str]
    recommended_action: str
```

---

## 24. Performance Tips & Advanced Patterns

### Model Caching with TypeAdapter

```python
from pydantic import TypeAdapter

# For validating non-model types efficiently
# TypeAdapter is faster than wrapping in a throwaway BaseModel

list_adapter = TypeAdapter(list[int])
result = list_adapter.validate_python(["1", "2", "3"])
print(result)  # [1, 2, 3]

dict_adapter = TypeAdapter(dict[str, list[str]])
result = dict_adapter.validate_json('{"a": ["1", "2"], "b": ["3"]}')
```

### Model Rebuild for Dynamic Fields

```python
from pydantic import BaseModel, create_model
from typing import Any

# Dynamically create models at runtime
def create_config_model(fields: dict[str, type]) -> type[BaseModel]:
    """Dynamically create a Pydantic model from a dict of field definitions."""
    field_definitions = {
        name: (field_type, ...) for name, field_type in fields.items()
    }
    return create_model("DynamicConfig", **field_definitions)

# Build model from discovered config schema
config_schema = {"host": str, "port": int, "debug": bool}
DynamicConfig = create_config_model(config_schema)

config = DynamicConfig(host="localhost", port=8080, debug=False)
print(config.model_dump())   # {'host': 'localhost', 'port': 8080, 'debug': False}
```

### Immutable Models (Frozen)

```python
from pydantic import BaseModel, ConfigDict

class ImmutableConfig(BaseModel):
    model_config = ConfigDict(frozen=True)  # Hashable + immutable

    host: str
    port: int

config = ImmutableConfig(host="localhost", port=5432)

# ❌ Raises PydanticUserError — frozen models can't be mutated
config.port = 3306

# ✅ Frozen models are hashable — can be used as dict keys or in sets
config_set = {config}
cache = {config: "some cached value"}
```

### Partial Updates Pattern (PATCH)

```python
from pydantic import BaseModel

class ServiceFull(BaseModel):
    name: str
    port: int
    replicas: int
    environment: str

# All fields optional — for PATCH requests
class ServicePatch(BaseModel):
    name: str | None = None
    port: int | None = None
    replicas: int | None = None
    environment: str | None = None

def apply_patch(original: ServiceFull, patch: ServicePatch) -> ServiceFull:
    """Apply partial update, keeping original values for unset fields."""
    # model_dump(exclude_unset=True) only returns fields that were explicitly set
    update_data = patch.model_dump(exclude_unset=True)
    updated = original.model_dump()
    updated.update(update_data)
    return ServiceFull.model_validate(updated)

original = ServiceFull(name="nginx", port=80, replicas=3, environment="prod")
patch = ServicePatch(replicas=5)   # Only update replicas

updated = apply_patch(original, patch)
print(updated.replicas)      # 5   (updated)
print(updated.port)          # 80  (preserved)
```

---

## 25. Quick Reference Cheatsheet

```python
# ─── INSTALLATION ─────────────────────────────────────────────────────────────
# pip install pydantic pydantic-settings "pydantic[email]"

# ─── BASIC MODEL ──────────────────────────────────────────────────────────────
from pydantic import BaseModel, Field, ConfigDict

class MyModel(BaseModel):
    name: str
    value: int = Field(default=0, ge=0)
    optional: str | None = None

# ─── FIELD CONSTRAINTS ────────────────────────────────────────────────────────
Field(..., min_length=1, max_length=100)   # String length
Field(..., ge=0, le=100)                   # Numeric: >=, <=
Field(..., gt=0, lt=100)                   # Numeric: >, <
Field(..., pattern=r'^[a-z]+$')            # Regex pattern
Field(..., alias="camelCaseName")          # JSON alias

# ─── VALIDATORS ───────────────────────────────────────────────────────────────
from pydantic import field_validator, model_validator

@field_validator("field_name")             # Single field
@field_validator("a", "b", "c")           # Multiple fields
@field_validator("field", mode="before")   # Pre-process input
@model_validator(mode="after")             # Cross-field (runs after parsing)
@model_validator(mode="before")            # Transform raw input

# ─── CONFIG ───────────────────────────────────────────────────────────────────
model_config = ConfigDict(
    extra="ignore",           # or "allow", "forbid"
    frozen=True,              # Immutable + hashable
    str_strip_whitespace=True,
    validate_assignment=True,
    from_attributes=True,     # ORM support
    use_enum_values=True,
    populate_by_name=True,
)

# ─── SERIALIZATION ────────────────────────────────────────────────────────────
obj.model_dump()                           # → dict
obj.model_dump(exclude_none=True)          # Skip None values
obj.model_dump(exclude_unset=True)         # Skip unset values (PATCH pattern)
obj.model_dump(include={"a", "b"})         # Only specific fields
obj.model_dump(by_alias=True)              # Use aliases as keys
obj.model_dump_json()                      # → JSON string
obj.model_dump_json(indent=2)              # → Pretty JSON

# ─── PARSING ─────────────────────────────────────────────────────────────────
MyModel.model_validate(dict_data)          # From dict
MyModel.model_validate_json(json_str)      # From JSON string
MyModel.model_validate(orm_obj)            # From ORM (from_attributes=True)

# ─── ERRORS ───────────────────────────────────────────────────────────────────
from pydantic import ValidationError
try:
    obj = MyModel(...)
except ValidationError as e:
    e.errors()         # List of error dicts
    e.error_count()    # Number of errors
    str(e)             # Human-readable string

# ─── SETTINGS ─────────────────────────────────────────────────────────────────
from pydantic_settings import BaseSettings, SettingsConfigDict
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_")
    port: int = 8000
    secret: SecretStr

# ─── COMPUTED FIELDS ──────────────────────────────────────────────────────────
from pydantic import computed_field
@computed_field
@property
def full_url(self) -> str:
    return f"{self.host}:{self.port}"

# ─── SPECIAL TYPES ────────────────────────────────────────────────────────────
from pydantic import HttpUrl, SecretStr, EmailStr, IPvAnyAddress

# ─── GENERIC MODELS ───────────────────────────────────────────────────────────
from typing import Generic, TypeVar
T = TypeVar("T")
class Response(BaseModel, Generic[T]):
    data: T
    success: bool

# ─── TYPE ADAPTER (Non-model validation) ──────────────────────────────────────
from pydantic import TypeAdapter
ta = TypeAdapter(list[int])
ta.validate_python(["1", "2"])  # [1, 2]
```

---

## Final Words

Pydantic is not just a validation library — it's the **type-safety backbone** of the modern Python DevOps and AI stack. Mastering it means your code:

- **Fails fast** — bad data is caught at the boundary, not deep in your pipeline
- **Self-documents** — your models are your API contracts, your config schemas, your infrastructure specs
- **Integrates everywhere** — FastAPI, SQLAlchemy, LangChain, Instructor, AWS Lambda, Kubernetes operators

**Junior Level** → `BaseModel`, `Field()`, basic validators, `BaseSettings` for `.env` files.

**Mid Level** → Nested models, cross-field validators, ORM integration, discriminated unions, error handling patterns.

**Senior Level** → Generic models, dynamic model creation, `TypeAdapter`, custom annotated types, layered settings, Pydantic as the schema backbone for AI pipelines.

---

*Built with ❤️ for DevOps Engineers in the AI-native infrastructure world.*  
*Pydantic version referenced: 2.x | pydantic-settings 2.x | Python 3.12+*
