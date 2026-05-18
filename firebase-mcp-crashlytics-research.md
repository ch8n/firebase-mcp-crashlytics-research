# Firebase MCP Crashlytics PR Agent

A comprehensive technical guide for building an autonomous CI/CD agent that polls Firebase Crashlytics for crashes and ANRs, analyzes them against your codebase, and opens PRs against your development branch with fix recommendations.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CRASHLYTICS PR AGENT                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐ │
│  │   Cron Job   │───▶│  Firebase    │───▶│    LLM      │───▶│   Git    │ │
│  │  (schedule) │    │  MCP Server  │    │  (analyze)  │    │   API    │ │
│  └──────────────┘    └──────────────┘    └──────────────┘    └──────────┘ │
│         │                  │                   │                   │       │
│         │         ┌────────┴────────┐         │         ┌────────┴────┐ │
│         │         │  Crashlytics API │         │         │   GitHub    │ │
│         │         │  (crashes/ANRs)  │         │         │   REST      │ │
│         │         └──────────────────┘         │         └─────────────┘ │
│         │                                        │                       │
│         │         ┌──────────────────┐           │                       │
│         │         │  Deduplication  │           │                       │
│         │         │  (SQLite/JSON)  │           │                       │
│         │         └──────────────────┘           │                       │
│         │                                        │                       │
│  ┌──────┴──────┐                         ┌────────┴───────┐               │
│  │  Scheduler  │                         │  PR on dev    │               │
│  │  (cronjob) │                         │  branch        │               │
│  └────────────┘                         └────────────────┘               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Firebase MCP Server Setup

### 1.1 Prerequisites

| Requirement | Command/Link |
|-------------|--------------|
| Node.js ≥20 | `node --version` |
| npm or pnpm | `npm --version` |
| Firebase CLI | `npm install -g firebase-tools` |
| Google Cloud project with Crashlytics enabled | [Firebase Console](https://console.firebase.google.com) |
| Service account with Crashlytics Viewer role | [IAM Console](https://console.cloud.google.com/iam-admin/iam) |

### 1.2 Install Firebase MCP Server

```bash
# Create project directory
mkdir -p ~/crashlytics-pr-agent && cd ~/crashlytics-pr-agent
npm init -y

# Install Firebase MCP server
npm install @anthropic-ai/claude-code firebase-mcp-server --save

# Or install from source (latest)
npm install firebase-mcp-server@latest --save
```

### 1.3 Configure Firebase MCP Server

Create `mcp-server.json` in your project root:

```json
{
  "mcpServers": {
    "firebase-crashlytics": {
      "command": "npx",
      "args": ["firebase-mcp-server"],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "./service-account.json",
        "FIREBASE_PROJECT_ID": "your-project-id"
      }
    }
  }
}
```

### 1.4 Create Service Account

1. Go to [Google Cloud Console → IAM & Admin → Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts)
2. Create service account → name it `crashlytics-mcp-reader`
3. Grant role: **Crashlytics Viewer** (or `roles/cloudcrashlytics.viewer`)
4. Create JSON key → download as `service-account.json`
5. Place in project root

```bash
# Secure the credentials
chmod 600 service-account.json
```

### 1.5 Verify MCP Server Connection

```bash
cd ~/crashlytics-pr-agent
npx firebase-mcp-server --project your-project-id
```

You should see available tools logged:

```
[info] Firebase MCP Server started
[info] Available tools:
  - firebase_crashlytics_get_crashes
  - firebase_crashlytics_get_anrs
  - firebase_crashlytics_get_issues
  - firebase_crashlytics_list_projects
```

---

## Phase 2: Claude Code / Claude CLI Setup

### 2.1 Install Claude Code

```bash
# macOS / Linux
curl -fsSL https://claude.com/install.sh | sh

# Verify
claude --version
```

### 2.2 Configure Claude Code with MCP

Create `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "firebase-crashlytics": {
      "command": "npx",
      "args": ["firebase-mcp-server"],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "/home/ch8n/crashlytics-pr-agent/service-account.json",
        "FIREBASE_PROJECT_ID": "your-project-id"
      }
    }
  }
}
```

### 2.3 Test Firebase MCP Tools

```bash
claude -p "List all recent crashes from Firebase Crashlytics using firebase_crashlytics_get_crashes"
```

Expected output includes crash ID, stack trace, affected users, timestamp.

---

## Phase 3: Deduplication Strategy

### 3.1 Problem: Duplicate PRs

The agent runs on a schedule. If the same crash/ANR persists across runs, it will create duplicate PRs. You need a persistent tracker.

### 3.2 Implementation: SQLite Database

```bash
# Create deduplication database
sqlite3 crashlytics_tracker.db <<EOF
CREATE TABLE IF NOT EXISTS processed_issues (
    issue_id TEXT PRIMARY KEY,
    issue_type TEXT CHECK(issue_type IN ('crash', 'anr', 'velocity', 'stability')),
    title TEXT,
    first_seen TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    pr_number INTEGER,
    pr_url TEXT,
    status TEXT DEFAULT 'open'
);

CREATE TABLE IF NOT EXISTS issue_signatures (
    issue_id TEXT PRIMARY KEY,
    stack_hash TEXT,
    package_name TEXT,
    exception_class TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_issue_type ON processed_issues(issue_type);
CREATE INDEX idx_pr_status ON processed_issues(status);
EOF
```

### 3.3 Deduplication Logic (Python)

Create `src/dedup.py`:

```python
#!/usr/bin/env python3
"""Deduplication handler for Crashlytics issues."""

import sqlite3
import hashlib
import logging
from datetime import datetime, timedelta
from typing import Optional

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

DB_PATH = "crashlytics_tracker.db"


def compute_signature(issue: dict) -> str:
    """Generate deterministic signature from stack trace.
    
    Uses package + exception class + first 3 stack frames.
    """
    package = issue.get("package", "")
    exception = issue.get("exceptionClass", "")
    stack = issue.get("stackTrace", {})
    frames = stack.get("frames", [])[:3]  # First 3 frames
    
    sig_parts = [package, exception]
    for frame in frames:
        sig_parts.append(frame.get("method", ""))
        sig_parts.append(str(frame.get("line", "")))
    
    return hashlib.sha256("|".join(sig_parts).encode()).hexdigest()


def is_duplicate(issue_id: str, signature: str) -> bool:
    """Check if issue was already processed."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # Check by issue ID (exact match)
    cursor.execute(
        "SELECT 1 FROM processed_issues WHERE issue_id = ?",
        (issue_id,)
    )
    if cursor.fetchone():
        conn.close()
        return True
    
    # Check by signature (similar crash pattern)
    cursor.execute(
        "SELECT 1 FROM issue_signatures WHERE stack_hash = ?",
        (signature,)
    )
    result = cursor.fetchone()
    conn.close()
    
    return result is not None


def mark_processed(
    issue_id: str,
    issue_type: str,
    title: str,
    signature: str,
    pr_number: Optional[int] = None,
    pr_url: Optional[str] = None
) -> None:
    """Mark issue as processed with PR details."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute(
        """INSERT OR REPLACE INTO processed_issues 
           (issue_id, issue_type, title, pr_number, pr_url, status)
           VALUES (?, ?, ?, ?, ?, 'open')""",
        (issue_id, issue_type, title, pr_number, pr_url)
    )
    
    cursor.execute(
        """INSERT OR REPLACE INTO issue_signatures 
           (issue_id, stack_hash, package_name, exception_class)
           VALUES (?, ?, ?, ?)""",
        (
            issue_id,
            signature,
            issue_id.split(".")[0] if "." in issue_id else "",
            signature[:16]  # partial
        )
    )
    
    conn.commit()
    conn.close()
    logger.info(f"Marked issue {issue_id} as processed")


def get_open_prs() -> list:
    """Get all issues with open PRs."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute(
        """SELECT issue_id, issue_type, title, pr_number, pr_url 
           FROM processed_issues 
           WHERE status = 'open' AND pr_number IS NOT NULL"""
    )
    
    results = [
        {
            "issue_id": row[0],
            "issue_type": row[1],
            "title": row[2],
            "pr_number": row[3],
            "pr_url": row[4]
        }
        for row in cursor.fetchall()
    ]
    
    conn.close()
    return results


def close_stale_prs(days: int = 7) -> int:
    """Mark PRs as closed if they're older than N days.
    
    Returns count of closed PRs.
    """
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute(
        """UPDATE processed_issues 
           SET status = 'closed'
           WHERE status = 'open' 
           AND pr_number IS NOT NULL
           AND first_seen < datetime('now', '-' || ? || ' days')""",
        (days,)
    )
    
    affected = cursor.rowcount
    conn.commit()
    conn.close()
    
    if affected:
        logger.info(f"Closed {affected} stale PRs")
    
    return affected


if __name__ == "__main__":
    # Test
    test_issue = {
        "id": "5f8a9b2c3d4e5f6a",
        "package": "com.example.app",
        "exceptionClass": "NullPointerException",
        "stackTrace": {
            "frames": [
                {"method": "onClick", "line": 42},
                {"method": "handleClick", "line": 101},
                {"method": "dispatchEvent", "line": 55}
            ]
        }
    }
    
    sig = compute_signature(test_issue)
    print(f"Signature: {sig}")
```

### 3.4 Run Cleanup Periodically

```bash
# Add to crontab - run weekly
crontab -e

# Add: every Sunday at 2am
0 2 * * 0 cd ~/crashlytics-pr-agent && python3 src/dedup.py --cleanup
```

---

## Phase 4: The PR Agent Script

### 4.1 Directory Structure

```
crashlytics-pr-agent/
├── src/
│   ├── __init__.py
│   ├── fetch_crashes.py      # Fetch from Firebase MCP
│   ├── fetch_anrs.py         # Fetch ANRs
│   ├── analyzer.py           # Analyze crashes with LLM
│   ├── pr_creator.py        # Create GitHub PR
│   └── dedup.py              # Deduplication (above)
├── scripts/
│   └── run_agent.sh          # Main entrypoint
├── config.yaml               # Configuration
├── requirements.txt          # Python deps
└── mcp-settings.json        # Claude MCP config
```

### 4.2 Install Python Dependencies

```bash
pip install pyyaml requests python-dotenv github3.py
```

### 4.3 Configuration (`config.yaml`)

```yaml
firebase:
  project_id: "your-project-id"
  service_account_path: "./service-account.json"

github:
  owner: "your-org"
  repo: "your-app"
  dev_branch: "develop"
  base_branch: "main"
  token_env: "GITHUB_TOKEN"
  auto_merge: false

agent:
  schedule: "0 */6 * * *"  # Every 6 hours
  max_issues_per_run: 5
  max_pr_age_days: 7
  min_crash_count: 3
  min_user_impact: 2
  skip_fatal: false  # Always process fatal crashes

paths:
  work_dir: "/home/ch8n/crashlytics-pr-agent"
  db_path: "./crashlytics_tracker.db"
  sources_path: "/path/to/android/source/code"
```

### 4.4 Fetch Crashes (`src/fetch_crashes.py`)

```python
#!/usr/bin/env python3
"""Fetch crashes and ANRs from Firebase Crashlytics via MCP."""

import json
import subprocess
import logging
from typing import List, Dict, Any

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def call_firebase_mcp(tool_name: str, **kwargs) -> List[Dict[str, Any]]:
    """Call Firebase MCP tool via Claude Code."""
    
    # Build the tool call prompt
    if tool_name == "firebase_crashlytics_get_crashes":
        prompt = f"""Use firebase_crashlytics_get_crashes to get recent crashes.
        Parameters: {json.dumps(kwargs)}
        Return the results as JSON."""
    elif tool_name == "firebase_crashlytics_get_anrs":
        prompt = f"""Use firebase_crashlytics_get_anrs to get recent ANRs.
        Parameters: {json.dumps(kwargs)}
        Return the results as JSON."""
    else:
        raise ValueError(f"Unknown tool: {tool_name}")
    
    # Call Claude Code with MCP
    result = subprocess.run(
        ["claude", "-p", f"{prompt} --print-only"],
        capture_output=True,
        text=True,
        timeout=60
    )
    
    if result.returncode != 0:
        logger.error(f"MCP call failed: {result.stderr}")
        return []
    
    try:
        return json.loads(result.stdout)
    except json.JSONDecodeError:
        logger.error(f"Failed to parse JSON: {result.stdout[:500]}")
        return []


def fetch_crashes(
    min_impact: int = 2,
    time_range_hours: int = 24
) -> List[Dict[str, Any]]:
    """Fetch crashes from last N hours with minimum user impact."""
    
    logger.info(f"Fetching crashes from last {time_range_hours}h")
    
    crashes = call_firebase_mcp(
        "firebase_crashlytics_get_crashes",
        projectId="your-project-id",
        timeRange={"value": time_range_hours, "unit": "HOURS"}
    )
    
    # Filter by impact
    filtered = [
        c for c in crashes
        if c.get("impactedDevices", 0) >= min_impact
    ]
    
    logger.info(f"Found {len(filtered)} crashes with ≥{min_impact} impacted devices")
    return filtered


def fetch_anrs(
    min_impact: int = 2,
    time_range_hours: int = 24
) -> List[Dict[str, Any]]:
    """Fetch ANRs from last N hours."""
    
    logger.info(f"Fetching ANRs from last {time_range_hours}h")
    
    anrs = call_firebase_mcp(
        "firebase_crashlytics_get_anrs",
        projectId="your-project-id",
        timeRange={"value": time_range_hours, "unit": "HOURS"}
    )
    
    filtered = [
        a for a in anrs
        if a.get("impactedDevices", 0) >= min_impact
    ]
    
    logger.info(f"Found {len(filtered)} ANRs with ≥{min_impact} impacted devices")
    return filtered


if __name__ == "__main__":
    crashes = fetch_crashes()
    anrs = fetch_anrs()
    
    print(f"Crashes: {len(crashes)}")
    print(f"ANRs: {len(anrs)}")
```

### 4.5 Analyze with LLM (`src/analyzer.py`)

```python
#!/usr/bin/env python3
"""Analyze crashes using LLM to generate fix recommendations."""

import os
import json
import subprocess
import logging
from typing import Dict, Any, List
from dataclasses import dataclass

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class FixRecommendation:
    root_cause: str
    fix_description: str
    files_to_modify: List[str]
    suggested_code: str
    confidence: float


def analyze_crash(crash: Dict[str, Any], codebase_path: str) -> FixRecommendation:
    """Use LLM to analyze crash and generate fix."""
    
    # Extract key crash info
    issue_id = crash.get("id", "unknown")
    exception = crash.get("exceptionClass", "Unknown")
    stack_trace = crash.get("stackTrace", {}).get("content", "")[:2000]
    package = crash.get("package", "")
    
    prompt = f"""You are a senior Android/Java developer analyzing a crash from Firebase Crashlytics.

## Crash Information
- Issue ID: {issue_id}
- Exception: {exception}
- Package: {package}

## Stack Trace
```
{stack_trace}
```

## Task
1. Analyze the stack trace to identify the root cause
2. Look at the codebase at {codebase_path} to find the relevant source files
3. Generate a fix with code suggestions
4. Output as JSON with this exact structure:
{{
  "root_cause": "One sentence describing why this crash occurs",
  "fix_description": "2-3 sentences on how to fix it",
  "files_to_modify": ["Relative path to file 1", "Relative path to file 2"],
  "suggested_code": "The code changes to apply",
  "confidence": 0.0-1.0
}}

Respond ONLY with valid JSON, no other text."""

    # Call LLM (using Claude Code or external API)
    result = subprocess.run(
        ["claude", "-p", prompt, "--print-only"],
        capture_output=True,
        text=True,
        timeout=120,
        env={**os.environ, "CLAUDE_MODEL": "claude-sonnet-4-20250514"}
    )
    
    if result.returncode != 0:
        logger.error(f"LLM analysis failed: {result.stderr}")
        return FixRecommendation(
            root_cause="Analysis failed",
            fix_description="Could not generate fix",
            files_to_modify=[],
            suggested_code="",
            confidence=0.0
        )
    
    try:
        data = json.loads(result.stdout)
        return FixRecommendation(**data)
    except (json.JSONDecodeError, TypeError) as e:
        logger.error(f"Failed to parse LLM response: {e}")
        return FixRecommendation(
            root_cause="Parse error",
            fix_description="Could not parse LLM output",
            files_to_modify=[],
            suggested_code="",
            confidence=0.0
        )


def analyze_anr(anr: Dict[str, Any], codebase_path: str) -> FixRecommendation:
    """Analyze ANR (Application Not Responding)."""
    
    issue_id = anr.get("id", "unknown")
    stack_trace = anr.get("stackTrace", {}).get("content", "")[:2000]
    main_thread = stack_trace  # ANRs are always on main thread
    
    prompt = f"""You are a senior Android developer analyzing an ANR (Application Not Responding) from Firebase Crashlytics.

## ANR Information
- Issue ID: {issue_id}

## Blocking Stack Trace
```
{main_thread}
```

## Task
1. Identify what's blocking the main thread
2. Check the codebase at {codebase_path} for the blocking operation
3. Recommend moving to background thread / using proper async patterns
4. Output as JSON:
{{
  "root_cause": "What operation is blocking",
  "fix_description": "How to move to background thread",
  "files_to_modify": ["file paths"],
  "suggested_code": "Use Handler, Coroutines, or ExecutorService",
  "confidence": 0.0-1.0
}}

Respond ONLY with valid JSON."""

    result = subprocess.run(
        ["claude", "-p", prompt, "--print-only"],
        capture_output=True,
        text=True,
        timeout=120
    )
    
    try:
        data = json.loads(result.stdout)
        return FixRecommendation(**data)
    except Exception as e:
        logger.error(f"ANR analysis failed: {e}")
        return FixRecommendation(
            root_cause="Analysis failed",
            fix_description="",
            files_to_modify=[],
            suggested_code="",
            confidence=0.0
        )
```

### 4.6 Create GitHub PR (`src/pr_creator.py`)

```python
#!/usr/bin/env python3
"""Create GitHub PR with fix branch."""

import os
import logging
from datetime import datetime
from typing import Optional
import github3

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class PRCreator:
    def __init__(self, token: str, owner: str, repo: str):
        self.gh = github3.login(token=token)
        self.owner = owner
        self.repo = repo
        self.repository = self.gh.repository(owner, repo)
    
    def create_fix_branch(
        self,
        issue_id: str,
        base_branch: str = "develop"
    ) -> Optional[str]:
        """Create a new branch for the fix."""
        
        # Get base branch reference
        ref = self.repository.ref(f"refs/heads/{base_branch}")
        if not ref:
            logger.error(f"Base branch {base_branch} not found")
            return None
        
        branch_name = f"fix/crashlytics-{issue_id[:8]}-{datetime.now().strftime('%Y%m%d')}"
        
        # Create new branch
        try:
            self.repository.create_ref(
                f"refs/heads/{branch_name}",
                ref.object.sha
            )
            logger.info(f"Created branch: {branch_name}")
            return branch_name
        except github3.exceptions.MethodNotAllowed:
            logger.warning(f"Branch {branch_name} already exists, using existing")
            return branch_name
    
    def apply_fix(
        self,
        branch: str,
        file_path: str,
        content: str,
        commit_message: str
    ) -> bool:
        """Apply fix to a file on the branch."""
        
        try:
            # Get current file SHA if exists
            try:
                existing = self.repository.file_contents(
                    file_path,
                    ref=f"refs/heads/{branch}"
                )
                sha = existing.sha
            except github3.exceptions.NotFoundError:
                sha = None
            
            # Create or update file
            self.repository.file_contents(
                f"{file_path}",
                content=content,
                message=commit_message,
                branch=branch,
                sha=sha
            )
            
            logger.info(f"Applied fix to {file_path}")
            return True
            
        except Exception as e:
            logger.error(f"Failed to apply fix: {e}")
            return False
    
    def create_pr(
        self,
        branch: str,
        title: str,
        body: str,
        dev_branch: str = "develop"
    ) -> Optional[int]:
        """Create PR against development branch."""
        
        try:
            pr = self.repository.create_pull(
                title=title,
                body=body,
                base=dev_branch,
                head=branch
            )
            
            logger.info(f"Created PR #{pr.number}: {pr.html_url}")
            return pr.number
            
        except Exception as e:
            logger.error(f"Failed to create PR: {e}")
            return None


def create_crash_fix_pr(
    crash: dict,
    fix: "FixRecommendation",
    pr_creator: PRCreator,
    dev_branch: str = "develop"
) -> Optional[int]:
    """Full workflow: branch → fix → PR."""
    
    issue_id = crash.get("id", "unknown")
    title = f"[Crashlytics] Fix {crash.get('exceptionClass', 'crash')}: {fix.root_cause[:50]}"
    
    # 1. Create branch
    branch = pr_creator.create_fix_branch(issue_id, dev_branch)
    if not branch:
        return None
    
    # 2. Apply each file fix
    for file_path, code in zip(fix.files_to_modify, [fix.suggested_code]):
        pr_creator.apply_fix(
            branch=branch,
            file_path=file_path,
            content=code,
            commit_message=f"Fix {crash.get('exceptionClass')}: {fix.root_cause}"
        )
    
    # 3. Create PR
    body = f"""## Crashlytics Issue

- **Issue ID**: `{issue_id}`
- **Type**: {crash.get('issueType', 'crash')}
- **Impacted Devices**: {crash.get('impactedDevices', 'N/A')}
- **First Seen**: {crash.get('firstSeen', 'N/A')}

## Root Cause

{fix.root_cause}

## Fix Description

{fix.fix_description}

## Files Modified

{chr(10).join(f'- {f}' for f in fix.files_to_modify)}

## Suggested Code

```java
{fix.suggested_code}
```

---

*This PR was auto-generated by the Crashlytics PR Agent.*"""
    
    return pr_creator.create_pr(branch, title, body, dev_branch)
```

### 4.7 Main Agent Entry Point (`scripts/run_agent.sh`)

```bash
#!/bin/bash
# Main entry point for Crashlytics PR Agent
# Run via cron: 0 */6 * * * /path/to/run_agent.sh

set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_DIR="$(dirname "$SCRIPT_DIR")"

cd "$PROJECT_DIR"

# Load config
export $(grep -v '^#' config.yaml | xargs)

# Check GitHub token
if [ -z "$GITHUB_TOKEN" ]; then
    echo "ERROR: GITHUB_TOKEN not set"
    exit 1
fi

echo "=== Starting Crashlytics PR Agent ==="
echo "Time: $(date)"

# Step 1: Fetch crashes and ANRs
echo "[1/4] Fetching crashes..."
python3 -c "
from src.fetch_crashes import fetch_crashes, fetch_anrs
import json

crashes = fetch_crashes(min_impact=$AGENT_MIN_IMPACT)
anrs = fetch_anrs(min_impact=$AGENT_MIN_IMPACT)

with open('/tmp/issues.json', 'w') as f:
    json.dump({'crashes': crashes, 'anrs': anrs}, f)
"

# Step 2: Check dedup, filter new issues
echo "[2/4] Checking for duplicates..."
python3 -c "
import json
from src.dedup import is_duplicate, compute_signature

with open('/tmp/issues.json') as f:
    data = json.load(f)

new_issues = []
for crash in data.get('crashes', []):
    sig = compute_signature(crash)
    if not is_duplicate(crash['id'], sig):
        crash['issue_type'] = 'crash'
        new_issues.append(crash)

for anr in data.get('anrs', []):
    sig = compute_signature(anr)
    if not is_duplicate(anr['id'], sig):
        anr['issue_type'] = 'anr'
        new_issues.append(anr)

print(f'Found {len(new_issues)} new issues')

with open('/tmp/new_issues.json', 'w') as f:
    json.dump(new_issues, f)
"

# Step 3: Analyze and create PRs
echo "[3/4] Analyzing and creating PRs..."
python3 -c "
import json
from src.analyzer import analyze_crash, analyze_anr
from src.pr_creator import PRCreator, create_crash_fix_pr
from src.dedup import mark_processed, compute_signature
import os

with open('/tmp/new_issues.json') as f:
    issues = json.load(f)

# Limit per run
issues = issues[:5]

gh = PRCreator(
    token=os.environ['GITHUB_TOKEN'],
    owner='$GITHUB_OWNER',
    repo='$GITHUB_REPO'
)

results = []
for issue in issues:
    try:
        if issue['issue_type'] == 'crash':
            fix = analyze_crash(issue, '$AGENT_SOURCES_PATH')
        else:
            fix = analyze_anr(issue, '$AGENT_SOURCES_PATH')
        
        if fix.confidence < 0.5:
            print(f'Skipping low-confidence fix for {issue[\"id\"]}')
            continue
        
        pr_num = create_crash_fix_pr(issue, fix, gh, '$GITHUB_DEV_BRANCH')
        
        if pr_num:
            sig = compute_signature(issue)
            mark_processed(
                issue['id'],
                issue['issue_type'],
                issue.get('exceptionClass', 'Unknown'),
                sig,
                pr_number=pr_num
            )
            results.append(pr_num)
    except Exception as e:
        print(f'Error processing {issue.get(\"id\")}: {e}')

print(f'Created {len(results)} PRs')
"

echo "[4/4] Done"
echo "=== Crashlytics PR Agent Complete ==="
```

### 4.8 Make Executable

```bash
chmod +x scripts/run_agent.sh
```

---

## Phase 5: Cron Job Scheduling

### 5.1 Create Cron Job

```bash
crontab -e

# Add: every 6 hours
0 */6 * * * cd /home/ch8n/crashlytics-pr-agent && ./scripts/run_agent.sh >> /var/log/crashlytics-pr-agent.log 2>&1
```

### 5.2 Verify Cron

```bash
# Check cron is running
service cron status

# List crontab
crontab -l

# Manual test
./scripts/run_agent.sh
```

---

## Phase 6: GitHub Actions CI/CD Integration

### 6.1 Workflow File

Create `.github/workflows/crashlytics-agent.yml`:

```yaml
name: Crashlytics PR Agent

on:
  schedule:
    # Run every 6 hours
    - cron: '0 */6 * * *'
  # Allow manual trigger
  workflow_dispatch:

jobs:
  crashlytics-pr:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          npm install -g firebase-mcp-server
      
      - name: Configure Firebase
        env:
          FIREBASE_CREDENTIALS: ${{ secrets.FIREBASE_CREDENTIALS }}
        run: |
          echo "$FIREBASE_CREDENTIALS" > service-account.json
      
      - name: Run PR Agent
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          FIREBASE_PROJECT_ID: ${{ secrets.FIREBASE_PROJECT_ID }}
        run: |
          ./scripts/run_agent.sh
      
      - name: Cleanup old PRs
        run: |
          python3 -c "from src.dedup import close_stale_prs; close_stale_prs(7)"
```

### 6.2 Required Secrets

| Secret | Description |
|--------|-------------|
| `GITHUB_TOKEN` | GitHub classic PAT with `repo` scope |
| `FIREBASE_CREDENTIALS` | Base64-encoded service account JSON |
| `FIREBASE_PROJECT_ID` | Your Firebase project ID |

---

## Phase 7: Deduplication Edge Cases

### 7.1 Handling All Duplicate Scenarios

| Scenario | Handling |
|----------|----------|
| Same issue_id across runs | Skip — already in `processed_issues` |
| Similar crash (different version) | Match by `stack_hash` signature |
| Same crash, different device type | Signature uses method + line only |
| Issue re-opened after fix | Create new branch with incremented suffix |
| PR closed manually | Re-run analysis on next cycle |
| PR merged, same crash returns | New entry since `status = 'closed'` allows re-run |

### 7.2 Enhanced Deduplication Logic

```python
# src/dedup.py — add these functions

def should_skip_issue(issue: dict) -> tuple[bool, str]:
    """Complete deduplication check.
    
    Returns (skip: bool, reason: str)
    """
    issue_id = issue.get("id", "")
    signature = compute_signature(issue)
    
    # Exact issue_id match
    if is_duplicate(issue_id, signature):
        return True, f"Issue {issue_id} already processed"
    
    # Check for existing open PR
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    cursor.execute(
        """SELECT pr_number, pr_url FROM processed_issues 
           WHERE issue_id = ? AND status = 'open'""",
        (issue_id,)
    )
    row = cursor.fetchone()
    conn.close()
    
    if row:
        return True, f"Open PR #{row[0]} already exists"
    
    return False, ""


def allow_reopen(issue_id: str) -> bool:
    """Check if a closed issue can be reopened."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # Check if it was merged/closed > 7 days ago
    cursor.execute(
        """SELECT status, first_seen FROM processed_issues 
           WHERE issue_id = ?""",
        (issue_id,)
    )
    row = cursor.fetchone()
    conn.close()
    
    if not row:
        return True
    
    status, first_seen = row
    if status == "closed":
        # Reopen if closed > 7 days ago
        from datetime import datetime
        closed_date = datetime.fromisoformat(first_seen)
        return (datetime.now() - closed_date).days > 7
    
    return True
```

---

## Phase 8: Testing

### 8.1 Unit Tests

```bash
pip install pytest pytest-mock

# test_dedup.py
import pytest
from src.dedup import compute_signature, is_duplicate, should_skip_issue

def test_signature_deterministic():
    issue = {
        "id": "test.123",
        "package": "com.example",
        "exceptionClass": "NullPointerException",
        "stackTrace": {"frames": [{"method": "onClick", "line": 42}]}
    }
    sig1 = compute_signature(issue)
    sig2 = compute_signature(issue)
    assert sig1 == sig2

def test_different_crashes_different_signatures():
    issue1 = {"id": "1", "package": "com.a", "exceptionClass": "NPE", "stackTrace": {"frames": [{"method": "a", "line": 1}]}}
    issue2 = {"id": "2", "package": "com.b", "exceptionClass": "IOE", "stackTrace": {"frames": [{"method": "b", "line": 2}]}}
    assert compute_signature(issue1) != compute_signature(issue2)

def test_should_skip_duplicate():
    issue = {"id": "existing.issue", "package": "x", "exceptionClass": "Y", "stackTrace": {"frames": []}}
    skip, reason = should_skip_issue(issue)
    assert skip == True
    assert "already processed" in reason
```

### 8.2 Integration Test

```bash
# Create mock crash and test full flow
python3 -c "
import json

mock_crash = {
    'id': 'test.crash.001',
    'exceptionClass': 'NullPointerException',
    'package': 'com.example.app',
    'impactedDevices': 5,
    'stackTrace': {
        'content': 'at com.example.MainActivity.onCreate(MainActivity.java:42)'
    }
}

with open('/tmp/mock_crash.json', 'w') as f:
    json.dump(mock_crash, f)

print('Mock crash saved. Run agent to test.')
"
```

---

## Complete File Structure

```
crashlytics-pr-agent/
├── .github/
│   └── workflows/
│       └── crashlytics-agent.yml
├── src/
│   ├── __init__.py
│   ├── fetch_crashes.py
│   ├── fetch_anrs.py
│   ├── analyzer.py
│   ├── pr_creator.py
│   └── dedup.py
├── scripts/
│   └── run_agent.sh
├── tests/
│   ├── test_dedup.py
│   └── test_analyzer.py
├── config.yaml
├── requirements.txt
├── service-account.json
└── README.md
```

---

## Summary

| Phase | What You Build |
|-------|----------------|
| 1 | Firebase MCP server with Crashlytics access |
| 2 | Claude Code configured with MCP tools |
| 3 | SQLite deduplication with signature matching |
| 4 | Python scripts: fetch → analyze → PR |
| 5 | Cron job for periodic runs |
| 6 | GitHub Actions workflow |
| 7 | Edge case handling for duplicates |
| 8 | Unit + integration tests |