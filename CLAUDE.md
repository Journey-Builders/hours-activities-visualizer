# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Construction Workforce Verification System designed to visualize worker location data on interactive floor blueprints and automatically verify timesheets based on spatial-temporal analysis. The system tracks worker movements throughout construction sites using IoT/GPS data and enables foremen to quickly review and approve daily reports.

## Development Commands

Since this project is in the blueprint phase, common development commands will be:

```bash
# Initial setup (once implemented)
npm install                 # Install dependencies
npm run dev                 # Start development server (frontend & backend)
npm run build              # Build production bundle
npm run test               # Run all tests
npm run test:watch         # Run tests in watch mode
npm run lint               # Run ESLint
npm run typecheck          # Run TypeScript type checking
npm run db:migrate         # Run database migrations
npm run db:seed            # Seed development data

# Docker commands
docker-compose up -d       # Start all services
docker-compose logs -f     # View logs
```

## Architecture Overview

### System Components

1. **Backend API (Node.js/Express/TypeScript)**
   - Location data ingestion from IoT devices
   - Spatial zone mapping using Turf.js
   - Time aggregation and verification algorithms
   - Real-time processing with Socket.io
   - Machine learning integration for anomaly detection

2. **Frontend (React/TypeScript)**
   - Interactive SVG-based floor plans using Konva.js
   - Real-time heatmap overlays
   - Time slider for playback
   - Worker verification panels
   - Batch verification workflows

3. **Data Processing Pipeline**
   - Point-in-polygon detection for zone assignment
   - Time aggregation with configurable thresholds
   - Confidence scoring based on GPS accuracy and historical patterns
   - Anomaly detection for unusual patterns

### Key Data Flow

1. **Location Ingestion**: IoT devices → API → Zone Mapping → Database
2. **Verification**: Worker Locations → Time Aggregation → Activity Matching → Confidence Scoring
3. **Visualization**: Database → API → React Components → Interactive Blueprints

### Core Services

- `ZoneMappingService`: Maps GPS coordinates to building zones
- `TimeAggregationService`: Calculates time spent in each zone
- `VerificationService`: Automates timesheet verification
- `RealtimeLocationProcessor`: Handles real-time location updates

### Technology Stack

- **Backend**: Node.js, Express, TypeScript, Turf.js (geospatial)
- **Frontend**: React, TypeScript, Konva.js (canvas rendering)
- **Real-time**: Socket.io, Redis
- **ML**: Python with scikit-learn (anomaly detection)
- **Database**: PostgreSQL/MongoDB (to be determined)
- **Testing**: Jest
- **Deployment**: Docker

## Implementation Priorities

When implementing features, follow this order:

1. Set up project structure with TypeScript configuration
2. Implement zone mapping algorithm using Turf.js
3. Create basic API endpoints for location ingestion
4. Build interactive blueprint UI with React and Konva.js
5. Implement time aggregation and verification logic
6. Add real-time updates using WebSockets
7. Integrate machine learning for anomaly detection
8. Add batch verification and export functionality

## Key Algorithms

### Zone Mapping
- Use Turf.js for point-in-polygon detection
- Account for GPS accuracy in confidence calculations
- Handle transition grace periods between zones

### Time Aggregation
- Minimum zone duration: 5 minutes
- Transition grace period: 2 minutes
- Consolidate multiple visits to the same zone

### Verification Scoring
- Auto-verify: >85% time in assigned zones + >7 hours total
- Needs review: 50-85% time in assigned zones
- Manual entry: <50% time in assigned zones

## Testing Strategy

- Unit tests for all services and algorithms
- Integration tests for API endpoints
- Component tests for React components
- E2E tests for critical workflows (verification, export)
- Performance tests for real-time processing

## Sample Data Structure

The system uses:
- 5 floors (Ground, 42, 43, 44, 51)
- 4-6 zones per floor
- 50 workers with varying IoT data quality
- 3-5 activities per day
- Location pings every 5 minutes