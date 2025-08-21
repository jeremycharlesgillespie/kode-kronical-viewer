# CLAUDE.md

This file provides guidance to Claude Code when working with the kode-kronical-viewer codebase.

## Project Overview

kode-kronical-viewer is a Django web dashboard for visualizing performance data from the kode-kronical library. It displays system metrics through a web UI, using DynamoDB for data storage.

## Key Architecture Decisions

### Data Flow
1. **kode-kronical-daemon** (background process) → collects system metrics → uploads to DynamoDB
2. **DynamoDB** → stores compressed metrics data in `kode-kronical-system` table
3. **Django backend** → reads from DynamoDB → serves API endpoints
4. **Vue.js frontend** → polls API every 2 minutes → displays real-time data

### Key Files and Their Purposes

#### Backend (Django)
- `kodekronicalweb/dashboard/system_services.py` - Core service for reading system data from DynamoDB
  - `get_recent_system_data()` - Retrieves recent metrics with parallel scanning
  - `get_system_dashboard_data()` - Aggregates data for dashboard view
  - Online/offline threshold: 6 minutes (360 seconds)

- `kodekronicalweb/dashboard/views.py` - Django views and API endpoints
- `kodekronicalweb/dashboard/api_views.py` - REST API views for frontend

#### Frontend (Vue.js)
- `src/views/SystemOverview.vue` - Main system monitoring page
- `src/stores/system.js` - Pinia store for system data management
- Polling interval: 2 minutes (120000ms)

#### Daemon
- `/Users/gman/code/kode-kronical-all/kode-kronical/kode-kronical-daemon` - System monitoring daemon
- Config: `/Users/gman/.config/kode-kronical/daemon.yaml`
- Uploads to DynamoDB every 60 seconds
- Uses system Python with boto3 installed via --user

### Optimized Data Collection (v2)
- **1-minute intervals**: Daemon collects data every 60 seconds (no more continuous sampling)
- **Frontend-ready format**: Data stored in structure that requires minimal Django processing
- **Hour-based partitioning**: Efficient DynamoDB queries using `hostname#YYYY-MM-DD-HH` keys
- **Automatic TTL**: All records expire after 30 days to control storage costs
- **Real-time access**: Sub-minute latency with optimized table structure

### Dual Service Architecture
- **Legacy service**: `system_services.py` - handles old compressed data format
- **Optimized service**: `optimized_system_service.py` - handles new v2 table format
- **Automatic fallback**: Django tries optimized first, falls back to legacy
- **Table detection**: Uses `kode-kronical-system` for optimized, `kode-kronical-system` for legacy

### Common Issues and Solutions

1. **"Last Update showing old data"**
   - Check daemon is running: `ps aux | grep kode-kronical-daemon`
   - Check daemon logs: `/Users/gman/.local/share/kode-kronical/logs/daemon.log`
   - Verify DynamoDB uploads in logs

2. **"System showing offline"**
   - Systems show offline if last update > 6 minutes old
   - Check DynamoDB for recent records
   - May be due to daemon not running or missing dependencies

3. **Django not finding recent data**
   - Uses parallel scanning across 8-16 segments
   - ConsistentRead disabled for performance
   - May experience eventual consistency delays

### Testing Commands

```bash
# Check if daemon is uploading
grep "Successfully uploaded" /Users/gman/.local/share/kode-kronical/logs/daemon.log | tail -5

# Test Django service
cd /Users/gman/code/kode-kronical-all/kode-kronical-viewer
DJANGO_SETTINGS_MODULE=kodekronicalweb.settings .venv/bin/python -c "
from kodekronicalweb.dashboard.system_services import system_data_service
recent = system_data_service.get_recent_system_data(hours=1, limit=5)
print(f'Found {len(recent)} records')
"

# Check latest DynamoDB record
aws dynamodb scan --table-name kode-kronical-system --limit 1 --output json | jq '.Items[0].timestamp.N'

# Check for hostname duplicates (IMPORTANT for dashboard accuracy)
.venv/bin/python ../scripts/check_hostname_duplicates.py
```

### Virtual Environments
- Django app: `.venv/` 
- Daemon: Uses system Python with --user packages