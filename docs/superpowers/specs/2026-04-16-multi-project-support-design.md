# Multi-Project Support for Octogent

**Date**: 2026-04-16  
**Status**: Design  
**Approach**: Option 1 — Registry-Based  

---

## Overview

Expand octogent from managing a single project (octogent itself) to orchestrating agents across multiple independent projects (octogent + Trojan + future projects). The design uses a lightweight registry and keeps project state local to maintain autonomy while providing a unified developer experience.

---

## Goals

1. Enable octogent to manage tentacles and agents for multiple separate codebases
2. Provide a single dashboard for switching between projects
3. Keep each project's state autonomous (local `.octogent/` folder)
4. Support CLI-based project registration and activation
5. Maintain backward compatibility for single-project mode

---

## Design

### 1. Registry & State Layout

**Central registry**: `~/.octogent/registry.json`

```json
{
  "projects": [
    {
      "id": "octogent",
      "name": "Octogent",
      "path": "/Users/vincentagwu/Documents/GitHub/octogent",
      "lastActive": "2026-04-16T18:45:00Z"
    },
    {
      "id": "trojan",
      "name": "Trojan",
      "path": "/Users/vincentagwu/Documents/GitHub/Trojan",
      "lastActive": null
    }
  ],
  "activeProject": "octogent"
}
```

**Per-project state**: Each project maintains its own `.octogent/` folder:
- `/Users/vincentagwu/Documents/GitHub/octogent/.octogent/state/tentacles.json`
- `/Users/vincentagwu/Documents/GitHub/octogent/.octogent/state/transcripts/`
- `/Users/vincentagwu/Documents/GitHub/octogent/.octogent/worktrees/`
- And similarly for `/Users/vincentagwu/Documents/GitHub/Trojan/.octogent/`

Each project's state is fully isolated and independent.

### 2. API Changes

**New project management endpoints**:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/projects` | List all registered projects + activeProject |
| POST | `/api/projects` | Register new project (validates path, `.octogent` exists) |
| PATCH | `/api/projects/{projectId}/activate` | Set activeProject, update lastActive |
| DELETE | `/api/projects/{projectId}` | Unregister project (keeps files) |

**Scoped tentacle endpoints**:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/projects/{projectId}/tentacles` | List tentacles for project |
| POST | `/api/projects/{projectId}/tentacles` | Create tentacle in project |
| GET | `/api/projects/{projectId}/tentacles/{tentacleId}/todo` | Read todo.md |
| POST | `/api/projects/{projectId}/tentacles/{tentacleId}/todo` | Update todo.md |
| + others | `/api/projects/{projectId}/...` | All existing tentacle operations scoped by projectId |

**Backward compatibility**:
- Existing endpoints (e.g., `/api/tentacles`) remain and operate on `activeProject`
- Single-project mode (no registry) continues to work unchanged

### 3. UI Changes

**Project selector dropdown**:
- Location: Top-left of dashboard (near logo)
- Displays current active project name
- Click to open dropdown listing all registered projects
- Selecting a project: calls `PATCH /api/projects/{projectId}/activate`, re-renders tentacles
- No page reload; data context switches in-place

**Session behavior**:
- On app load: fetch `/api/projects`, read `activeProject`, load that project's tentacles
- Dropdown selection updates `activeProject` in registry + local React state
- Re-fetch tentacles for selected project; keep all other UI unchanged

**No other UI modifications** — tentacle list, todo view, and terminal management remain identical; only the data context changes.

### 4. CLI & Initialization

**New commands**:

```bash
octogent init <project-path>
# Initialize octogent for a new project
# - Creates .octogent/ in <project-path>
# - Adds entry to ~/.octogent/registry.json
# - Sets as activeProject

octogent list
# Print all registered projects and activeProject

octogent use <project-id>
# Set activeProject in registry

octogent start [--project <id>]
# Start octogent server
# - If --project given, set as activeProject first
# - Otherwise use lastActive from registry
# - In single-project mode, auto-detects .octogent/
```

**Backward compatibility**:
- Running `octogent` in a git repo with `.octogent/` auto-detects single-project mode (no registry needed)
- Existing workflows continue unchanged

### 5. Documentation & README Updates

**Update `README.md`**:
- Add "Multi-Project Support" section explaining registry concept
- Add CLI examples: `octogent init`, `octogent list`, `octogent use`
- Update "How It Works" to mention project switching via UI dropdown

**New guide**: `docs/guides/managing-multiple-projects.md`
- Registry structure and purpose
- Step-by-step: registering a new project
- UI walkthrough: switching projects
- CLI reference: all new commands
- Limitations in v1 (no cross-project tentacle communication)
- Migration from single-project mode

**Update reference docs**:
- `docs/reference/filesystem-layout.md` — add registry.json diagram
- `docs/reference/api.md` — document `/api/projects/*` endpoints

---

## Trade-offs

| Aspect | Choice | Why |
|--------|--------|-----|
| State isolation | Local per-project | Projects remain autonomous, easier to backup/move |
| Project discovery | Central registry | Single source of truth for all projects |
| Switching performance | Re-fetch tentacles | Simple to implement; projects are typically infrequent switches |
| Future scalability | Can upgrade to Option 2 | Registry-based foundation supports unified index later |
| Backward compatibility | Full | Single-project mode works unchanged |

---

## Implementation Order

1. **Create registry schema** + file I/O utilities
2. **Update API layer** — add `/api/projects/*` endpoints, make existing routes project-aware
3. **Update UI** — add project dropdown, wire to project switching
4. **Add CLI commands** — `init`, `list`, `use`, update `start`
5. **Write documentation** — README updates, new guide, reference updates
6. **Test multi-project workflow** — register octogent + trojan, verify switching

---

## Success Criteria

- [ ] Registry file created and read correctly
- [ ] Can register a new project via CLI
- [ ] Can switch projects via UI dropdown
- [ ] Each project's tentacles load independently when switched
- [ ] Backward compatibility: single-project mode still works
- [ ] CLI commands (`octogent init`, `list`, `use`) work as specified
- [ ] Documentation updated with multi-project examples

---

## Future Enhancements (Not in Scope)

- Cross-project tentacle messaging (agents in octogent coordinating with agents in trojan)
- Unified search across all projects
- Project templates / quick setup
- Web UI for project registration (currently CLI-only)
