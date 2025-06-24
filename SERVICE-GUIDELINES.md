# Service Development Guidelines

A comprehensive guide for building robust, scalable, and maintainable services based on modern best practices and real-world experience.

## Table of Contents

1. [Service Architecture](#service-architecture)
2. [Project Structure](#project-structure)
3. [API Design](#api-design)
4. [Database Design](#database-design)
5. [Configuration Management](#configuration-management)
6. [Error Handling](#error-handling)
7. [Testing Strategy](#testing-strategy)
8. [Documentation](#documentation)
9. [Deployment & DevOps](#deployment--devops)
10. [Monitoring & Observability](#monitoring--observability)
11. [Security](#security)
12. [Performance](#performance)

---

## Service Architecture

### 1. Layered Architecture Pattern

Structure your service with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│                    API Layer                            │
│  Routes, Controllers, Request/Response Handling         │
├─────────────────────────────────────────────────────────┤
│                 Service Layer                           │
│  Business Logic, Orchestration, Workflow Management     │
├─────────────────────────────────────────────────────────┤
│                 Data Layer                              │
│  Database Models, ORM, Data Access Logic               │
├─────────────────────────────────────────────────────────┤
│               External Services                         │
│  Third-party APIs, Message queues, External databases   │
└─────────────────────────────────────────────────────────┘
```

### 2. Service Layer Pattern

**Example Structure:**
```python
# Main orchestration service
class UserExtractionService:
    def __init__(self):
        self.external_service = ExternalApiService()
        self.active_jobs = set()
    
    def start_extraction(self, config):
        # Orchestrates the entire workflow
        pass
    
    def get_status(self, job_id):
        # Business logic for status checking
        pass

# Dedicated external service handler
class ExternalApiService:
    def __init__(self):
        self.base_url = config.API_BASE_URL
        self.timeout = config.REQUEST_TIMEOUT
    
    def authenticate(self, credentials):
        # Handle authentication logic
        pass
    
    def fetch_data(self, params):
        # Handle data fetching with retries, rate limiting
        pass
```

### 3. Configuration-Driven Design

**Principle:** All behavior should be configurable without code changes.

```python
class Config:
    # Database Configuration
    DATABASE_URL = os.getenv('DATABASE_URL')
    
    # External API Configuration
    API_BASE_URL = os.getenv('API_BASE_URL')
    API_TIMEOUT = int(os.getenv('API_TIMEOUT', 30))
    
    # Application Settings
    BATCH_SIZE = int(os.getenv('BATCH_SIZE', 100))
    MAX_RETRIES = int(os.getenv('MAX_RETRIES', 3))
    
    # Environment-specific configurations
    @staticmethod
    def get_config():
        env = os.getenv('FLASK_ENV', 'development')
        return config_map.get(env, DevelopmentConfig)
```

---

## Project Structure

### Recommended Directory Layout

```
service-name/
├── app.py                          # Application entry point
├── config.py                       # Configuration management
├── requirements.txt                # Python dependencies
├── requirements-test.txt           # Test dependencies
├── .env.example                    # Environment variables template
├── docker-compose.yml              # Docker orchestration
├── Dockerfile                      # Container definition
├── README.md                       # Comprehensive documentation
├── setup.ps1                       # Windows setup script
├── Makefile                        # Unix/Linux commands
│
├── models/                         # Data layer
│   ├── __init__.py
│   ├── base.py                     # Base model classes
│   ├── user.py                     # Domain models
│   └── database.py                 # Database setup
│
├── services/                       # Business logic layer
│   ├── __init__.py
│   ├── orchestration_service.py    # Main business logic
│   ├── external_api_service.py     # External integrations
│   └── data_processing_service.py  # Data transformation
│
├── api/                           # API layer
│   ├── __init__.py
│   ├── routes.py                  # Route definitions
│   ├── schemas.py                 # Request/response schemas
│   └── middleware.py              # API middleware
│
├── utils/                         # Shared utilities
│   ├── __init__.py
│   ├── validators.py              # Input validation
│   ├── helpers.py                 # Helper functions
│   └── exceptions.py              # Custom exceptions
│
├── tests/                         # Test suite
│   ├── __init__.py
│   ├── test_api.py                # API integration tests
│   ├── test_services.py           # Service unit tests
│   ├── test_models.py             # Model tests
│   └── fixtures/                  # Test data
│
├── docs/                          # Additional documentation
│   ├── api.md                     # API documentation
│   ├── deployment.md              # Deployment guide
│   └── troubleshooting.md         # Common issues
│
└── scripts/                       # Utility scripts
    ├── init_db.py                 # Database initialization
    ├── seed_data.py               # Test data seeding
    └── cleanup.py                 # Maintenance scripts
```

### File Organization Principles

1. **Single Responsibility**: Each file has one clear purpose
2. **Logical Grouping**: Related functionality is grouped together
3. **Clear Naming**: File names clearly indicate their purpose
4. **Dependency Direction**: Dependencies flow inward (API → Services → Models)

---

## API Design

### 1. RESTful API Standards

**Resource-Based URLs:**
```
GET    /api/resources              # List all resources
POST   /api/resources              # Create new resource
GET    /api/resources/{id}         # Get specific resource
PUT    /api/resources/{id}         # Update resource (full)
PATCH  /api/resources/{id}         # Update resource (partial)
DELETE /api/resources/{id}         # Delete resource
```

**Status-Based Operations:**
```
POST   /api/scan/{id}/start        # Start a job
GET    /api/scan/{id}/status       # Check job status
GET    /api/scan/{id}/result       # Get job results
POST   /api/scan/{id}/cancel       # Cancel job
DELETE /api/scan/{id}              # Delete job
```

### 2. HTTP Status Codes

Use appropriate HTTP status codes:

```python
# Success responses
200 OK           # Successful GET, PUT, PATCH
201 Created      # Successful POST
202 Accepted     # Async operation started
204 No Content   # Successful DELETE

# Client error responses
400 Bad Request           # Invalid request data
401 Unauthorized         # Authentication required
403 Forbidden           # Access denied
404 Not Found           # Resource doesn't exist
409 Conflict            # Resource conflict (e.g., already exists)
422 Unprocessable Entity # Validation failed

# Server error responses
500 Internal Server Error # Unexpected server error
502 Bad Gateway          # External service error
503 Service Unavailable  # Temporary service issue
```

### 3. Request/Response Schema Validation

**Using Marshmallow for Python:**
```python
from marshmallow import Schema, fields, validate

class JobStartSchema(Schema):
    name = fields.Str(required=True, validate=validate.Length(min=1, max=255))
    config = fields.Dict(required=True)
    priority = fields.Int(validate=validate.Range(min=1, max=10), load_default=5)

class JobStatusSchema(Schema):
    id = fields.Str()
    status = fields.Str(validate=validate.OneOf(['pending', 'running', 'completed', 'failed']))
    started_at = fields.DateTime(format='iso')
    completed_at = fields.DateTime(format='iso', allow_none=True)
    progress = fields.Int(validate=validate.Range(min=0, max=100))
    error = fields.Str(allow_none=True)
```

### 4. API Documentation

**Use OpenAPI/Swagger:**
```python
from flask_restx import Api, Resource, fields

api = Api(
    version='1.0.0',
    title='Service API',
    description='Comprehensive API for service operations',
    doc='/docs/',
    validate=True
)

# Define models for documentation
job_model = api.model('Job', {
    'id': fields.String(description='Job identifier'),
    'status': fields.String(description='Current status'),
    'created_at': fields.DateTime(description='Creation timestamp')
})

@api.route('/jobs')
class JobList(Resource):
    @api.doc('list_jobs')
    @api.marshal_list_with(job_model)
    def get(self):
        """List all jobs"""
        return jobs
```

---

## Database Design

### 1. Entity Relationship Design

**Core Principles:**
- **Normalization**: Eliminate data redundancy
- **Referential Integrity**: Use foreign keys appropriately
- **Indexing Strategy**: Index frequently queried columns
- **Scalability**: Design for growth

**Example Schema:**
```python
from sqlalchemy import Column, String, DateTime, ForeignKey, Index
from sqlalchemy.orm import relationship

class Job(Base):
    __tablename__ = 'jobs'
    
    id = Column(String, primary_key=True, default=generate_uuid)
    name = Column(String, nullable=False)
    status = Column(String, nullable=False, index=True)
    config = Column(JSON, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    results = relationship("JobResult", back_populates="job", cascade="all, delete-orphan")
    
    # Indexes for performance
    __table_args__ = (
        Index('idx_job_status_created', 'status', 'created_at'),
        Index('idx_job_name', 'name'),
    )

class JobResult(Base):
    __tablename__ = 'job_results'
    
    id = Column(String, primary_key=True, default=generate_uuid)
    job_id = Column(String, ForeignKey('jobs.id'), nullable=False)
    data = Column(JSON, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relationship
    job = relationship("Job", back_populates="results")
```

### 2. Database Operations

**Session Management:**
```python
from contextlib import contextmanager

@contextmanager
def get_db_session():
    session = Session()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
def save_job(job_data):
    with get_db_session() as session:
        job = Job(**job_data)
        session.add(job)
        return job.id
```

### 3. Migration Strategy

**Use Database Migration Tools:**
```python
# Using Alembic for SQLAlchemy
from alembic import command
from alembic.config import Config

def run_migrations():
    alembic_cfg = Config("alembic.ini")
    command.upgrade(alembic_cfg, "head")
```

---

## Configuration Management

### 1. Environment-Based Configuration

```python
import os
from dotenv import load_dotenv

load_dotenv()

class BaseConfig:
    """Base configuration class"""
    SECRET_KEY = os.getenv('SECRET_KEY', 'dev-secret-key')
    DATABASE_URL = os.getenv('DATABASE_URL')
    
    # Application settings
    BATCH_SIZE = int(os.getenv('BATCH_SIZE', 100))
    REQUEST_TIMEOUT = int(os.getenv('REQUEST_TIMEOUT', 30))
    
    # External service settings
    EXTERNAL_API_BASE_URL = os.getenv('EXTERNAL_API_BASE_URL')
    EXTERNAL_API_TIMEOUT = int(os.getenv('EXTERNAL_API_TIMEOUT', 30))

class DevelopmentConfig(BaseConfig):
    DEBUG = True
    LOG_LEVEL = 'DEBUG'

class ProductionConfig(BaseConfig):
    DEBUG = False
    LOG_LEVEL = 'WARNING'

class TestingConfig(BaseConfig):
    TESTING = True
    DATABASE_URL = os.getenv('TEST_DATABASE_URL', 'sqlite:///:memory:')

config_map = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'testing': TestingConfig
}
```

### 2. Configuration Validation

```python
class ConfigValidator:
    @staticmethod
    def validate_config(config):
        required_vars = [
            'DATABASE_URL',
            'SECRET_KEY',
            'EXTERNAL_API_BASE_URL'
        ]
        
        missing_vars = []
        for var in required_vars:
            if not getattr(config, var, None):
                missing_vars.append(var)
        
        if missing_vars:
            raise ValueError(f"Missing required configuration: {', '.join(missing_vars)}")
```

---

## Error Handling

### 1. Custom Exception Hierarchy

```python
class ServiceException(Exception):
    """Base exception for service errors"""
    def __init__(self, message, error_code=None, details=None):
        self.message = message
        self.error_code = error_code
        self.details = details or {}
        super().__init__(self.message)

class ValidationError(ServiceException):
    """Raised when input validation fails"""
    pass

class ExternalServiceError(ServiceException):
    """Raised when external service calls fail"""
    pass

class DatabaseError(ServiceException):
    """Raised when database operations fail"""
    pass
```

### 2. Global Error Handling

```python
from flask import jsonify
import logging

logger = logging.getLogger(__name__)

@app.errorhandler(ValidationError)
def handle_validation_error(error):
    logger.warning(f"Validation error: {error.message}")
    return jsonify({
        'error': 'Validation failed',
        'message': error.message,
        'details': error.details
    }), 400

@app.errorhandler(ExternalServiceError)
def handle_external_service_error(error):
    logger.error(f"External service error: {error.message}")
    return jsonify({
        'error': 'External service error',
        'message': 'An external service is temporarily unavailable'
    }), 502

@app.errorhandler(Exception)
def handle_unexpected_error(error):
    logger.error(f"Unexpected error: {str(error)}", exc_info=True)
    return jsonify({
        'error': 'Internal server error',
        'message': 'An unexpected error occurred'
    }), 500
```

### 3. Retry Logic

```python
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=1, backoff=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            retries = 0
            current_delay = delay
            
            while retries < max_retries:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    retries += 1
                    if retries >= max_retries:
                        raise e
                    
                    logger.warning(f"Attempt {retries} failed, retrying in {current_delay}s: {str(e)}")
                    time.sleep(current_delay)
                    current_delay *= backoff
            
        return wrapper
    return decorator

# Usage
@retry_on_failure(max_retries=3, delay=1, backoff=2)
def call_external_api(url, data):
    response = requests.post(url, json=data, timeout=30)
    response.raise_for_status()
    return response.json()
```

---

## Testing Strategy

### 1. Test Pyramid

```
┌─────────────────────────────────────┐
│           E2E Tests                 │  ← Few, slow, expensive
│         (Full workflow)             │
├─────────────────────────────────────┤
│        Integration Tests            │  ← Some, medium speed
│     (API endpoints, DB, etc.)       │
├─────────────────────────────────────┤
│           Unit Tests                │  ← Many, fast, cheap
│    (Individual functions/methods)   │
└─────────────────────────────────────┘
```

### 2. Test Organization

```python
# test_services.py - Unit tests
import pytest
from unittest.mock import Mock, patch
from services.job_service import JobService

class TestJobService:
    def setup_method(self):
        self.service = JobService()
    
    def test_start_job_success(self):
        # Test successful job start
        config = {'key': 'value'}
        job_id = self.service.start_job('test-job', config)
        assert job_id is not None
    
    @patch('services.job_service.external_api')
    def test_start_job_external_failure(self, mock_api):
        # Test handling of external API failure
        mock_api.authenticate.side_effect = Exception("API Error")
        
        with pytest.raises(ExternalServiceError):
            self.service.start_job('test-job', {})

# test_api.py - Integration tests
class TestJobAPI:
    def test_complete_job_workflow(self, client):
        # Test complete workflow from start to finish
        
        # Start job
        response = client.post('/api/jobs/start', json={
            'name': 'test-job',
            'config': {'key': 'value'}
        })
        assert response.status_code == 202
        
        job_id = response.json['job_id']
        
        # Check status
        response = client.get(f'/api/jobs/{job_id}/status')
        assert response.status_code == 200
        assert response.json['status'] in ['pending', 'running', 'completed']
```

### 3. Test Configuration

```python
# conftest.py - Test fixtures
import pytest
from app import create_app
from models import init_db

@pytest.fixture(scope='session')
def app():
    app = create_app('testing')
    with app.app_context():
        init_db()
        yield app

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def sample_job_data():
    return {
        'name': 'test-job',
        'config': {'batch_size': 10}
    }
```

---

## Documentation

### 1. README Structure

A comprehensive README should include:

```markdown
# Service Name

Brief description and key features

## Quick Start
- Installation steps
- Basic usage examples
- Links to detailed documentation

## API Documentation
- Endpoint reference
- Authentication details
- Request/response examples

## Configuration
- Environment variables
- Configuration options
- Example configurations

## Deployment
- Docker setup
- Environment-specific deployments
- Scaling considerations

## Monitoring
- Health checks
- Metrics and logging
- Troubleshooting guide

## Contributing
- Development setup
- Code standards
- Testing requirements
```

### 2. Code Documentation

```python
class JobService:
    """
    Service for managing job lifecycle and orchestration.
    
    This service handles job creation, execution monitoring, and result retrieval.
    It coordinates between the API layer and external services while maintaining
    job state in the database.
    
    Attributes:
        external_service: Service for external API communication
        active_jobs: Set of currently running job IDs
    """
    
    def start_job(self, name: str, config: dict) -> str:
        """
        Start a new job with the given configuration.
        
        Args:
            name: Human-readable name for the job
            config: Dictionary containing job configuration parameters
            
        Returns:
            str: Unique job identifier
            
        Raises:
            ValidationError: If configuration is invalid
            ExternalServiceError: If external service authentication fails
            
        Example:
            >>> service = JobService()
            >>> job_id = service.start_job('data-sync', {'source': 'api'})
            >>> print(f"Started job: {job_id}")
        """
        pass
```

### 3. API Documentation

Use tools like Swagger/OpenAPI for interactive documentation:

```yaml
# openapi.yaml
openapi: 3.0.0
info:
  title: Service API
  version: 1.0.0
  description: Comprehensive API for service operations

paths:
  /api/jobs:
    post:
      summary: Create a new job
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/JobRequest'
      responses:
        '202':
          description: Job created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/JobResponse'

components:
  schemas:
    JobRequest:
      type: object
      required:
        - name
        - config
      properties:
        name:
          type: string
          description: Job name
        config:
          type: object
          description: Job configuration
```

---

## Deployment & DevOps

### 1. Docker Configuration

**Multi-stage Dockerfile:**
```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.11-slim

WORKDIR /app

# Copy installed packages from builder stage
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Copy application code
COPY . .

# Create non-root user
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health', timeout=5)"

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

**Docker Compose:**
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/dbname
      - FLASK_ENV=production
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: dbname
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d dbname"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### 2. Environment Management

**Environment-specific configurations:**
```yaml
# docker-compose.dev.yml
version: '3.8'
services:
  app:
    environment:
      - FLASK_ENV=development
      - FLASK_DEBUG=true
      - LOG_LEVEL=DEBUG
    volumes:
      - .:/app  # Mount source code for development

# docker-compose.prod.yml
version: '3.8'
services:
  app:
    environment:
      - FLASK_ENV=production
      - FLASK_DEBUG=false
      - LOG_LEVEL=WARNING
    restart: unless-stopped
```

### 3. CI/CD Pipeline

**GitHub Actions example:**
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-test.txt
      
      - name: Run tests
        run: pytest --cov=. --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to production
        run: |
          # Deployment script
          echo "Deploying to production..."
```

---

## Monitoring & Observability

### 1. Health Checks

```python
@app.route('/health')
def health_check():
    """Comprehensive health check endpoint"""
    health_status = {
        'status': 'healthy',
        'timestamp': datetime.utcnow().isoformat(),
        'service': 'Service Name',
        'version': '1.0.0',
        'checks': {}
    }
    
    # Database health
    try:
        db.session.execute('SELECT 1')
        health_status['checks']['database'] = 'healthy'
    except Exception as e:
        health_status['checks']['database'] = f'unhealthy: {str(e)}'
        health_status['status'] = 'unhealthy'
    
    # External service health
    try:
        external_service.ping()
        health_status['checks']['external_service'] = 'healthy'
    except Exception as e:
        health_status['checks']['external_service'] = f'unhealthy: {str(e)}'
        health_status['status'] = 'degraded'
    
    status_code = 200 if health_status['status'] == 'healthy' else 503
    return jsonify(health_status), status_code
```

### 2. Logging Strategy

```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }
        
        if record.exc_info:
            log_entry['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_entry)

# Configure logging
def setup_logging(log_level='INFO'):
    logging.basicConfig(
        level=getattr(logging, log_level),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.StreamHandler(),
            logging.FileHandler('app.log')
        ]
    )
    
    # Add JSON formatter for structured logging
    json_handler = logging.StreamHandler()
    json_handler.setFormatter(JSONFormatter())
    
    logger = logging.getLogger('structured')
    logger.addHandler(json_handler)
    logger.setLevel(logging.INFO)
```

### 3. Metrics Collection

```python
from prometheus_client import Counter, Histogram, Gauge, generate_latest

# Define metrics
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'HTTP request duration')
ACTIVE_JOBS = Gauge('active_jobs_total', 'Number of active jobs')

@app.before_request
def before_request():
    request.start_time = time.time()

@app.after_request
def after_request(response):
    duration = time.time() - request.start_time
    REQUEST_DURATION.observe(duration)
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.endpoint,
        status=response.status_code
    ).inc()
    return response

@app.route('/metrics')
def metrics():
    return generate_latest()
```

---

## Security

### 1. Authentication & Authorization

```python
from functools import wraps
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity

app.config['JWT_SECRET_KEY'] = os.getenv('JWT_SECRET_KEY')
jwt = JWTManager(app)

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')
        if not api_key or not validate_api_key(api_key):
            return jsonify({'error': 'Invalid API key'}), 401
        return f(*args, **kwargs)
    return decorated_function

def require_role(role):
    def decorator(f):
        @wraps(f)
        @jwt_required()
        def decorated_function(*args, **kwargs):
            current_user = get_jwt_identity()
            if not user_has_role(current_user, role):
                return jsonify({'error': 'Insufficient permissions'}), 403
            return f(*args, **kwargs)
        return decorated_function
    return decorator
```

### 2. Input Validation & Sanitization

```python
from marshmallow import Schema, fields, validate, ValidationError

class JobCreateSchema(Schema):
    name = fields.Str(
        required=True,
        validate=[
            validate.Length(min=1, max=255),
            validate.Regexp(r'^[a-zA-Z0-9_-]+$', error='Invalid characters in name')
        ]
    )
    config = fields.Dict(required=True)
    
    @validates('config')
    def validate_config(self, value):
        if not isinstance(value, dict):
            raise ValidationError('Config must be a dictionary')
        
        # Additional config validation
        required_keys = ['source', 'destination']
        for key in required_keys:
            if key not in value:
                raise ValidationError(f'Missing required config key: {key}')

def validate_and_sanitize_input(schema_class):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            try:
                schema = schema_class()
                validated_data = schema.load(request.get_json())
                return f(validated_data, *args, **kwargs)
            except ValidationError as err:
                return jsonify({'error': 'Validation failed', 'details': err.messages}), 400
        return decorated_function
    return decorator
```

### 3. Rate Limiting

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["1000 per day", "100 per hour"]
)

@app.route('/api/jobs', methods=['POST'])
@limiter.limit("10 per minute")
@require_api_key
def create_job():
    # Job creation logic
    pass
```

---

## Performance

### 1. Database Optimization

```python
# Use connection pooling
engine = create_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600
)

# Optimize queries with proper indexing
class Job(Base):
    __tablename__ = 'jobs'
    
    id = Column(String, primary_key=True)
    status = Column(String, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Strategic indexes
    __table_args__ = (
        Index('idx_job_status', 'status'),
        Index('idx_job_created_at', 'created_at'),
        Index('idx_job_status_created', 'status', 'created_at'),
    )

# Use pagination for large datasets
def get_jobs(page=1, per_page=20, status=None):
    query = Job.query
    
    if status:
        query = query.filter(Job.status == status)
    
    return query.paginate(
        page=page,
        per_page=per_page,
        error_out=False
    )
```

### 2. Caching Strategy

```python
from flask_caching import Cache

cache = Cache(app, config={
    'CACHE_TYPE': 'redis',
    'CACHE_REDIS_URL': os.getenv('REDIS_URL', 'redis://localhost:6379')
})

@cache.memoize(timeout=300)
def get_job_statistics():
    # Expensive calculation
    return calculate_statistics()

@cache.cached(timeout=60, key_prefix='job_list')
def get_job_list():
    return Job.query.all()

# Cache invalidation
def invalidate_job_cache(job_id):
    cache.delete(f'job_{job_id}')
    cache.delete('job_list')
```

### 3. Asynchronous Processing

```python
from celery import Celery

# Configure Celery
celery = Celery(
    app.import_name,
    broker=os.getenv('CELERY_BROKER_URL', 'redis://localhost:6379'),
    backend=os.getenv('CELERY_RESULT_BACKEND', 'redis://localhost:6379')
)

@celery.task
def process_job_async(job_id, config):
    """Asynchronous job processing"""
    try:
        # Update job status
        update_job_status(job_id, 'running')
        
        # Process job
        result = process_job(config)
        
        # Save result
        save_job_result(job_id, result)
        update_job_status(job_id, 'completed')
        
    except Exception as e:
        update_job_status(job_id, 'failed', error=str(e))
        raise

# API endpoint
@app.route('/api/jobs', methods=['POST'])
def create_job():
    job_id = generate_job_id()
    config = request.get_json()
    
    # Start async processing
    process_job_async.delay(job_id, config)
    
    return jsonify({'job_id': job_id, 'status': 'queued'}), 202
```

---

## Summary

### Key Principles

1. **Separation of Concerns**: Clear boundaries between layers
2. **Configuration-Driven**: All behavior configurable via environment
3. **Comprehensive Testing**: Unit, integration, and end-to-end tests
4. **Robust Error Handling**: Graceful failure handling and recovery
5. **Observability**: Comprehensive logging, metrics, and monitoring
6. **Security First**: Authentication, authorization, and input validation
7. **Performance Optimization**: Caching, async processing, and database optimization
8. **Documentation**: Clear, comprehensive, and up-to-date documentation

### Development Checklist

- [ ] Project structure follows layered architecture
- [ ] Configuration management implemented
- [ ] Database models with proper relationships and indexes
- [ ] API endpoints with proper HTTP status codes
- [ ] Input validation and sanitization
- [ ] Comprehensive error handling
- [ ] Unit and integration tests
- [ ] Health check endpoint
- [ ] Logging and monitoring
- [ ] Security measures (authentication, rate limiting)
- [ ] Docker configuration
- [ ] Comprehensive documentation
- [ ] Performance optimizations

### Maintenance Checklist

- [ ] Regular dependency updates
- [ ] Performance monitoring and optimization
- [ ] Security vulnerability scanning
- [ ] Log analysis and cleanup
- [ ] Database maintenance and optimization
- [ ] Documentation updates
- [ ] Test coverage analysis
- [ ] Backup and disaster recovery testing

This guideline provides a comprehensive framework for building robust, scalable, and maintainable services. Adapt these patterns to your specific technology stack and requirements while maintaining the core principles of good service design.