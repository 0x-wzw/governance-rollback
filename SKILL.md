---
name: governance-rollback
version: 1.0.0
description: "Approval gates with versioned configuration changes and safe rollback. Adapted from Paperclip's governance principle."
author: October (10D Entity)
keywords: [governance, rollback, approval, versioning, safety, config]
---

# Governance with Rollback 🛡️

> **Approval gates + versioned changes + safe rollback**
> 
> *Adapted from Paperclip's governance safety principle*

## Overview

Critical changes require governance:
- **Approval Gates** — Changes need explicit approval
- **Versioning** — All changes tracked with history
- **Rollback** — Safe reversion to previous state
- **Audit Trail** — Who changed what, when

## Architecture

```
Change Request
    ↓
Approval Gate (human or automated)
    ↓
[APPROVED] → Version + Apply
    ↓
Monitor Impact
    ↓
[FAILURE] → Rollback to previous version
```

## Change Categories

| Category | Examples | Approval Required |
|----------|----------|-------------------|
| **Critical** | Model routing changes, security rules | Yes (human) |
| **Standard** | Skill updates, config tweaks | Yes (auto) |
| **Low Risk** | Documentation, comments | No |

## Usage

### Request Change

```python
from governance import ChangeRequest

request = ChangeRequest(
    category="critical",
    description="Update model router to use ClawRouter",
    changes={
        "file": "~/.openclaw/openclaw.json",
        "patch": {...}
    },
    requester="October"
)

# Submit for approval
request_id = governance.submit(request)
# Returns: "CHG-2026-0322-001"
```

### Approve Change

```python
# Human approval (via Telegram/Discord)
governance.approve("CHG-2026-0322-001", approver="Z")

# Or auto-approval for standard changes
if request.risk_score < 0.3:
    governance.auto_approve(request_id)
```

### Rollback on Failure

```python
# Monitor after change
if monitor.detect_degradation():
    # Automatic rollback
    governance.rollback("CHG-2026-0322-001")
    
    # Or manual rollback
    governance.rollback_to_version(42)
```

## Implementation

### Versioned Config Store

```python
class VersionedConfig:
    """Configuration with versioning and rollback."""
    
    def __init__(self, config_path: str):
        self.path = Path(config_path)
        self.versions_dir = self.path.parent / ".versions"
        self.versions_dir.mkdir(exist_ok=True)
    
    def save_version(self, description: str, approver: str = None):
        """Save current config as new version."""
        version_id = len(list(self.versions_dir.iterdir())) + 1
        timestamp = datetime.now().isoformat()
        
        version = {
            "id": version_id,
            "timestamp": timestamp,
            "description": description,
            "approver": approver,
            "hash": self._calculate_hash()
        }
        
        # Copy config to version
        version_path = self.versions_dir / f"v{version_id}.json"
        shutil.copy(self.path, version_path)
        
        # Save metadata
        meta_path = self.versions_dir / f"v{version_id}.meta.json"
        meta_path.write_text(json.dumps(version, indent=2))
        
        return version_id
    
    def rollback(self, version_id: int):
        """Rollback to specific version."""
        version_path = self.versions_dir / f"v{version_id}.json"
        
        if not version_path.exists():
            raise VersionNotFoundError(version_id)
        
        # Backup current
        self.save_version("Pre-rollback backup", approver="system")
        
        # Restore version
        shutil.copy(version_path, self.path)
        
        log(f"Rolled back to version {version_id}")
    
    def list_versions(self) -> List[Dict]:
        """List all versions with metadata."""
        versions = []
        for meta_file in sorted(self.versions_dir.glob("*.meta.json")):
            versions.append(json.loads(meta_file.read_text()))
        return versions
```

### Approval Gate

```python
class ApprovalGate:
    """Enforce approval before critical changes."""
    
    def __init__(self, config: Dict):
        self.required_approvers = config.get("approvers", ["Z"])
        self.auto_approve_threshold = config.get("auto_threshold", 0.3)
    
    def evaluate(self, change: ChangeRequest) -> ApprovalResult:
        """Evaluate if change needs approval."""
        
        # Calculate risk score
        risk = self._calculate_risk(change)
        
        if risk < self.auto_approve_threshold:
            return ApprovalResult(
                approved=True,
                method="auto",
                reason="Low risk"
            )
        
        # Require human approval
        return ApprovalResult(
            approved=False,
            method="manual",
            reason=f"High risk: {risk}",
            requires_approval_from=self.required_approvers
        )
```

## Integration

### In Critical Changes

```python
def update_model_router(new_config: Dict):
    """Update requires governance."""
    
    # Create change request
    request = ChangeRequest(
        category="critical",
        description="Update model routing",
        changes=new_config
    )
    
    # Submit to governance
    result = governance.evaluate(request)
    
    if not result.approved:
        notify("Change requires approval", result.requires_approval_from)
        return False
    
    # Apply with version
    version = config_store.save_version("Model router update")
    
    try:
        apply_config(new_config)
        monitor(60)  # Monitor for 60 seconds
        return True
    except Exception as e:
        config_store.rollback(version)
        raise
```

## Comparison to Paperclip

| Aspect | Paperclip | Swarm Governance |
|--------|-----------|-----------------|
| **Approval** | Human gates | Human + auto thresholds |
| **Versioning** | Config changes | File-based versions |
| **Rollback** | Manual | Manual + auto on failure |
| **Audit** | Basic logging | Full version history |
| **Integration** | Org structure | Token economy aware |

---

*Approval gates with versioned changes and safe rollback*
*Adapted from Paperclip's governance safety principle*
