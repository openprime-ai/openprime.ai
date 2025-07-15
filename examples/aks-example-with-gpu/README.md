# AKS Example with GPU - OpenPrime AI Architecture

This example demonstrates the deployment of OpenPrime AI on Azure Kubernetes Service (AKS) with GPU support for optimal AI/ML workloads performance.

## Architecture Overview

The following Mermaid diagram illustrates the complete Kubernetes architecture deployed via Helmfile and Helm charts:

```mermaid
graph TB
    subgraph "External Access"
        Internet[Internet Users]
        Cert[Let's Encrypt Certificate Manager]
    end

    subgraph "AKS Cluster"
        subgraph "Ingress Layer"
            Nginx[NGINX Ingress Controller]
            Internet --> Nginx
        end

        subgraph "GPU Operator Namespace"
            GPUOp[NVIDIA GPU Operator<br/>v25.3.4]
            GPURuntime[NVIDIA Container Runtime]
            GPUOp --> GPURuntime
        end

        subgraph "MinIO Operator Namespace"
            MinIOOp[MinIO Operator<br/>v7.1.1]
        end

        subgraph "OpenPrime AI Namespace"
            subgraph "Web Layer"
                OpenWebUI[OpenWeb UI<br/>Port 8080<br/>openprime.ai]
                Nginx --> OpenWebUI
            end

            subgraph "AI/ML Layer"
                Ollama[Ollama LLM Server<br/>Port 11434<br/>ollama.openprime.ai<br/>GPU Support]
                LightRAG[LightRAG Engine<br/>Port 9621<br/>lightrag.openprime.ai<br/>Graph RAG System]
                GPURuntime -.-> Ollama
                OpenWebUI --> Ollama
                LightRAG --> Ollama
            end

            subgraph "Database Layer"
                Redis[Redis HA<br/>Port 6379<br/>Key-Value Store]
                Neo4j[Neo4j Community<br/>Port 7687<br/>Graph Database]
                Qdrant[Qdrant Vector DB<br/>Port 6333<br/>qdrant.openprime.ai<br/>Vector Storage]

                LightRAG --> Redis
                LightRAG --> Neo4j
                LightRAG --> Qdrant
                OpenWebUI --> Qdrant
            end

            subgraph "Storage Layer"
                MinIO[MinIO Tenant<br/>minio.openprime.ai<br/>Object Storage]
                MinIOOp --> MinIO
            end
        end

        subgraph "Persistent Storage"
            AzureDisk[Azure Managed Disks<br/>managed-csi StorageClass]
            OpenWebUI --> AzureDisk
            Ollama --> AzureDisk
            Redis --> AzureDisk
            Neo4j --> AzureDisk
            Qdrant --> AzureDisk
            MinIO --> AzureDisk
            LightRAG --> AzureDisk
        end

        subgraph "Certificates"
            Cert --> Nginx
            TLS1[TLS: openprime.ai]
            TLS2[TLS: ollama.openprime.ai]
            TLS3[TLS: qdrant.openprime.ai]
            TLS4[TLS: lightrag.openprime.ai]
            TLS5[TLS: minio.openprime.ai]
        end
    end

    subgraph "GPU Nodes"
        GPUNode[AKS GPU Nodes<br/>NVIDIA Container Runtime<br/>Spot Instances with Tolerations]
        Ollama -.-> GPUNode
    end

    subgraph "Data Flow"
        OpenWebUI -->|RAG Queries| LightRAG
        LightRAG -->|Embeddings| Ollama
        LightRAG -->|Knowledge Graph| Neo4j
        LightRAG -->|Vector Search| Qdrant
        LightRAG -->|Cache/KV| Redis
        OpenWebUI -->|Vector DB| Qdrant
        OpenWebUI -->|Documents| MinIO
    end

    classDef webLayer fill:#e1f5fe
    classDef aiLayer fill:#f3e5f5
    classDef dbLayer fill:#e8f5e8
    classDef storageLayer fill:#fff3e0
    classDef gpuLayer fill:#ffebee
    classDef ingressLayer fill:#f1f8e9

    class OpenWebUI webLayer
    class Ollama,LightRAG aiLayer
    class Redis,Neo4j,Qdrant dbLayer
    class MinIO,AzureDisk storageLayer
    class GPUOp,GPURuntime,GPUNode gpuLayer
    class Nginx ingressLayer
```

## Component Overview

### Core AI Components
- **OpenWeb UI**: Primary web interface for interacting with AI models
- **Ollama**: Large Language Model server with GPU acceleration
- **LightRAG**: Advanced Graph-based Retrieval-Augmented Generation system

### Database Layer
- **Redis HA**: High-availability key-value store for caching and session management
- **Neo4j**: Graph database for storing knowledge relationships
- **Qdrant**: Vector database for semantic search and embeddings

### Infrastructure Components
- **NVIDIA GPU Operator**: Manages GPU resources and container runtime
- **MinIO**: S3-compatible object storage for documents and artifacts
- **NGINX Ingress**: Load balancing and SSL termination

### Storage & Security
- **Azure Managed Disks**: Persistent storage with CSI driver
- **Let's Encrypt**: Automated SSL certificate management
- **Pod Security Contexts**: Non-root containers with minimal privileges

## Deployment Architecture

### Helmfile Deployment Order
1. **MinIO Operator** - Provides object storage capabilities
2. **GPU Operator** - Enables GPU support for AI workloads
3. **OpenPrime AI** - Main application stack (depends on operators)

### Network Communication
- Internal service-to-service communication via ClusterIP services
- External access through NGINX Ingress with TLS termination
- GPU workloads scheduled on nodes with NVIDIA container runtime

### Resource Management
- GPU resources allocated to Ollama for LLM inference
- Persistent volumes for model storage and database persistence
- Resource limits and requests configured for production workloads

This architecture provides a scalable, production-ready AI platform with GPU acceleration, comprehensive data storage, and secure external access.