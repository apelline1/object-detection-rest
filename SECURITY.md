# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 1.0.0   | :white_check_mark: |
| < 1.0.0 | :x:                |

## Security Updates (v1.0.0)

### Enhanced Security Features

This release includes important security improvements and best practices:

#### Dependency Management
- **Pinned Versions**: All dependencies now have explicit version ranges
- **Updated Packages**: Using latest stable versions with security patches
  - Flask >= 3.1.0 (includes security fixes)
  - Gunicorn >= 23.0.0 (latest stable)
  - TensorFlow >= 2.18.0 (includes security updates)
  - Pillow >= 11.0.0 (critical security fixes)
  - Werkzeug >= 3.1.3 (security patches)
  - NumPy >= 2.2.1 (latest stable)
  - Matplotlib >= 3.10.0 (updated)

#### Security Audit Results
- **Python Packages**: No known vulnerabilities detected (pip-audit)
- **Dependency Tree**: Clean security scan
- **Best Practices**: Version pinning implemented

### Input Validation Improvements

#### Image Size Limits
- Maximum image size: 10MB (prevents DoS via memory exhaustion)
- Base64 validation with proper error handling
- Automatic padding correction

#### Request Validation
- JSON schema validation
- Required field checking
- Type validation for all inputs
- Sanitized error messages (no stack traces to client in production)

### Error Handling

#### TensorFlow-Specific Errors
- `ResourceExhaustedError`: Out of memory handling
- `InvalidArgumentError`: Invalid image data handling
- Graceful degradation for model loading failures

#### API Error Responses
- No sensitive information in error messages
- Proper HTTP status codes
- Detailed logging server-side only

## Security Best Practices for Deployment

### Container Security

#### Base Image
```dockerfile
# Use official Python images with security updates
FROM registry.access.redhat.com/ubi8/python-311

# Run as non-root user
USER 1001
```

#### Image Scanning
```bash
# Scan container images before deployment
podman scan quay.io/apelline/object-detection-rest:latest

# Or use trivy
trivy image quay.io/apelline/object-detection-rest:latest
```

### OpenShift/Kubernetes Security

#### Resource Limits
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "2Gi"
    cpu: "1000m"
```

#### Security Context
```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

#### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: object-detection-rest-netpol
spec:
  podSelector:
    matchLabels:
      app: object-detection-rest
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: object-detection-app  # Only allow frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - {} # Allow all egress (adjust as needed)
```

### Application Configuration

#### Gunicorn Security Settings

```python
# gunicorn_config.py
workers = 1  # Single worker for memory management
threads = 2  # Limited concurrency

# Security headers
timeout = 120  # Prevent hung requests
keepalive = 5  # Prevent connection exhaustion

# Proxy configuration
forwarded_allow_ips = '*'  # Adjust for your proxy IP
secure_scheme_headers = {'X-Forwarded-Proto': 'https'}
```

#### Environment Variables
```bash
# Never commit these to Git
export SECRET_KEY='<generate-random-secret>'
export FLASK_ENV='production'  # Disable debug mode
export GUNICORN_TIMEOUT='120'

# Use secrets management
oc create secret generic app-secrets \
  --from-literal=secret-key='<random-key>'
```

### TLS/HTTPS Configuration

#### OpenShift Routes
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: object-detection-rest
spec:
  to:
    kind: Service
    name: object-detection-rest
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

#### Certificate Management
- Use cert-manager for automatic certificate rotation
- Enable HSTS (HTTP Strict Transport Security)
- Configure secure cipher suites

### Input Sanitization

#### API Endpoints
```python
# Already implemented in prediction.py
def validate_image_input(body):
    """Validate and sanitize image input"""
    if not isinstance(body, dict):
        raise ValueError("Request must be JSON object")
    
    if 'image' not in body:
        raise ValueError("Missing required field: image")
    
    image_data = body['image']
    if not isinstance(image_data, str):
        raise ValueError("Image must be base64 string")
    
    # Size validation
    if len(image_data) > 15000000:  # ~10MB base64
        raise ValueError("Image too large")
    
    return image_data
```

### Authentication & Authorization

#### Add JWT Authentication (Recommended)
```python
from flask_jwt_extended import JWTManager, jwt_required

# Add to wsgi.py
app.config['JWT_SECRET_KEY'] = os.environ.get('JWT_SECRET_KEY')
jwt = JWTManager(app)

@app.route('/predictions', methods=['POST'])
@jwt_required()  # Require valid JWT token
def create_prediction():
    # ... existing code
```

#### API Key Authentication (Alternative)
```python
from functools import wraps
from flask import request, jsonify

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')
        if not api_key or api_key != os.environ.get('API_KEY'):
            return jsonify({'error': 'Invalid API key'}), 401
        return f(*args, **kwargs)
    return decorated_function

@app.route('/predictions', methods=['POST'])
@require_api_key
def create_prediction():
    # ... existing code
```

### Rate Limiting

#### Using Flask-Limiter
```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["100 per hour", "10 per minute"]
)

@app.route('/predictions', methods=['POST'])
@limiter.limit("5 per minute")  # Limit expensive operations
def create_prediction():
    # ... existing code
```

### Monitoring & Auditing

#### Logging Best Practices
```python
import logging
import json

# Structured logging
logger = logging.getLogger(__name__)

def log_security_event(event_type, details):
    """Log security-relevant events"""
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'event_type': event_type,
        'details': details,
        'user_ip': request.remote_addr
    }
    logger.warning(json.dumps(log_entry))

# Usage
log_security_event('invalid_image', {'error': 'oversized_image'})
```

#### Health Monitoring
- Use `/status` endpoint for liveness probes
- Implement readiness probes
- Monitor memory usage (TensorFlow can be memory-intensive)
- Set up alerts for error rate spikes

### Dependency Management

#### Regular Updates
```bash
# Check for vulnerabilities weekly
pip-audit -r requirements.txt

# Update dependencies (test thoroughly)
pip list --outdated
pip install --upgrade <package-name>
```

#### Automated Scanning
- Configure Dependabot/Renovate for automated PR updates
- Run security scans in CI/CD pipeline
- Use container image scanning in registry

### Data Privacy

#### Image Handling
- **Temporary Storage Only**: Images processed in memory, not persisted
- **No Logging of Image Data**: Never log base64 image data
- **Secure Deletion**: Ensure TensorFlow tensors are properly deallocated

#### GDPR Compliance
- No personal data stored
- Images processed and discarded immediately
- Implement data processing agreements if needed
- Provide data processing documentation

### Incident Response

#### Security Incident Procedure
1. **Detection**: Monitor logs for suspicious activity
2. **Assessment**: Determine severity and impact
3. **Containment**: Isolate affected services
4. **Remediation**: Apply patches, rotate secrets
5. **Review**: Post-incident analysis and documentation

#### Contact
- Security issues: [security contact to be configured]
- Response time: 48 hours for critical issues

## Security Checklist for Production

- [ ] All dependencies use pinned versions
- [ ] Container image scanned for vulnerabilities
- [ ] Running as non-root user
- [ ] Resource limits configured
- [ ] TLS/HTTPS enabled
- [ ] Authentication implemented
- [ ] Rate limiting configured
- [ ] Network policies in place
- [ ] Security context constraints applied
- [ ] Secrets managed via Kubernetes/OpenShift secrets
- [ ] Logging configured (no sensitive data)
- [ ] Health check endpoints configured
- [ ] Monitoring and alerting set up
- [ ] Backup and recovery plan documented
- [ ] Security scanning in CI/CD pipeline

## Known Limitations

### TensorFlow Security
- TensorFlow models can be vulnerable to adversarial attacks
- No model input sanitization beyond size limits
- Consider implementing adversarial detection if critical

### Gunicorn Configuration
- `forwarded_allow_ips = '*'` trusts all proxies
- Recommendation: Configure specific proxy IPs in production

### Model Integrity
- No signature verification for loaded models
- Ensure model files are from trusted sources
- Consider implementing model checksum validation

## Reporting a Vulnerability

If you discover a security vulnerability, please:

1. **Do NOT** open a public GitHub issue
2. Email: [security contact needed]
3. Include:
   - Vulnerability description
   - Steps to reproduce
   - Impact assessment
   - Suggested fix (if available)

### Response Timeline
- **Acknowledgment**: 48 hours
- **Initial Assessment**: 7 days
- **Fix Implementation**: 30 days (critical), 90 days (high)
- **Disclosure**: After fix is deployed

## Compliance

### Standards
- OWASP Top 10 compliance
- CIS Kubernetes Benchmark
- OpenShift security best practices
- Container security standards

### Certifications
- [To be configured based on requirements]

---

**Last Updated**: 2026-04-22  
**Security Version**: 1.0  
**Next Review**: 2026-07-22
