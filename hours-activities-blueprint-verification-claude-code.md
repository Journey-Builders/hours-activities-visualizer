# Blueprint Verification System - Claude Code Implementation Guide

## Project Overview

Build a spatial verification system for construction daily reports that visualizes worker location data on interactive floor blueprints, enabling foremen to quickly verify and submit accurate timesheets.

## Core Architecture

```typescript
// Core data structures
interface WorkerLocation {
  workerId: string;
  timestamp: Date;
  coordinates: {
    x: number;
    y: number;
    z: number; // floor level
    accuracy: number; // GPS accuracy in meters
  };
  floorId: string;
  zoneId?: string; // computed from coordinates
}

interface Zone {
  id: string;
  floorId: string;
  name: string;
  type: 'primary' | 'support' | 'transit' | 'restricted';
  geometry: {
    type: 'rectangle' | 'polygon';
    coordinates: number[][];
  };
  assignedActivities: string[];
}

interface ActivityAssignment {
  activityId: string;
  activityCode: string;
  description: string;
  plannedZones: {
    zoneId: string;
    type: 'primary' | 'support' | 'auxiliary';
    expectedHours: number;
  }[];
  assignedWorkers: string[];
  plannedDate: Date;
  plannedHours: number;
}

interface VerificationResult {
  workerId: string;
  date: Date;
  totalHours: number;
  verificationStatus: 'auto-verified' | 'needs-review' | 'manual-entry';
  confidence: number;
  zoneBreakdown: {
    zoneId: string;
    hours: number;
    percentage: number;
    isAssigned: boolean;
  }[];
  suggestedActivity?: string;
  anomalies: string[];
}
```

## Implementation Prompt for Claude Code

```markdown
I need to build a construction workforce verification system with the following components:

## 1. Backend API (Node.js/Express)

Create an API that:
- Ingests worker location data from IoT devices (GPS coordinates with timestamps)
- Maps coordinates to building zones using spatial queries
- Calculates time spent in each zone per worker
- Generates verification scores based on planned vs actual locations
- Provides real-time and historical data endpoints

Key endpoints needed:
- POST /api/locations/batch - Bulk upload location data
- GET /api/workers/:date/summary - Daily summary with zone breakdowns
- GET /api/floors/:floorId/heatmap - Heatmap data for visualization
- POST /api/activities/verify - Verify worker-activity assignments
- GET /api/reports/daily/:date - Generate daily report data

## 2. Frontend Application (React/TypeScript)

Build an interactive blueprint interface with:
- SVG-based floor plans with clickable zones
- Real-time heatmap overlay showing worker density
- Time slider for playback throughout the day
- Worker markers with verification status indicators
- Side panel showing zone details and worker lists
- Batch verification actions

Key features:
- Drag-and-drop to reassign workers
- Click zone to see all workers and verify activity
- Confidence scoring visualization
- Anomaly highlighting
- Export verified report

## 3. Data Processing Pipeline

Implement algorithms for:
- Point-in-polygon detection for zone assignment
- Time aggregation with configurable thresholds
- Confidence scoring based on:
  - GPS accuracy
  - Time consistency in zone
  - Historical patterns
  - Proximity to assigned zones
- Anomaly detection for unusual patterns

## 4. Sample Data Structure

Use this schema for testing:
- 5 floors (Ground, 42, 43, 44, 51)
- 4-6 zones per floor
- 50 workers with varying IoT data quality
- 3-5 activities per day
- Location pings every 5 minutes
```

## Detailed Implementation Files

### 1. Backend Zone Mapping Service

```typescript
// services/zoneMapping.service.ts
import { point, polygon } from '@turf/helpers';
import booleanPointInPolygon from '@turf/boolean-point-in-polygon';

export class ZoneMappingService {
  private zones: Map<string, Zone> = new Map();
  
  async mapLocationToZone(location: WorkerLocation): Promise<string | null> {
    const floors = await this.getFloorZones(location.floorId);
    
    for (const zone of floors) {
      const pt = point([location.coordinates.x, location.coordinates.y]);
      const poly = polygon([zone.geometry.coordinates]);
      
      if (booleanPointInPolygon(pt, poly)) {
        // Account for GPS accuracy
        const confidence = this.calculateLocationConfidence(
          location.coordinates.accuracy,
          zone.geometry,
          pt
        );
        
        if (confidence > 0.7) {
          return zone.id;
        }
      }
    }
    
    return null;
  }
  
  private calculateLocationConfidence(
    accuracy: number,
    zoneGeometry: any,
    point: any
  ): number {
    // Calculate distance from point to zone boundary
    const distanceToBoundary = this.getDistanceToBoundary(point, zoneGeometry);
    
    // Higher confidence if point is well within zone despite GPS accuracy
    if (distanceToBoundary > accuracy * 2) return 1.0;
    if (distanceToBoundary > accuracy) return 0.85;
    if (distanceToBoundary > accuracy * 0.5) return 0.7;
    
    return 0.5;
  }
}
```

### 2. Worker Time Aggregation

```typescript
// services/timeAggregation.service.ts
export class TimeAggregationService {
  private readonly MIN_ZONE_DURATION = 5; // minutes
  private readonly TRANSITION_GRACE_PERIOD = 2; // minutes
  
  async aggregateWorkerTime(
    workerId: string,
    date: Date
  ): Promise<ZoneTimeBreakdown[]> {
    const locations = await this.getWorkerLocations(workerId, date);
    const zoneVisits = this.groupLocationsByZone(locations);
    
    return zoneVisits.map(visit => ({
      zoneId: visit.zoneId,
      totalMinutes: this.calculateDuration(visit.entries),
      visits: this.consolidateVisits(visit.entries),
      confidence: this.calculateConfidence(visit.entries)
    }));
  }
  
  private consolidateVisits(entries: LocationEntry[]): Visit[] {
    const visits: Visit[] = [];
    let currentVisit: Visit | null = null;
    
    entries.forEach((entry, index) => {
      if (!currentVisit) {
        currentVisit = { start: entry.timestamp, end: entry.timestamp };
      } else {
        const timeSinceLastEntry = 
          entry.timestamp.getTime() - currentVisit.end.getTime();
        
        if (timeSinceLastEntry <= this.TRANSITION_GRACE_PERIOD * 60 * 1000) {
          currentVisit.end = entry.timestamp;
        } else {
          if (this.getVisitDuration(currentVisit) >= this.MIN_ZONE_DURATION) {
            visits.push(currentVisit);
          }
          currentVisit = { start: entry.timestamp, end: entry.timestamp };
        }
      }
    });
    
    if (currentVisit && 
        this.getVisitDuration(currentVisit) >= this.MIN_ZONE_DURATION) {
      visits.push(currentVisit);
    }
    
    return visits;
  }
}
```

### 3. Frontend Blueprint Component

```tsx
// components/BlueprintView.tsx
import React, { useState, useEffect } from 'react';
import { Stage, Layer, Rect, Circle, Text } from 'react-konva';
import { HeatmapOverlay } from './HeatmapOverlay';
import { WorkerMarker } from './WorkerMarker';
import { ZoneDetailsPanel } from './ZoneDetailsPanel';

interface BlueprintViewProps {
  floorId: string;
  date: Date;
  onVerifyActivity: (zoneId: string, activityId: string) => void;
}

export const BlueprintView: React.FC<BlueprintViewProps> = ({
  floorId,
  date,
  onVerifyActivity
}) => {
  const [selectedZone, setSelectedZone] = useState<string | null>(null);
  const [timeSliderValue, setTimeSliderValue] = useState(100);
  const [viewMode, setViewMode] = useState<'heatmap' | 'workers'>('heatmap');
  const [zones, setZones] = useState<Zone[]>([]);
  const [workerLocations, setWorkerLocations] = useState<WorkerLocation[]>([]);
  
  useEffect(() => {
    loadFloorData();
  }, [floorId, date]);
  
  const loadFloorData = async () => {
    const [zonesData, workersData] = await Promise.all([
      fetch(`/api/floors/${floorId}/zones`).then(r => r.json()),
      fetch(`/api/floors/${floorId}/workers?date=${date}`).then(r => r.json())
    ]);
    
    setZones(zonesData);
    setWorkerLocations(workersData);
  };
  
  const handleZoneClick = (zoneId: string) => {
    setSelectedZone(zoneId);
    // Load zone details in side panel
  };
  
  const getZoneColor = (zone: Zone) => {
    if (selectedZone === zone.id) return '#1a73e8';
    
    const workerCount = getWorkersInZone(zone.id).length;
    if (workerCount > 5) return 'rgba(76, 175, 80, 0.3)';
    if (workerCount > 2) return 'rgba(255, 152, 0, 0.3)';
    if (workerCount > 0) return 'rgba(244, 67, 54, 0.3)';
    return 'rgba(255, 255, 255, 0.1)';
  };
  
  return (
    <div className="blueprint-container">
      <div className="blueprint-canvas">
        <Stage width={1200} height={800}>
          <Layer>
            {/* Render zones */}
            {zones.map(zone => (
              <Rect
                key={zone.id}
                x={zone.geometry.x}
                y={zone.geometry.y}
                width={zone.geometry.width}
                height={zone.geometry.height}
                fill={getZoneColor(zone)}
                stroke="rgba(255, 255, 255, 0.3)"
                strokeWidth={2}
                cornerRadius={5}
                onClick={() => handleZoneClick(zone.id)}
                onTap={() => handleZoneClick(zone.id)}
              />
            ))}
            
            {/* Zone labels */}
            {zones.map(zone => (
              <Text
                key={`label-${zone.id}`}
                x={zone.geometry.x + zone.geometry.width / 2}
                y={zone.geometry.y + zone.geometry.height / 2}
                text={zone.name}
                fontSize={14}
                fill="white"
                align="center"
                listening={false}
              />
            ))}
          </Layer>
          
          {/* Heatmap overlay */}
          {viewMode === 'heatmap' && (
            <Layer>
              <HeatmapOverlay 
                workerLocations={workerLocations}
                timeProgress={timeSliderValue / 100}
              />
            </Layer>
          )}
          
          {/* Worker markers */}
          {viewMode === 'workers' && (
            <Layer>
              {getActiveWorkers(timeSliderValue).map(worker => (
                <WorkerMarker
                  key={worker.workerId}
                  worker={worker}
                  onClick={() => handleWorkerClick(worker.workerId)}
                />
              ))}
            </Layer>
          )}
        </Stage>
      </div>
      
      {/* Time controls */}
      <TimeSlider 
        value={timeSliderValue}
        onChange={setTimeSliderValue}
        startTime="07:00"
        endTime="17:00"
      />
      
      {/* Side panel */}
      {selectedZone && (
        <ZoneDetailsPanel
          zoneId={selectedZone}
          date={date}
          onVerify={(activityId) => onVerifyActivity(selectedZone, activityId)}
          onClose={() => setSelectedZone(null)}
        />
      )}
    </div>
  );
};
```

### 4. Verification Algorithm

```typescript
// services/verification.service.ts
export class VerificationService {
  async verifyWorkerActivity(
    workerId: string,
    date: Date,
    plannedActivity: ActivityAssignment
  ): Promise<VerificationResult> {
    const workerZoneTime = await this.timeAggregation.aggregateWorkerTime(
      workerId, 
      date
    );
    
    const result: VerificationResult = {
      workerId,
      date,
      totalHours: this.calculateTotalHours(workerZoneTime),
      verificationStatus: 'manual-entry',
      confidence: 0,
      zoneBreakdown: [],
      anomalies: []
    };
    
    // Calculate zone breakdown
    result.zoneBreakdown = workerZoneTime.map(zoneTime => {
      const isAssigned = this.isZoneAssignedToActivity(
        zoneTime.zoneId,
        plannedActivity
      );
      
      return {
        zoneId: zoneTime.zoneId,
        hours: zoneTime.totalMinutes / 60,
        percentage: (zoneTime.totalMinutes / result.totalHours / 60) * 100,
        isAssigned
      };
    });
    
    // Calculate confidence score
    const assignedHours = result.zoneBreakdown
      .filter(z => z.isAssigned)
      .reduce((sum, z) => sum + z.hours, 0);
    
    result.confidence = assignedHours / result.totalHours;
    
    // Determine verification status
    if (result.confidence > 0.85 && result.totalHours >= 7) {
      result.verificationStatus = 'auto-verified';
    } else if (result.confidence > 0.5) {
      result.verificationStatus = 'needs-review';
      
      // Identify anomalies
      if (result.totalHours < 6) {
        result.anomalies.push('Low total hours detected');
      }
      
      const unassignedPercentage = (1 - result.confidence) * 100;
      if (unassignedPercentage > 30) {
        result.anomalies.push(
          `${unassignedPercentage.toFixed(0)}% time in unassigned zones`
        );
      }
    }
    
    return result;
  }
}
```

### 5. Sample Data Generator

```typescript
// scripts/generateSampleData.ts
export function generateSampleData() {
  const floors = [
    { id: 'ground', name: 'Ground Floor', level: 0 },
    { id: 'level42', name: 'Level 42', level: 42 },
    { id: 'level43', name: 'Level 43', level: 43 },
    { id: 'level44', name: 'Level 44', level: 44 },
    { id: 'level51', name: 'Level 51', level: 51 }
  ];
  
  const workers = Array.from({ length: 50 }, (_, i) => ({
    id: `W${String(i + 1).padStart(3, '0')}`,
    name: generateWorkerName(),
    role: ['Carpenter', 'Electrician', 'Plumber', 'Mason'][i % 4],
    company: ['SubCon A', 'SubCon B', 'Main Contractor'][i % 3]
  }));
  
  // Generate realistic location data
  const locationData = workers.flatMap(worker => {
    const workStart = new Date('2024-01-15T07:00:00');
    const workEnd = new Date('2024-01-15T17:00:00');
    const locations: WorkerLocation[] = [];
    
    // Generate location pings every 5 minutes
    for (let time = workStart; time <= workEnd; time.setMinutes(time.getMinutes() + 5)) {
      const location = generateWorkerLocation(worker, time);
      locations.push(location);
    }
    
    return locations;
  });
  
  return { floors, workers, locationData };
}

function generateWorkerLocation(worker: any, timestamp: Date): WorkerLocation {
  // Simulate realistic movement patterns
  const hour = timestamp.getHours();
  let floorId = 'level43'; // Primary work floor
  let zoneType = 'primary';
  
  // Morning briefing
  if (hour < 8) {
    floorId = 'ground';
    zoneType = 'staging';
  }
  // Lunch break
  else if (hour === 12) {
    floorId = 'ground';
    zoneType = 'break';
  }
  // Random support activities
  else if (Math.random() < 0.2) {
    zoneType = 'support';
  }
  
  return {
    workerId: worker.id,
    timestamp,
    coordinates: generateCoordinates(floorId, zoneType),
    floorId,
    accuracy: Math.random() * 10 + 5 // 5-15m accuracy
  };
}
```

## Refinement Strategies

### 1. Machine Learning Integration

```python
# ml/pattern_recognition.py
import pandas as pd
from sklearn.cluster import DBSCAN
from sklearn.preprocessing import StandardScaler

class WorkPatternAnalyzer:
    def __init__(self):
        self.scaler = StandardScaler()
        
    def identify_work_clusters(self, location_data):
        """Identify spatial-temporal clusters of work activity"""
        # Prepare features: x, y, time_of_day
        features = self.extract_features(location_data)
        
        # Apply DBSCAN clustering
        clustering = DBSCAN(eps=10, min_samples=5).fit(features)
        
        # Identify primary work zones
        clusters = self.analyze_clusters(clustering, location_data)
        
        return clusters
    
    def detect_anomalies(self, worker_day_data, historical_patterns):
        """Detect unusual movement patterns"""
        current_pattern = self.extract_pattern(worker_day_data)
        
        # Compare with historical patterns
        similarity_scores = [
            self.calculate_similarity(current_pattern, hist)
            for hist in historical_patterns
        ]
        
        if max(similarity_scores) < 0.7:
            return {
                'is_anomaly': True,
                'confidence': 1 - max(similarity_scores),
                'description': self.describe_anomaly(current_pattern)
            }
        
        return {'is_anomaly': False}
```

### 2. Real-time Processing

```typescript
// services/realtimeProcessor.ts
import { Server } from 'socket.io';
import { Redis } from 'ioredis';

export class RealtimeLocationProcessor {
  private io: Server;
  private redis: Redis;
  private processingQueue: Map<string, WorkerLocation[]> = new Map();
  
  constructor(io: Server) {
    this.io = io;
    this.redis = new Redis();
    this.startBatchProcessor();
  }
  
  async processLocationUpdate(location: WorkerLocation) {
    // Add to processing queue
    const key = `${location.workerId}-${location.floorId}`;
    if (!this.processingQueue.has(key)) {
      this.processingQueue.set(key, []);
    }
    this.processingQueue.get(key)!.push(location);
    
    // Update real-time cache
    await this.updateWorkerCache(location);
    
    // Emit to connected clients
    this.io.to(`floor-${location.floorId}`).emit('worker-update', {
      workerId: location.workerId,
      location: location.coordinates,
      timestamp: location.timestamp
    });
  }
  
  private startBatchProcessor() {
    setInterval(async () => {
      for (const [key, locations] of this.processingQueue) {
        if (locations.length > 0) {
          await this.processBatch(locations);
          this.processingQueue.set(key, []);
        }
      }
    }, 5000); // Process every 5 seconds
  }
}
```

## Testing & Deployment

### 1. Unit Tests

```typescript
// tests/verification.test.ts
describe('VerificationService', () => {
  it('should auto-verify workers with >85% assigned zone time', async () => {
    const mockWorkerData = generateMockWorkerDay({
      assignedZoneTime: 7.5,
      totalTime: 8.5
    });
    
    const result = await verificationService.verifyWorkerActivity(
      'W001',
      new Date('2024-01-15'),
      mockActivity
    );
    
    expect(result.verificationStatus).toBe('auto-verified');
    expect(result.confidence).toBeGreaterThan(0.85);
  });
  
  it('should flag workers with excessive unassigned time', async () => {
    const mockWorkerData = generateMockWorkerDay({
      assignedZoneTime: 3,
      totalTime: 8
    });
    
    const result = await verificationService.verifyWorkerActivity(
      'W002',
      new Date('2024-01-15'),
      mockActivity
    );
    
    expect(result.verificationStatus).toBe('needs-review');
    expect(result.anomalies).toContain('62% time in unassigned zones');
  });
});
```

### 2. Docker Deployment

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy application
COPY . .

# Build TypeScript
RUN npm run build

# Expose ports
EXPOSE 3000

# Start application
CMD ["node", "dist/server.js"]
```

## Next Steps for Claude Code

1. **Set up the project structure** with the provided architecture
2. **Implement the zone mapping algorithm** using the spatial queries
3. **Create the interactive blueprint UI** with React and Konva.js
4. **Build the verification pipeline** with confidence scoring
5. **Add real-time updates** using WebSockets
6. **Integrate with actual IoT data** feeds
7. **Implement batch verification** workflows
8. **Add export functionality** for approved reports

This system will transform manual timesheet entry into a visual, intuitive verification process that foremen can complete in minutes instead of hours.