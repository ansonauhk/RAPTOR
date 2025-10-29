# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RAPTOR (Rapid AI-Powered Text and Object Recognition) is a Content Insight Engine for enterprise AI applications. It's a multi-modal content analysis platform that processes video, audio, images, and documents using AI/ML models with semantic search capabilities.

**Current Release**: Aigle 0.1 (Beta) - First community release
**Working Directory**: `/mnt/c/dev/RAPTOR/Aigle/0.1/`

## Development Commands

### Environment Setup

```bash
# Create and activate conda environment
conda create -n CIE python=3.10
conda activate CIE

# Install dependencies
pip install -r requirements.txt

# For PaddleOCR (optional)
python -m pip install paddlepaddle-gpu==3.0.0 -i https://www.paddlepaddle.org.cn/packages/stable/cu118/
python -m pip install "paddleocr[all]"
```

### Docker Deployment

```bash
cd Aigle/0.1/raptor

# Make scripts executable
chmod +x check-services.sh deploy.sh logs.sh

# Deploy all services
./deploy.sh

# Check service status
./check-services.sh

# View service logs
./logs.sh <service_name>
```

### Kafka Services

```bash
cd Aigle/0.1/raptor/kafka

# Create Kafka topics
chmod +x create_topic.sh
sudo ./create_topic.sh

# Start processing services (video/audio/image/document)
cd services
chmod +x start_services.sh
./start_services.sh

# Check service status
./check_services.sh

# Stop services
./stop_services.sh
```

### Testing

```bash
# Test Kafka services
cd Aigle/0.1/raptor/kafka/test_service
python test.py

# View service logs
cd Aigle/0.1/raptor/kafka
tail -f <service_name>.log
# Example service names: document_orchestrator_service, audio_analysis_service, video_summary_service

# Check Redis data
sudo docker exec -it redis-kafka_dev redis-cli --raw
GET "document_orchestrator:correlation_id"
```

### API Testing

```bash
# Test ModelLifecycle service
curl -s http://192.168.157.165:8086/docs

# Test AssetManagement service
curl -s http://192.168.157.165:8010/docs

# Search APIs (video/audio/document/image)
curl -X POST "http://192.168.157.165:8822/audio_search" \
  -H "Content-Type: application/json" \
  -d '{"query_text": "OpenAI", "embedding_type": "text", "limit": 5}'
```

## Architecture Overview

### Core Components

RAPTOR consists of three major subsystems that work together:

#### 1. AI Model Lifecycle Management (AiModelLifecycle)
- **Purpose**: MLOps platform for model versioning, registration, deployment, and inference
- **Port**: 8010
- **Key Features**:
  - MLflow for model registry and tracking
  - LakeFS for Git-like model versioning
  - Unified inference API supporting Ollama and Transformers engines
  - Supports text generation, VLM, ASR, OCR, video/audio/document analysis
- **Architecture**: 3-layer inference system (Engine → Manager → API)
- **Model Sources**: HuggingFace Hub, local Ollama models

#### 2. Asset Management (asset_management)
- **Purpose**: Digital asset storage with versioning, access control, and lifecycle management
- **Port**: 8086
- **Key Features**:
  - JWT-based authentication with RBAC
  - SeaweedFS cluster (3 masters, 4 volumes) with 011 replication
  - LakeFS for version control
  - MySQL for metadata and audit logs
  - Automated TTL-based archival and destruction
- **Storage Stack**: FastAPI → LakeFS → SeaweedFS S3 → Qdrant (metadata)

#### 3. Multi-Modal Processing Pipeline (kafka/services)
- **Purpose**: Event-driven content analysis workflows
- **Architecture**: Kafka-based microservices with Redis state management
- **Processing Flow**: Orchestrator → Analysis → Summary → Save to Qdrant
- **Media Types**: Video, Audio, Document, Image
- **Services per type**:
  - Video: 7 services (orchestrator, analysis, frame description, OCR, scene detection, summary, save2qdrant)
  - Audio: 6 services (orchestrator, recognizer, diarization, classifier, summary, save2qdrant)
  - Document: 4 services (orchestrator, analysis, summary, save2qdrant)
  - Image: 3 services (orchestrator, processing, save2qdrant)

### Infrastructure Stack

**AI/ML Layer**:
- MLflow (experiment tracking, model registry)
- LangChain (LLM orchestration)
- Ollama (local LLM inference)
- Transformers/PyTorch (model execution)
- Whisper (speech recognition)
- PaddleOCR (text recognition)

**Storage Layer**:
- SeaweedFS (distributed object storage)
- LakeFS (data versioning)
- Qdrant (vector database)
- Redis Cluster (caching, state management)
- MySQL (metadata, audit logs)

**Message/Compute Layer**:
- Apache Kafka (event streaming)
- FastAPI (REST APIs)
- Docker Compose (orchestration)

**Observability**:
- Prometheus (metrics)
- Grafana (dashboards)
- MLflow UI (experiment tracking)

## Working with the Codebase

### Service Architecture Patterns

**Event-Driven Processing**: All media processing services follow an orchestrator pattern where an orchestrator service receives requests via Kafka, coordinates work across specialized processing services, tracks state in Redis, and publishes results to Qdrant.

**State Management**: Each orchestrator maintains a correlation_id in Redis to track processing status. Check state with:
```bash
redis-cli GET "<media_type>_orchestrator:<correlation_id>"
```

**Service Communication**: Services communicate via Kafka topics. Topic naming follows the pattern: `<media_type>_<stage>` (e.g., `document_analysis`, `video_summary`).

### Key Configuration Files

- `/Aigle/0.1/raptor/.env` - Docker service configurations (ports, credentials, endpoints)
- `/Aigle/0.1/raptor/docker-compose.yaml` - All service definitions and networking
- `/Aigle/0.1/raptor/AiModelLifecycle/src/core/configs/base.yaml` - MLflow/LakeFS/Ollama endpoints

### Port Mappings (Host:Container)

**Core Services**:
- Asset Management API: 8086:8000
- Model Lifecycle API: 8010:8010
- MLflow UI: 5000:5000
- RedisInsight: 5540:5540

**Redis Cluster**: 7000-7005 (nodes 1-6)

**SeaweedFS**:
- Masters: 9343-9345 (HTTP), 19343-19345 (gRPC)
- Volumes: 8091-8094 (HTTP), 18091-18094 (gRPC)
- Filer: 8898:8888, S3 Gateway: 8343:8333
- Admin UI: 23656:23646

**Search APIs**:
- Video: 8821:8811
- Audio: 8822:8812
- Document: 8823:8813
- Image: 8824:8814

**Kafka**: 19002-19004 (brokers 1-3), Kafdrop UI: 9020:9000

**Monitoring**: Prometheus: 9091:9090, Grafana: 3031:3000

### Model Registration Workflow

For quick Ollama model deployment:

```bash
# 1. List available models
ollama list
# or via API:
curl "http://192.168.157.165:8010/models/local?model_source=ollama"

# 2. Register model to MLflow
curl -X POST "http://192.168.157.165:8010/models/register_ollama" \
  -H "Content-Type: application/json" \
  -d '{
    "local_model_name": "qwen2.5:7b",
    "task": "text-generation-ollama",
    "registered_name": "qwenforsummary",
    "stage": "production"
  }'

# 3. Run inference
curl -X POST "http://192.168.157.165:8010/inference/infer" \
  -H "Content-Type: application/json" \
  -d '{
    "task": "text-generation-ollama",
    "engine": "ollama",
    "model_name": "qwenforsummary",
    "data": {"inputs": "你好"}
  }'
```

### Common Development Workflows

**Adding a new task type to inference system**:
1. Create handler in `AiModelLifecycle/src/inference/models/<task>.py` extending `BaseModelHandler`
2. Register handler in `AiModelLifecycle/src/inference/registry.py`
3. Update task-to-engine mapping in router

**Adding a new media processing service**:
1. Create service directory under `kafka/services/<media>_<function>_service/`
2. Implement Kafka consumer/producer logic
3. Add service to `start_services.sh` with proper CUDA_VISIBLE_DEVICES
4. Create corresponding Kafka topic in `create_topic.sh`

**Debugging service failures**:
1. Check Docker container status: `./check-services.sh`
2. View service logs: `./logs.sh <service_name>` or `tail -f <service>.log`
3. Check Redis state: `redis-cli GET "<service>:correlation_id"`
4. Verify Kafka topics: Access Kafdrop UI at port 9020

## Important Notes

- The IP address `192.168.157.165` is hardcoded in many examples - adjust for your environment
- Default credentials: Most services use `dht888888` as password (check `.env` files)
- CUDA configuration: Each Kafka service has `CUDA_VISIBLE_DEVICES` setting to prevent OOM
- NFS mounts: SeaweedFS uses NFS for persistent storage - ensure NFS server is configured
- Service dependencies: Start Docker services before Kafka services; Kafka services depend on Redis, MLflow, and Qdrant

## Development Guidelines

**When modifying inference system**: The 3-layer architecture (Engine/Manager/API) must be maintained. New engines go in `inference/engines/`, new model handlers in `inference/models/`, routing logic in `manager.py`.

**When adding new services**: Follow the orchestrator pattern. Each media type should have dedicated services for orchestration, analysis, summarization, and Qdrant persistence.

**When working with storage**: All file operations go through LakeFS, which sits on top of SeaweedFS. Direct S3 operations should use the `object_store.py` abstraction layer.

**CUDA memory management**: When deploying services, explicitly set `CUDA_VISIBLE_DEVICES` to distribute GPU workload and prevent OOM errors across multiple services.

## Documentation Resources

- System Design: `Aigle/0.1/CIE_System_Design_and_Architecture_1.8.pdf`
- Technical Implementation: `Aigle/0.1/doc/CIE_System_Technical_Implementation_1.2.pdf`
- Model Lifecycle README: `Aigle/0.1/raptor/AiModelLifecycle/README.md`
- Asset Management README: `Aigle/0.1/raptor/asset_management/README.md`
- Kafka Services README: `Aigle/0.1/raptor/kafka/README.md`
- Main CHANGELOG: `MAIN_DOCUMENTATION/CHANGELOG.md`
