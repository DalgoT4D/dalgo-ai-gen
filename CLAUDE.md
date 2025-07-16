# Dalgo Platform - Development Guide

## Project Overview

### What is Dalgo?

Dalgo is an open-source, multi-tenant data platform specifically designed for the social sector and NGOs. Built and maintained by Project Tech4Dev, Dalgo simplifies data management for organizations that need flexible, scalable data infrastructure but lack technical resources.

**Mission**: To democratize data capabilities for social sector organizations by providing an integrated, user-friendly data platform that handles the entire data lifecycle from ingestion to visualization.

**Key Capabilities**:
- **Multi-tenant Architecture**: Supports multiple organizations with shared infrastructure
- **Data Warehouse Support**: Compatible with PostgreSQL and BigQuery
- **Flexible Data Sources**: Integrates with 100+ data sources via Airbyte
- **SQL-based Transformations**: Uses dbt for data transformation workflows
- **Workflow Orchestration**: Manages complex data pipelines through Prefect
- **User Management**: Role-based access control and organization management
- **Commercial Hosting**: Available at [dashboard.dalgo.org](https://dashboard.dalgo.org)

**Target Audience**: NGOs, social sector organizations, and mission-driven entities needing robust data management without the complexity of building custom data infrastructure.

### Platform Architecture

The Dalgo platform consists of 5 main repositories plus 2 external services:

### Core Repositories
- **DDP_backend**: Django backend API server
- **webapp**: Next.js frontend (legacy)
- **webapp_v2**: Next.js frontend (modern)
- **prefect-proxy**: FastAPI proxy for Prefect orchestration
- **prefect-airbyte**: Python library for Prefect-Airbyte integration

### External Services
- **Prefect**: Workflow orchestration (port 4200)
- **Airbyte**: Data ingestion (port 8000)

## Service Management (PM2)

The platform uses PM2 for service orchestration. All services are defined in `dalgo_dev.config.js`.

### Global Commands
```bash
# Start all services
pm2 start dalgo_dev.config.js

# Stop all services
pm2 stop dalgo_dev.config.js

# Restart all services
pm2 restart dalgo_dev.config.js

# Monitor services
pm2 monit

# View logs for all services
pm2 logs

# View logs for specific service
pm2 logs [service-name]

# List all services
pm2 list
```

## Individual Service Management

### DDP_backend (Django Backend)

**Technology Stack**: Django 4.2, Python 3.10+, PostgreSQL, Celery, Redis
**Purpose**: Management backend with API endpoints for NGO onboarding, user management, and integration with Airbyte/Prefect
**Port**: 8002

#### PM2 Services
- `django-backend-asgi`: Main Django ASGI server
- `django-celery-worker`: Celery worker for background tasks
- `django-celery-beat`: Celery beat scheduler

#### Setup Commands
```bash
cd DDP_backend

# Install dependencies (virtual environment pre-configured)
uv sync

# Run database migrations
python3 manage.py migrate

# Load seed data
python3 manage.py loaddata seed/*.json

# Create system user
python3 manage.py create-system-orguser

# Create first org and user
python3 manage.py createorganduser <Org Name> <Email address> --role super-admin
```

#### Manual Run Commands
```bash
cd DDP_backend

# Run Django server
uvicorn ddpui.asgi:application --workers 4 --host 0.0.0.0 --port 8002 --timeout-keep-alive 60 --reload

# Run Celery worker
celery -A ddpui worker -n ddpui

# Run Celery beat
celery -A ddpui beat
```

#### Testing & Code Quality
```bash
cd DDP_backend

# Run tests
pytest ddpui/tests

# Run tests with coverage
pytest --coverage

# Code formatting
black .

# Linting
pylint ddpui/

# Pre-commit hooks
pre-commit run --all-files
```

#### Key Features
- REST API endpoints for frontend communication
- NGO client onboarding
- User management and authentication
- Airbyte workspace configuration
- Prefect job orchestration
- DBT integration

---

### webapp (Frontend v1)

**Technology Stack**: Next.js 14, React 18, TypeScript, Material-UI
**Purpose**: Management frontend application
**Port**: 3000 (default)

#### PM2 Service
- `webapp`: Next.js development server

#### Setup Commands
```bash
cd webapp

# Install dependencies
yarn install
```

#### Manual Run Commands
```bash
cd webapp

# Development server
yarn dev

# Production build
yarn build

# Production server
yarn start
```

#### Testing & Code Quality
```bash
cd webapp

# Run Jest tests
yarn test

# Run Jest tests with coverage
yarn test:coverage

# Run Cypress E2E tests
yarn cy:run

# Open Cypress UI
yarn cypress

# Linting
yarn lint

# Code formatting
yarn format:write
```

#### Key Features
- NGO management interface
- Data pipeline configuration
- User authentication and management
- Integration with Django backend

---

### webapp_v2 (Frontend v2)

**Technology Stack**: Next.js 15, React 19, TypeScript, Tailwind CSS, Zustand, SWR
**Purpose**: Modern dashboard with data visualization capabilities
**Port**: 3001

#### PM2 Service
- `v2-webapp`: Next.js development server

#### Setup Commands
```bash
cd webapp_v2

# Install dependencies
npm install
```

#### Manual Run Commands
```bash
cd webapp_v2

# Development server
npm run dev

# Production build
npm run build

# Production server
npm start
```

#### Testing & Code Quality
```bash
cd webapp_v2

# Run tests
npm test

# Run tests in watch mode
npm run test:watch

# Run tests with coverage
npm run test:coverage

# Run tests in CI mode
npm run test:ci

# Linting
npm run lint

# Code formatting
npm run format:write
```

#### Key Features
- Modern dashboard interface
- Data visualization with ECharts, Nivo, Recharts
- Real-time alerts and notifications
- Responsive design
- Authentication and protected routes

---

### prefect-proxy

**Technology Stack**: FastAPI, Python 3.10+, Prefect 3.x
**Purpose**: Proxy service for Prefect orchestration, bridging async Prefect with Django
**Port**: 8085

#### PM2 Services
- `prefect-proxy`: FastAPI proxy server
- `prefect-server`: Prefect server
- `prefect-worker-ddp-1`: Prefect worker for ddp queue
- `prefect-worker-ddp-2`: Prefect worker for ddp queue
- `prefect-worker-manual-dbt`: Prefect worker for manual-dbt queue

#### Setup Commands
```bash
cd prefect-proxy

# Install dependencies (virtual environment pre-configured)
uv sync

# Create .env from template
cp .env.template .env
```

#### Manual Run Commands
```bash
cd prefect-proxy

# Run proxy server
uvicorn proxy.main:app --reload --port 8085

# Run Prefect server
uv run prefect server start

# Run Prefect workers
uv run prefect worker start -q ddp --pool dev_dalgo_work_pool --limit 1
uv run prefect worker start -q manual-dbt --pool dev_dalgo_work_pool --limit 1
```

#### Testing & Code Quality
```bash
cd prefect-proxy

# Run tests
pytest

# Run tests with coverage
pytest --cov

# Code formatting
black .

# Linting
pylint proxy/

# Pre-commit hooks
pre-commit run --all-files
```

#### Key Features
- FastAPI endpoints for Prefect integration
- Async workflow management
- Multiple worker queues (ddp, manual-dbt)
- Integration with Google Cloud services

---

### prefect-airbyte

**Technology Stack**: Python 3.7+, Prefect collections
**Purpose**: Prefect-Airbyte integration library for data pipeline orchestration
**Usage**: Library/dependency used by prefect-proxy

#### Setup Commands
```bash
cd prefect-airbyte

# Install in development mode (virtual environment pre-configured)
pip install -e ".[dev]"
```

#### Testing & Code Quality
```bash
cd prefect-airbyte

# Run tests
pytest

# Run tests with coverage
pytest --cov

# Pre-commit hooks
pre-commit run --all-files
```

#### Key Features
- Prefect tasks for Airbyte operations
- Connection sync management
- Flow orchestration utilities
- Integration with Airbyte API

## Port Assignments

| Service | Port | Purpose |
|---------|------|---------|
| Django Backend | 8002 | Main API server |
| Prefect Proxy | 8085 | Prefect orchestration proxy |
| Prefect Server | 4200 | External workflow engine |
| Airbyte | 8000 | External data ingestion |
| Webapp | 3000 | Frontend v1 |
| Webapp v2 | 3001 | Frontend v2 |

## Service Interactions & Data Flow

Understanding how services interact is crucial for effective development and debugging. Here's how the Dalgo platform components communicate:

### High-Level Data Flow

```
[Data Sources] → [Airbyte:8000] → [Data Warehouse] → [dbt via Prefect] → [Transformed Data] → [Frontend Visualization]
```

### Detailed Service Interactions

#### 1. Frontend → Django Backend (API Communication)
- **webapp** (port 3000) and **webapp_v2** (port 3001) make HTTP requests to **DDP_backend** (port 8002)
- **Authentication**: JWT tokens for user authentication
- **API Endpoints**: REST API for user management, organization setup, pipeline configuration
- **WebSocket**: Real-time updates for pipeline status and notifications

#### 2. Django Backend → Prefect Proxy (Orchestration)
- **DDP_backend** sends workflow requests to **prefect-proxy** (port 8085)
- **Purpose**: Trigger dbt runs, manage data pipeline schedules, monitor job status
- **Communication**: HTTP requests to FastAPI endpoints
- **Queue Management**: Jobs are queued in `ddp` and `manual-dbt` queues

#### 3. Prefect Proxy → Prefect Server (Workflow Management)
- **prefect-proxy** communicates with **Prefect Server** (port 4200)
- **Purpose**: Submit flows, monitor execution, manage deployments
- **Communication**: Prefect SDK calls and HTTP API requests
- **Worker Coordination**: Multiple workers poll for jobs from different queues

#### 4. Django Backend → Airbyte (Data Ingestion)
- **DDP_backend** configures and monitors **Airbyte** (port 8000)
- **Purpose**: Set up data sources, create connections, trigger syncs
- **Communication**: HTTP requests to Airbyte API
- **Configuration**: Workspace, source, destination, and connection management

#### 5. Prefect Workers → External Services
- **prefect-worker-ddp-1/2**: Execute dbt transformations and data quality checks
- **prefect-worker-manual-dbt**: Handle manual dbt runs triggered by users
- **Integration**: Uses **prefect-airbyte** library for Airbyte operations within workflows

#### 6. Celery Background Tasks
- **django-celery-worker**: Processes background tasks for DDP_backend
- **django-celery-beat**: Schedules periodic tasks
- **Purpose**: Email notifications, data processing, cleanup tasks
- **Communication**: Redis as message broker

### Data Persistence

#### PostgreSQL Database
- **Primary Database**: Stores user data, organizations, pipeline configurations
- **Connected to**: DDP_backend for application data
- **Managed by**: Django ORM with migrations

#### Data Warehouses
- **PostgreSQL**: For smaller organizations or development
- **BigQuery**: For larger organizations with complex analytics needs
- **Purpose**: Store ingested and transformed data
- **Access**: Through dbt models and direct SQL queries

### Security & Authentication

#### JWT Authentication Flow
1. User logs in via **webapp/webapp_v2**
2. **DDP_backend** validates credentials and issues JWT token
3. Token is included in subsequent API requests
4. **DDP_backend** validates tokens for protected endpoints

#### Service-to-Service Authentication
- **Prefect Proxy**: Uses API keys for Prefect server communication
- **Airbyte**: Uses basic auth or API tokens
- **Internal Services**: Shared secrets or environment-based authentication

### Error Handling & Monitoring

#### Webhook Notifications
- **Prefect** sends webhook notifications to **DDP_backend** on job completion/failure
- **Endpoint**: `POST /webhooks/v1/notification/`
- **Purpose**: Update job status, trigger alerts, clean up resources

#### Logging & Monitoring
- **Centralized Logging**: Each service logs to its own directory
- **PM2 Monitoring**: `pm2 monit` provides real-time service monitoring
- **Error Tracking**: Sentry integration for error monitoring

### Development Communication Patterns

#### Frontend Development
- **API Calls**: Use SWR or React Query for data fetching
- **State Management**: Zustand for webapp_v2, Context API for webapp
- **Real-time Updates**: WebSocket connections for live status updates

#### Backend Development
- **Async Operations**: Celery for background tasks
- **API Design**: Django REST Framework with proper serialization
- **Database**: Django ORM with PostgreSQL

#### Integration Testing
- **End-to-End**: Test complete data flow from ingestion to visualization
- **Service Mocking**: Mock external services (Airbyte, Prefect) for unit tests
- **Database**: Use test databases for isolation

## Development Workflow

1. **Start All Services**: `pm2 start dalgo_dev.config.js`
2. **Monitor Services**: `pm2 monit`
3. **Check Logs**: `pm2 logs [service-name]`
4. **Make Changes**: Edit code in respective repositories
5. **Test Changes**: Run service-specific tests
6. **Restart Service**: `pm2 restart [service-name]`

## Environment Setup

Each Python service has its virtual environment pre-configured:
- `DDP_backend/.venv`
- `prefect-proxy/.venv`
- `prefect-airbyte/.venv`

## Key Commands Summary

```bash
# Service Management
pm2 start dalgo_dev.config.js     # Start all services
pm2 stop dalgo_dev.config.js      # Stop all services
pm2 restart dalgo_dev.config.js   # Restart all services
pm2 monit                         # Monitor services

# Backend Development
cd DDP_backend && pytest ddpui/tests                    # Test backend
cd DDP_backend && python manage.py migrate              # Database migrations
cd DDP_backend && python manage.py loaddata seed/*.json # Load seed data

# Frontend Development
cd webapp && yarn test             # Test frontend v1
cd webapp_v2 && npm test           # Test frontend v2

# Proxy Development
cd prefect-proxy && pytest        # Test proxy
cd prefect-proxy && uv sync       # Install dependencies

# Code Quality
black .                           # Format Python code
pylint [directory]                # Lint Python code
yarn lint                         # Lint JavaScript/TypeScript
npm run lint                      # Lint JavaScript/TypeScript
```

This development guide ensures Claude can effectively work with the Dalgo platform by understanding the service architecture, development workflows, and available commands for each repository.