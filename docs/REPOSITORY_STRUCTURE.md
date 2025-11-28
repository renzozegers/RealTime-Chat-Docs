# Repository Structure

This repository contains architecture and engineering documentation only. Source code is not included.

## Directory Structure

```
chat-server/
├── README.md                          # Main README - architecture overview
├── LICENSE                            # MIT License
├── .gitignore                         # Git ignore rules
│
├── diagrams/                          # Architecture diagrams (Mermaid format)
│   ├── system-architecture.md        # High-level system architecture
│   ├── message-delivery-flow.md      # Message delivery sequence diagram
│   └── reconnection-flow.md          # Reconnection and offline sync flow
│
└── docs/                              # Complete technical documentation
    ├── ARCHITECTURE.md                # Detailed system architecture
    ├── DELIVERY_PIPELINE.md           # Message delivery pipeline details
    ├── RECONNECTION.md                # Reconnection and offline sync
    ├── CACHING.md                     # Caching strategy (Redis and in-memory)
    ├── SCALING.md                     # Horizontal scaling and cluster setup
    ├── DB_OPTIMIZATION.md             # SQL optimizations and schema design
    ├── API_REFERENCE.md               # Complete WebSocket and HTTP API
    ├── SEQUENCE_DIAGRAMS.md           # Detailed interaction sequence diagrams
    ├── TESTING_STRATEGY.md            # Testing approach and test suite
    ├── REPOSITORY_STRUCTURE.md        # This file
    │
    └── (SQL schema files excluded - NDA compliance)
```

## File Organization

### Documentation Files

**Architecture Documentation:**
- `ARCHITECTURE.md`: Complete system architecture, components, data persistence, security
- `DELIVERY_PIPELINE.md`: Message delivery stages, validation, persistence, caching
- `RECONNECTION.md`: Disconnection detection, grace period, event queue delivery
- `CACHING.md`: Multi-layer caching strategy, Redis and in-memory
- `SCALING.md`: Horizontal scaling, cluster mode, limitations
- `DB_OPTIMIZATION.md`: Query optimizations, indexes, connection pooling

**API Documentation:**
- `API_REFERENCE.md`: Complete WebSocket message types and HTTP endpoints
- `SEQUENCE_DIAGRAMS.md`: Detailed interaction sequence diagrams

**Development Documentation:**
- `TESTING_STRATEGY.md`: Testing approach, test structure, execution

### Diagrams

**Location:** `diagrams/`

All diagrams are in Mermaid format and can be rendered in:
- GitHub (automatic rendering)
- Documentation tools (MkDocs, Docusaurus, etc.)
- Mermaid Live Editor

**Diagrams:**
1. **system-architecture.md**: High-level system architecture with all components
2. **message-delivery-flow.md**: Complete message delivery sequence
3. **reconnection-flow.md**: Reconnection and offline event delivery flow

## Documentation Principles

### Architecture-Focused

This repository focuses on:
- System architecture and design
- Engineering decisions and trade-offs
- Performance characteristics
- Scaling strategies
- Database schema and optimizations
- API contracts and interfaces

### No Source Code

This repository does NOT include:
- Source code files (handlers, database operations, security utilities)
- Configuration files (package.json, env.example)
- Test files
- Deployment scripts
- Utility scripts

### Technical and Concise

All documentation:
- Technical and engineer-focused
- No marketing language
- Includes implementation details

## Key Documentation Files

### System Architecture
- `docs/ARCHITECTURE.md`: Complete system design
- `diagrams/system-architecture.md`: Visual architecture diagram

### Message Flow
- `docs/DELIVERY_PIPELINE.md`: Message delivery details
- `diagrams/message-delivery-flow.md`: Delivery sequence diagram

### Reconnection
- `docs/RECONNECTION.md`: Reconnection logic
- `diagrams/reconnection-flow.md`: Reconnection sequence diagram

### Performance
- `docs/DB_OPTIMIZATION.md`: Database optimizations
- `docs/CACHING.md`: Caching strategies
- `docs/SCALING.md`: Scaling approaches

### API
- `docs/API_REFERENCE.md`: Complete API documentation
- `docs/SEQUENCE_DIAGRAMS.md`: Interaction flows

## What's Included

- Architecture diagrams (Mermaid format)
- System design documentation
- Message flow documentation
- Reconnection and offline delivery documentation
- Caching strategy documentation
- Scaling guide
- Database optimization documentation
- API reference
- Sequence diagrams
- Testing strategy
- Database optimization documentation (SQL schemas excluded - NDA)

## What's NOT Included

- Source code files
- Configuration files
- Test files
- Deployment scripts
- Utility scripts
- Package dependencies

## Usage

This repository is intended for:
- Architecture review and understanding
- Engineering design decisions
- API contract reference
- Performance optimization reference
- Scaling strategy reference

