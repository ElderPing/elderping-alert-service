# ElderPing Alert Service

## Overview

The Alert Service is a Node.js-based microservice responsible for handling emergency alerts and notifications in the ElderPing platform. It manages alert creation, delivery, tracking, and escalation for elderly users in emergency situations.

## Technology Stack

- **Runtime**: Node.js 18
- **Framework**: Express.js
- **Database**: PostgreSQL
- **Container**: Docker (multi-stage build with Alpine Linux)
- **Security**: Non-root user execution (USER node)

## Features

- Emergency alert creation and management
- Multi-channel alert delivery (SMS, email, push notifications)
- Alert escalation to family members
- Alert tracking and status updates
- Alert history and analytics
- Integration with external emergency services
- Health check endpoint for monitoring

## API Endpoints

### Alerts
- `POST /api/alert/create` - Create emergency alert
- `GET /api/alert/:userId` - Get alerts for user
- `GET /api/alert/:id` - Get specific alert
- `PUT /api/alert/:id` - Update alert status
- `DELETE /api/alert/:id` - Delete alert
- `POST /api/alert/:id/escalate` - Escalate alert to family

### Alert Notifications
- `POST /api/alert/notify` - Send alert notification
- `GET /api/alert/:id/notifications` - Get notification history

### Health
- `GET /health` - Health check endpoint

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `DB_HOST` | PostgreSQL database host | Yes |
| `DB_PORT` | PostgreSQL database port | Yes |
| `DB_USER` | PostgreSQL username | Yes |
| `DB_PASSWORD` | PostgreSQL password | Yes |
| `DB_NAME` | Database name (alert_db) | Yes |
| `PORT` | Service port (default: 3000) | No |
| `SMS_API_KEY` | SMS service API key (optional) | No |
| `EMAIL_SERVICE_KEY` | Email service API key (optional) | No |

## Database Schema

### Alerts Table
```sql
CREATE TABLE alerts (
  id          SERIAL PRIMARY KEY,
  user_id     INT NOT NULL,
  type        VARCHAR(50) NOT NULL, -- emergency, medical, fall, etc.
  severity    VARCHAR(20) NOT NULL, -- low, medium, high, critical
  status      VARCHAR(20) DEFAULT 'active',
  location    VARCHAR(200),
  description TEXT,
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  resolved_at TIMESTAMP
);
```

### Alert Notifications Table
```sql
CREATE TABLE alert_notifications (
  id          SERIAL PRIMARY KEY,
  alert_id    INT NOT NULL REFERENCES alerts(id) ON DELETE CASCADE,
  channel     VARCHAR(50) NOT NULL, -- sms, email, push
  recipient   VARCHAR(100) NOT NULL,
  status      VARCHAR(20) DEFAULT 'pending',
  sent_at     TIMESTAMP,
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Alert Escalations Table
```sql
CREATE TABLE alert_escalations (
  id          SERIAL PRIMARY KEY,
  alert_id    INT NOT NULL REFERENCES alerts(id) ON DELETE CASCADE,
  escalated_to INT NOT NULL,
  escalated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  status      VARCHAR(20) DEFAULT 'pending'
);
```

## Docker Image

- **Repository**: `arunnsimon/elderpinq-alert-service`
- **Tags**: 
  - `dev-latest` - Development builds from develop branch
  - `prod-latest` - Production builds from main branch
  - `<version>` - Release tags

## CI/CD Pipeline

The service uses GitHub Actions for continuous integration and deployment:

1. **Security Scanning**
   - SAST (Static Application Security Testing)
   - SCA (Software Composition Analysis)
   - Trivy vulnerability scanning

2. **Docker Build & Publish**
   - Multi-stage Docker build
   - Push to Docker Hub
   - Tagged based on branch (dev-latest/prod-latest)

3. **GitOps Deployment**
   - Updates Helm chart image tag in elderping-k8s-charts
   - ArgoCD automatically syncs changes

## Kubernetes Deployment

### Helm Chart
Located in `elderping-k8s-charts/microservices/alert-service/`

**Resources:**
- Deployment with 2 replicas
- Service (ClusterIP on port 3000)
- HorizontalPodAutoscaler (1-3 replicas, 80% CPU target)

**Configuration:**
- Namespace: elderping-dev (dev) / elderping-prod (prod)
- Resource requests: 50m CPU, 128Mi memory
- Resource limits: 300m CPU, 256Mi memory
- Liveness/Readiness probes on /health endpoint

## Security Features

- **Non-root container**: Runs as `node` user (not root)
- **Environment variables**: Sensitive data via Kubernetes Secrets
- **Network policies**: Restricts ingress/egress traffic (when enabled)
- **API keys**: Stored securely via Kubernetes Secrets

## Development

### Local Setup
```bash
# Install dependencies
npm install

# Set environment variables
cp .env.example .env
# Edit .env with your values

# Run development server
npm run dev

# Run tests
npm test

# Build for production
npm run build
```

### Docker Build
```bash
# Build image
docker build -t elderping-alert-service .

# Run container
docker run -p 3000:3000 --env-file .env elderping-alert-service
```

## Monitoring

- **Health Check**: `GET /health` returns service status
- **Metrics**: Exposed for Prometheus scraping
- **Logs**: Collected by Loki
- **Dashboards**: Grafana dashboards for monitoring
- **Alert Metrics**: Alert volume, response time, delivery success rate

## Troubleshooting

### Common Issues

**Database Connection Failed**
- Verify DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME
- Check PostgreSQL is accessible from the pod
- Verify network policies allow database access

**Alert Delivery Failed**
- Verify SMS/Email API keys are configured
- Check notification service status
- Verify recipient contact information

**Container Not Starting**
- Check pod logs: `kubectl logs <pod-name> -n elderping-dev`
- Verify resource limits are sufficient
- Check liveness probe configuration

## Contributing

1. Create feature branch from develop
2. Make changes and test locally
3. Commit with descriptive message
4. Push to feature branch
5. Create pull request to develop

## License

Proprietary - ElderPing Platform
