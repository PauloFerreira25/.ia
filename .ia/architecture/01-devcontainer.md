---
name: devcontainer
description: "Read before making any changes to .devcontainer, .devcontainer/docker-compose.yml, or VSCode settings."
---

# Devcontainer Architecture

## docker-compose.yml

`.devcontainer/docker-compose.yml` is exclusively a local development environment file. It exists to support the devcontainer.

Never treat it as a production or staging docker-compose file.
Never apply production concerns (security hardening, resource limits, restart policies, secrets management) to it.

---

## Data Volumes

All data volumes must use the hidden directory `.data-volumes` at the project root.

When a container needs one volume:
```
.data-volumes/<containerName>
```

When a container needs multiple volumes:
```
.data-volumes/<containerName>/<volumeName>
```

Examples:
```
container: bancoDeDados
volume: .data-volumes/bancoDeDados

container: sistemaX
volumes:
  .data-volumes/sistemaX/volume1
  .data-volumes/sistemaX/volume2
```

Never create data volumes outside `.data-volumes`.

## Service Files

All files belonging to a service (Dockerfile, configuration files, etc.) must live under `.devcontainer/container/<serviceName>/`.

```
.devcontainer/container/<serviceName>/Dockerfile
.devcontainer/container/<serviceName>/<config-file>
```

Example:
```
.devcontainer/container/nginx/Dockerfile
.devcontainer/container/nginx/nginx.conf

.devcontainer/container/postgres/Dockerfile
```

Never place service-specific files directly in `.devcontainer/` or outside their service directory.

## VSCode Configuration

All VSCode settings, extensions, and configurations must be defined in `.devcontainer/devcontainer.json`.

Never create or modify files in the `.vscode/` directory for this purpose.

This ensures VSCode configuration is tied to the containerized environment and consistent across all developers using the devcontainer.
