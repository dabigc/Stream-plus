# Rules Directory

This directory contains all persistent configuration and state for Stream Plus.

## Files

- `auto_assignment_rules.json` - Auto-assignment rules configuration
- `sorting_rules.json` - Stream sorting rules configuration
- `global_rule_settings.json` - Global exclusion patterns and settings
- `channel_groups.json` - Channel groups definitions (created when groups are defined)
- `execution_state.json` - Last execution state tracking (created on first rule execution)
- `m3u_refresh_state.json` - M3U refresh timestamp tracking (created on first M3U refresh)

## Docker Volume Mounting

When running Stream Plus in Docker, this directory is mounted as a volume to persist configuration across container restarts and updates.

```yaml
volumes:
  - ./rules:/app/rules
```

All configuration files in this directory will be preserved when:
- Updating the Docker image
- Restarting the container
- Rebuilding the container

## Backup

To backup your Stream Plus configuration, simply backup this `rules/` directory.
