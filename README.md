# Object Detection REST API

A Flask-based REST API for real-time object detection using TensorFlow and the OpenImages v4 SSD MobileNet v2 model. Designed for deployment on OpenShift as a Source-to-Image (s2i) application.

## Quick Start

### Prerequisites
- Python 3.11+
- TensorFlow 2.18.0+
- OpenShift CLI (`oc`) for deployment

### Local Development

```bash
# Clone the repository
git clone https://github.com/apelline1/object-detection-rest_original.git
cd object-detection-rest_original

# Install dependencies
pip install -r requirements.txt

# Run Flask server
FLASK_APP=wsgi.py flask run

# Test the API
curl -X POST http://localhost:5000/predictions \
  -H "Content-Type: application/json" \
  -d '{"image": "<base64-encoded-image>"}'
```

### OpenShift Deployment

```bash
# Create new project
oc new-project object-detection

# Deploy using s2i
oc new-app python:3.11~https://github.com/apelline1/object-detection-rest_original

# Expose service
oc expose svc/object-detection-rest_original

# Or use the deployment manifest
oc apply -f deployment.yaml
```

## API Endpoints

### `/predictions` - Object Detection
**POST** - Detect objects in a base64-encoded image
```json
{
  "image": "base64_encoded_image_string"
}
```

### `/api/images` - Frontend Integration
**POST** - Enhanced endpoint with detailed logging and error handling

### `/status` or `/` - Health Check
**GET** - Check service health

### `/test` - Backend Verification
**GET/POST** - Verify backend is working

## Features

- ✅ Real-time object detection using TensorFlow
- ✅ Support for JPEG, PNG, GIF, BMP formats
- ✅ Automatic base64 decoding with data URL handling
- ✅ 10MB image size limit
- ✅ Comprehensive error handling
- ✅ Health check endpoints
- ✅ Extensive request logging
- ✅ Model preloading for faster responses
- ✅ OpenShift/Kubernetes ready

## Documentation

For comprehensive documentation, see:
- [Technical Documentation](DOCUMENTATION.md) - Architecture, API details, deployment
- [Security Policy](SECURITY.md) - Security best practices, deployment checklist

## Development Workflow

The repository includes Jupyter notebooks for development:

1. **0_sandbox.ipynb** - Getting started
2. **1_explore.ipynb** - Data exploration
3. **2_predict.ipynb** - Test predictions
4. **3_run_flask.ipynb** - Run Flask locally
5. **4_test_flask.ipynb** - Test API endpoints

## Integration

This API integrates with the [object-detection-app](https://github.com/apelline1/object-detection-app) frontend application.

### Environment Variable Configuration

Set this in your frontend deployment:
```bash
OBJECT_DETECTION_URL=http://object-detection-rest:8080/predictions
```

## Configuration

### Environment Variables

```bash
GUNICORN_PROCESSES=1          # Worker processes (default: 1)
GUNICORN_THREADS=2            # Threads per worker (default: 2)
GUNICORN_TIMEOUT=120          # Request timeout in seconds (default: 120)
GUNICORN_BIND=0.0.0.0:8080   # Bind address (default: 0.0.0.0:8080)
```

### Resource Requirements

**Minimum**:
- Memory: 512Mi
- CPU: 250m

**Recommended** (for production):
- Memory: 2Gi
- CPU: 1000m

## Model Information

- **Model**: OpenImages v4 SSD MobileNet v2
- **Location**: `models/openimages_v4_ssd_mobilenet_v2_1/`
- **Format**: TensorFlow SavedModel
- **Detection Limit**: Top 10 detections per image

## Security

- Input validation on all endpoints
- Size limits to prevent DoS attacks
- Pinned dependency versions
- No known security vulnerabilities (pip-audit clean)
- See [SECURITY.md](SECURITY.md) for details

## License

This project is licensed under the GPLv3 License - see the [LICENSE](LICENSE) file for details.

## Repository

- **URL**: https://github.com/apelline1/object-detection-rest_original
- **Version**: 1.0.0
- **Owner**: apelline1

## Support

For issues and questions:
- Open an issue on GitHub
- See [DOCUMENTATION.md](DOCUMENTATION.md) for troubleshooting

## Recent Updates

### Version 1.0.0 (2026-04-22)
- ✅ Pinned dependency versions for security
- ✅ Comprehensive technical documentation
- ✅ Security policy and best practices guide
- ✅ Enhanced error handling
- ✅ Multi-format image support
- ✅ Production-ready deployment configuration

---

**Maintained by**: apelline1  
**Last Updated**: 2026-04-22
