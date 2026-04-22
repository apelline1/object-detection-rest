# Object Detection REST API - Technical Documentation

## Overview
The Object Detection REST API is a Flask-based microservice that provides real-time object detection capabilities using TensorFlow and the OpenImages v4 SSD MobileNet v2 model. It's designed to run on OpenShift as an s2i (Source-to-Image) application and integrates with the object-detection-app frontend.

## Architecture

### Technology Stack
- **Web Framework**: Flask 3.1.0+
- **WSGI Server**: Gunicorn 23.0.0+
- **Machine Learning**: TensorFlow 2.18.0+
- **Model**: OpenImages v4 SSD MobileNet v2
- **Image Processing**: Pillow 11.0.0+, NumPy 2.2.1+
- **Visualization**: Matplotlib 3.10.0+
- **Container Platform**: OpenShift (s2i deployment)

### Application Structure
```
object-detection-rest_original/
├── README.md                   # Project documentation
├── LICENSE                     # GPLv3 License
├── requirements.txt            # Python dependencies with version pinning
├── wsgi.py                     # Flask application entry point
├── prediction.py               # Core prediction logic and model loading
├── gunicorn_config.py          # Gunicorn WSGI server configuration
├── deployment.yaml             # Kubernetes/OpenShift deployment manifest
├── .s2i/                       # Source-to-Image configuration
│   └── environment             # s2i environment variables
├── models/                     # TensorFlow model directory
│   └── openimages_v4_ssd_mobilenet_v2_1/
├── sample-requests/            # Sample API request examples
├── blank.jpeg                  # Blank image for model preloading
├── 0_sandbox.ipynb            # Getting started notebook
├── 1_explore.ipynb            # Data exploration notebook
├── 2_predict.ipynb            # Prediction testing notebook
├── 3_run_flask.ipynb          # Local Flask testing notebook
└── 4_test_flask.ipynb         # Flask endpoint testing notebook
```

## Core Features

### 1. Object Detection API

#### `/predictions` - Primary prediction endpoint
- **Method**: POST
- **Content-Type**: application/json
- **Request Body**:
  ```json
  {
    "image": "base64_encoded_image_string"
  }
  ```
- **Response**:
  ```json
  {
    "detections": [
      {
        "box": {
          "yMin": 0.123,
          "xMin": 0.456,
          "yMax": 0.789,
          "xMax": 0.012
        },
        "class": "dog",
        "label": "dog",
        "score": 0.95
      }
    ]
  }
  ```

#### `/api/images` - Frontend integration endpoint
- **Method**: POST
- **Content-Type**: application/json
- **Features**:
  - Extensive request logging for debugging
  - Image size validation (10MB limit)
  - Multiple image format support (JPEG, PNG, GIF, BMP)
  - Data URL prefix handling
  - Detailed error messages

#### `/status` or `/` - Health check endpoint
- **Method**: GET
- **Response**: 
  ```json
  {
    "status": "ok",
    "message": "Service is healthy"
  }
  ```

#### `/test` - Backend verification endpoint
- **Method**: GET or POST
- **Response**:
  ```json
  {
    "status": "ok",
    "message": "Backend is working",
    "endpoint": "/test"
  }
  ```

### 2. Image Processing Features

- **Format Support**: JPEG, PNG, GIF, BMP
- **Base64 Decoding**: Automatic handling of data URL prefixes
- **Size Limits**: 10MB maximum image size
- **Auto-padding**: Automatic base64 padding correction
- **Error Handling**: Comprehensive error handling for invalid formats

### 3. Model Features

- **Model**: OpenImages v4 SSD MobileNet v2
- **Preloading**: Automatic model warmup on startup
- **Detection Limit**: Returns top 10 detections
- **Score Threshold**: Configurable detection confidence
- **Memory Management**: TensorFlow ResourceExhausted error handling

## API Endpoints Detail

### Prediction Flow

1. **Image Reception**: Receive base64-encoded image
2. **Validation**: Check image size and format
3. **Decoding**: Decode base64 to bytes
4. **TensorFlow Processing**: 
   - Decode image to tensor
   - Convert to float32
   - Add batch dimension
   - Run through model
5. **Post-processing**: Clean and format detections
6. **Response**: Return JSON with bounding boxes and labels

### Error Handling

The API provides detailed error responses for:
- **400 Bad Request**: Invalid JSON, missing image field, empty image
- **500 Internal Server Error**: 
  - TensorFlow errors (InvalidArgument, ResourceExhausted)
  - Model loading failures
  - Image decoding failures
  - Out of memory errors

## Configuration

### Environment Variables

```bash
# Gunicorn Configuration
GUNICORN_PROCESSES=1          # Number of worker processes
GUNICORN_THREADS=2            # Threads per worker
GUNICORN_TIMEOUT=120          # Request timeout in seconds
GUNICORN_BIND=0.0.0.0:8080   # Bind address and port

# Application Configuration
APP_CONFIG=gunicorn_config.py # Gunicorn config file path
```

### Gunicorn Settings

- **Workers**: 1 (default) - single process to manage TensorFlow memory
- **Threads**: 2 (default) - thread-based concurrency
- **Timeout**: 120 seconds - for large image processing
- **Bind**: 0.0.0.0:8080 - listen on all interfaces
- **Forwarded IPs**: Allow all (for reverse proxy)
- **Secure Scheme**: X-Forwarded-Proto header support

## Deployment

### OpenShift s2i Deployment

1. **Create Project**:
   ```bash
   oc new-project object-detection
   ```

2. **Create Application**:
   ```bash
   oc new-app python:3.11~https://github.com/apelline1/object-detection-rest_original
   ```

3. **Expose Service**:
   ```bash
   oc expose svc/object-detection-rest
   ```

4. **Configure Resources** (via deployment.yaml):
   - Requests: 512Mi memory, 250m CPU
   - Limits: 2Gi memory, 1000m CPU

### Using Deployment Manifest

```bash
# Apply the deployment configuration
oc apply -f deployment.yaml

# Verify deployment
oc get pods -l app=object-detection-rest
oc get svc object-detection-rest
oc get route object-detection-rest
```

### Private Registry Configuration

Update `deployment.yaml` for internal Quay registry:
```yaml
imagePullSecrets:
  - name: quay-registry-secret

containers:
  - image: quay-registry.example.com/namespace/object-detection-rest:latest
```

## Development Workflow

### Local Development with Jupyter

1. **Clone Repository**:
   ```bash
   git clone https://github.com/apelline1/object-detection-rest_original.git
   cd object-detection-rest_original
   ```

2. **Install Dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

3. **Explore Notebooks**:
   - `0_sandbox.ipynb` - Getting started
   - `1_explore.ipynb` - Data exploration
   - `2_predict.ipynb` - Test predictions
   - `3_run_flask.ipynb` - Run Flask locally
   - `4_test_flask.ipynb` - Test endpoints

4. **Run Flask Locally**:
   ```bash
   FLASK_APP=wsgi.py flask run
   ```

5. **Test Endpoint**:
   ```bash
   curl -X POST http://localhost:5000/predictions \
     -H "Content-Type: application/json" \
     -d '{"image": "base64_encoded_string"}'
   ```

### Making Changes

1. Modify `prediction.py` for prediction logic
2. Update `wsgi.py` for new endpoints
3. Test locally using notebooks
4. Update `requirements.txt` if adding dependencies
5. Commit and push to trigger rebuild (if webhook configured)

## Git Repository Information

### Repository Details
- **URL**: https://github.com/apelline1/object-detection-rest_original
- **Owner**: apelline1
- **License**: GPLv3

### Recent Development History

1. **Model Loading Improvements** (7649e20)
   - Added model loading status messages for better visibility

2. **Multi-Format Support** (3c7405d)
   - Fixed image decoding to support PNG, JPEG, and other formats
   - Improved error handling for various image types

3. **Error Handling Enhancements** (31754c7, f785d47, 568b861, 1f961eb, 9012d0c)
   - Added test endpoint
   - Improved handling of large images
   - Extensive request logging for debugging
   - TensorFlow-specific exception handling
   - Detailed error messages

4. **Frontend Integration** (e403bd0)
   - Added `/api/images` endpoint for frontend compatibility
   - Enhanced error handling for production use

5. **Deployment Configuration** (a339d96)
   - Added deployment.yaml for OpenShift/Kubernetes
   - Internal Quay registry configuration

6. **Core Functionality** (1fce360, 279a5e2)
   - Implemented predictive code
   - TensorFlow integration and testing
   - Pillow installation instructions

## Model Information

### OpenImages v4 SSD MobileNet v2

- **Architecture**: Single Shot Detector (SSD)
- **Backbone**: MobileNet v2
- **Training Dataset**: OpenImages v4
- **Use Case**: Real-time object detection
- **Performance**: Optimized for mobile/edge deployment
- **Classes**: Multiple object classes from OpenImages dataset

### Model Loading

- Model loaded at application startup
- Preloaded with blank image for warmup
- Located in `models/openimages_v4_ssd_mobilenet_v2_1/`
- TensorFlow SavedModel format

## Performance Considerations

### Memory Management
- Single worker process (TensorFlow memory optimization)
- 10MB image size limit
- Resource exhaustion error handling
- Model preloading to reduce cold start time

### Scaling Considerations
- Horizontal scaling via multiple pods
- Each pod should have dedicated memory (2Gi limit)
- Consider GPU support for higher throughput
- Load balancer recommended for production

### Response Times
- Typical: 100-500ms for small images
- Large images: 1-3 seconds
- Cold start: 5-10 seconds (model loading)

## Testing

### Unit Testing
```bash
python -m pytest tests/
```

### Integration Testing
Use provided Jupyter notebooks:
- `3_run_flask.ipynb` - Start Flask server
- `4_test_flask.ipynb` - Test all endpoints

### Sample Requests
Check `sample-requests/` directory for example payloads.

## Troubleshooting

### Common Issues

1. **Model Not Loading**
   - Verify `models/` directory exists
   - Check TensorFlow version compatibility
   - Review startup logs

2. **Image Decode Errors**
   - Verify base64 encoding
   - Check image format (JPEG, PNG supported)
   - Ensure proper data URL prefix handling

3. **Memory Errors**
   - Reduce image size
   - Increase pod memory limits
   - Check for memory leaks

4. **Timeout Errors**
   - Increase gunicorn timeout
   - Optimize image preprocessing
   - Consider async processing for large images

## Integration with Frontend

This API integrates with the object-detection-app frontend:
- Frontend sends images via `/api/images` endpoint
- Receives bounding box coordinates and labels
- Renders detections on canvas overlay

Example integration:
```javascript
const response = await fetch('http://object-detection-rest:8080/api/images', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ image: base64Image })
});
const result = await response.json();
```

## Security Considerations

- Input validation on all endpoints
- Size limits to prevent DoS
- Error messages sanitized (no stack traces in production)
- TLS recommended for production
- Authentication not implemented (add middleware if needed)

## Monitoring & Logging

- Extensive logging to stdout/stderr
- Container-friendly logging format
- Request/response logging
- Error tracking with full stack traces
- Health check endpoint for liveness/readiness probes

## Future Enhancements

- [ ] Add authentication/authorization
- [ ] Implement rate limiting
- [ ] Add Prometheus metrics
- [ ] Support batch processing
- [ ] Add model versioning
- [ ] GPU support
- [ ] Async processing queue
- [ ] Response caching

---

**Last Updated**: 2026-04-22  
**Documentation Version**: 1.0  
**API Version**: 1.0.0
