# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Stream Plus is a Flask web application for managing Dispatcharr channels and streams with intelligent auto-assignment and stream sorting capabilities. It provides a web interface and CLI for creating rules that automatically assign IPTV streams to channels and sort them based on quality metrics.

**Core Technologies:**
- Backend: Flask (Python 3.11)
- Frontend: Bootstrap 5 with vanilla JavaScript
- Deployment: Docker with Alpine Linux base
- External API: Dispatcharr API (requires authentication)

## Development Commands

### Running the Application

```bash
# Local development (after setting up .env)
python app.py

# Docker development build
docker-compose -f docker-compose.dev.yml up -d

# Docker production
docker-compose up -d
```

### Testing

The project does not have a formal test suite. Manual testing is done through:
- Web interface at http://localhost:5000
- CLI rule execution with `execute_rules.py`

### CLI Rule Execution

```bash
# Execute all enabled rules (both assignment and sorting)
python execute_rules.py --all

# Execute only auto-assignment rules
python execute_rules.py --assignment

# Execute only sorting rules
python execute_rules.py --sorting

# Execute specific rule IDs with verbose output
python execute_rules.py --all --rule-ids 1 2 3 --verbose

# Docker execution
docker exec stream-plus python execute_rules.py --all
```

## Architecture

### Core Components

1. **Flask Application** (`app.py` - 26k+ tokens, read in chunks)
   - Main web server with SSE (Server-Sent Events) for real-time updates
   - Routes for auto-assignment rules, sorting rules, and channel groups
   - Background task execution for long-running operations
   - Profile management integration with Dispatcharr

2. **Data Models** (`models.py`)
   - `AutoAssignmentRule`: Rules for automatically assigning streams to channels
   - `RulesManager`: Persistence and CRUD operations for assignment rules
   - `StreamMatcher`: Evaluation engine for matching streams against rule conditions
   - Supports migration logic from older rule formats (e.g., v0.3.2 → v0.3.3)

3. **Stream Sorter Models** (`stream_sorter_models.py`)
   - `SortingRule`: Rules for scoring and ordering streams within channels
   - `SortingCondition`: Individual scoring conditions (M3U source, bitrate, codec, etc.)
   - `ChannelGroup`: Grouping of channels for easier rule management
   - `StreamSorter`: Scoring engine that evaluates conditions and sorts streams

4. **Dispatcharr Client** (`api/dispatcharr_client.py`)
   - JWT authentication with token refresh
   - Comprehensive API wrapper for Dispatcharr endpoints
   - Stream testing using ffmpeg/ffprobe (see line 643+)
   - Automatic pagination for large datasets

5. **Rule Execution CLI** (`execute_rules.py`)
   - Standalone script for automated rule execution
   - M3U source refresh before rule execution
   - Supports both assignment and sorting rules
   - Detailed logging and error handling

### Data Flow

1. **Auto-Assignment Flow:**
   - User creates rule with conditions (regex, resolution, codec, bitrate, etc.)
   - Rule optionally tests streams with ffmpeg/ffprobe to get real statistics
   - StreamMatcher evaluates streams against all conditions
   - Matching streams are assigned to the specified channel
   - Channel profiles can be enabled/disabled based on match results

2. **Stream Sorting Flow:**
   - User creates sorting rule with scoring conditions
   - Each condition awards points when met (e.g., h265 codec = 10 points)
   - StreamSorter calculates total score for each stream
   - Streams are reordered in channel by score (highest first)
   - Works on specific channels, channel groups, or all channels

3. **Profile Management Logic (v0.3.3+):**
   - When streams match a rule: Enable channel in assigned profiles (or all if none selected)
   - When no streams match: Disable channel in ALL profiles
   - This is a major inversion from the old "protected profiles" logic

### Key Design Patterns

- **Background Processing:** Long operations (rule execution, stream testing) run in threads with SSE progress updates
- **Forced Includes/Excludes:** Manual overrides in `AutoAssignmentRule.force_include_stream_ids` and `force_exclude_stream_ids`
- **Resolution Normalization:** Streams are normalized to 720p, 1080p, 2160p, or SD for consistent filtering
- **Stream Statistics:** ffmpeg/ffprobe extract real stream data (bitrate, codec, fps, resolution, pixel format)

## Important Implementation Details

### Stream Testing

Stream testing (models.py:336+ and dispatcharr_client.py:643+) uses ffmpeg/ffprobe to analyze streams:
- **ffprobe first:** Quick check if stream is accessible and extract metadata
- **ffmpeg second:** Read stream for configured duration to calculate actual bitrate
- User-agent spoofing to prevent provider detection (configurable via `STREAM_TEST_USER_AGENT`)
- Failed tests clear stream statistics to prevent stale data
- Configurable test duration (`STREAM_TEST_DURATION`, default 10 seconds)
- Delay between tests (`STREAM_TEST_DELAY`, default 3 seconds) to avoid provider rate limiting

### Rule Evaluation Logic

**Auto-Assignment Rules:**
1. Basic filtering: regex pattern + M3U account filter
2. Stream testing (if enabled): Get real statistics for matching streams
3. Condition evaluation: Check all conditions (bitrate, codec, resolution, fps, pixel format)
4. Apply force includes/excludes
5. Assign to channel and update profile status

**Sorting Rules:**
1. Get all streams from target channels
2. Test streams (if enabled) to refresh statistics
3. Evaluate each condition against each stream, accumulate points
4. Sort by total score (descending)
5. Update channel stream order via Dispatcharr API

### API Quirks

- Dispatcharr API may return lists directly or paginated objects with `results` key
- Channel updates require full channel object (PUT) or partial data (PATCH)
- Stream order in channels: Use PATCH with `{"streams": [id1, id2, ...]}` first, fall back to PUT
- M3U refresh is a POST to `/api/m3u/refresh/`

## Configuration

Required environment variables (see `.env.example`):
```
DISPATCHARR_API_URL=http://127.0.0.1:9191
DISPATCHARR_API_USER=user
DISPATCHARR_API_PASSWORD=password
```

Optional variables:
```
PORT=5000
FLASK_DEBUG=false
SECRET_KEY=change-this-secret-key-in-production
STREAM_TEST_DURATION=10
STREAM_TEST_TIMEOUT_BUFFER=30
STREAM_TEST_DELAY=3
STREAM_TEST_USER_AGENT=Mozilla/5.0...
CACHE_TTL=300
TZ=UTC
```

### Performance Optimization

**Caching (added in performance optimization update):**
- Channel groups and Dispatcharr statistics are cached for `CACHE_TTL` seconds (default: 300 = 5 minutes)
- Cache reduces repeated expensive API calls to Dispatcharr (452 channels + 895 groups + 54k streams)
- Lower `CACHE_TTL` = more up-to-date data but more API load
- Higher `CACHE_TTL` = better performance but potentially stale data
- Cache is automatically bypassed when explicitly refreshing (e.g., `/api/channel-groups` endpoint)
- Implementation in `stream_sorter_models.py:578+` and `app.py:120+`

## File Locations

- **Rules persistence:** `auto_assignment_rules.json`, `sorting_rules.json` (in `/app/rules` in Docker)
- **Channel groups:** `channel_groups.json` (loaded from Dispatcharr API or local file)
- **M3U refresh state:** `m3u_refresh_state.json` (tracks last refresh timestamp)
- **Execution state:** `execution_state.json` (tracks background task progress)
- **Templates:** `templates/` directory with Jinja2 HTML templates
- **Static files:** `static/` directory with CSS, JavaScript, and images

## Common Patterns

### Reading Large Files

`app.py` is over 26k tokens. Use offset/limit parameters or grep to read specific sections:
```python
# Read specific route
Read(file_path="/Users/.../app.py", offset=500, limit=100)

# Search for function
Grep(pattern="def execute_auto_assignment_in_background", path="app.py")
```

### Adding New Rule Conditions

1. Add field to `AutoAssignmentRule` or `SortingCondition` dataclass
2. Update `_stream_matches_rule()` or `_evaluate_condition()` evaluation logic
3. Add UI inputs in templates (modal forms)
4. Update JavaScript to handle new field in AJAX requests
5. Test with preview functionality before saving

### Migration Logic

When adding new fields or changing data models:
1. Add migration code in `from_dict()` methods (see models.py:90-118)
2. Provide default values for missing fields
3. Test loading old rule files to ensure backward compatibility
4. Document breaking changes in CHANGELOG.md

## Version History Notes

- **v0.3.3:** Major profile logic inversion (protected → assigned profiles)
- **v0.3.0:** Added pixel format support and manual stream overrides
- **v0.2.7:** CLI M3U refresh integration
- **v0.2.0:** Added stream test delay to prevent provider detection

When working with older codebases or issues, check CHANGELOG.md for context on past changes and known issues.
