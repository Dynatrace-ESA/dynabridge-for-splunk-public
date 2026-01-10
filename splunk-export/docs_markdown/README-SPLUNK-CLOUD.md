# DynaBridge Splunk Cloud Export Script
## Prerequisites Guide for Splunk Cloud (Classic & Victoria Experience)

**Version**: 4.1.0
**Last Updated**: January 2026
**Related Documents**: [Script-Generated Analytics Reference](SCRIPT-GENERATED-ANALYTICS-REFERENCE.md) | [Cloud Export Specification](SPLUNK-CLOUD-EXPORT-SPECIFICATION.md)

---

> **Developed for Dynatrace One by Enterprise Solutions & Architecture**
> *An ACE Services Division of Dynatrace*

---

## Quick Start

```bash
# This script runs from YOUR machine (not on Splunk Cloud)
# You need network access to your Splunk Cloud instance

./dynabridge-splunk-cloud-export.sh
```

---

## How This Differs from Enterprise Export

| Aspect | Enterprise Script | Cloud Script (This One) |
|--------|------------------|-------------------------|
| **Where you run it** | ON the Splunk server | ANYWHERE (your laptop, jump host) |
| **Access method** | SSH + File system | REST API only |
| **What you need** | SSH access + splunk user | Network access + API credentials |
| **File reading** | Reads props.conf, etc. | Reconstructs from REST API |

---

## Prerequisites Checklist

### 1. Network Access

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         NETWORK REQUIREMENTS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Your machine must be able to reach:                                     │
│                                                                          │
│    https://<your-stack>.splunkcloud.com:8089                            │
│                                                                          │
│  This is the Splunk Cloud REST API management port.                     │
│                                                                          │
│  TEST IT:                                                                │
│  $ curl -I https://acme-corp.splunkcloud.com:8089/services/server/info  │
│                                                                          │
│  If this fails, check:                                                   │
│  • Corporate firewall rules                                              │
│  • VPN requirements                                                      │
│  • Splunk Cloud IP allowlist settings                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2. Splunk Cloud Credentials

You need ONE of the following:

#### Option A: API Token (Recommended)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CREATING AN API TOKEN                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Log into Splunk Cloud web UI                                        │
│     https://your-stack.splunkcloud.com                                  │
│                                                                          │
│  2. Click Settings (gear icon) → Tokens                                 │
│                                                                          │
│  3. Click "New Token"                                                   │
│                                                                          │
│  4. Configure the token:                                                │
│     • Name: DynaBridge Export Token                                     │
│     • Expiration: Set appropriate (e.g., 7 days)                        │
│     • Audience: Search (if asked)                                       │
│                                                                          │
│  5. Copy the token value (shown only once!)                             │
│                                                                          │
│  6. Store it securely - you'll need it for the export script            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Option B: Username/Password

- Your Splunk Cloud admin username and password
- If MFA is required, you may need to create an API token instead

### 3. Required Permissions

The user or token must have these Splunk capabilities:

| Capability | Required? | What It's Used For |
|------------|-----------|-------------------|
| `admin_all_objects` | **Required** | Access all apps and knowledge objects |
| `list_users` | **Required** | Collect user information |
| `list_roles` | **Required** | Collect role definitions |
| `search` | **Required** | Run usage analytics queries |
| `rest_access` | **Required** | Make REST API calls |
| `list_indexes` | Recommended | Get index metadata |

#### Checking Your Permissions

Run this in Splunk Cloud search:
```spl
| rest /services/authentication/current-context
| table username, roles, capabilities
```

### 4. Local Machine Requirements

| Requirement | Purpose | Check Command |
|-------------|---------|---------------|
| `bash` 4.0+ | Script execution | `bash --version` |
| `curl` | REST API calls | `curl --version` |
| `Python 3` | JSON parsing | `python3 --version` |
| Disk space | Store export | 500MB+ free |

---

## Splunk Cloud Stack URL

### Finding Your Stack URL

Your Splunk Cloud stack URL is the address you use to access Splunk Cloud:

```
https://<stack-name>.splunkcloud.com
```

**Examples**:
- `https://acme-corp.splunkcloud.com`
- `https://mycompany-prod.splunkcloud.com`
- `https://enterprise1.splunkcloud.com`

### Testing Connectivity

```bash
# Test if you can reach the REST API
curl -I "https://your-stack.splunkcloud.com:8089/services/server/info"

# Expected: HTTP/2 401 (Unauthorized - but reachable)
# If you get connection refused or timeout, check network/firewall
```

---

## Supported Splunk Cloud Types

| Cloud Type | Supported | Notes |
|------------|-----------|-------|
| Splunk Cloud Classic | ✅ Yes | Legacy multi-tenant |
| Splunk Cloud Victoria Experience | ✅ Yes | Current default |
| Splunk Cloud on AWS | ✅ Yes | Single-tenant |
| Splunk Cloud on GCP | ✅ Yes | Single-tenant |
| Splunk Cloud on Azure | ✅ Yes | Single-tenant |

---

## What Data Can Be Collected

### ✅ Fully Available via REST API

| Data Type | REST Endpoint | Notes |
|-----------|---------------|-------|
| Dashboards | `/data/ui/views` | Classic + Dashboard Studio |
| Saved Searches | `/saved/searches` | Includes alerts, reports |
| Users | `/authentication/users` | Full user list |
| Roles | `/authorization/roles` | With capabilities |
| Macros | `/admin/macros` | All search macros |
| Eventtypes | `/saved/eventtypes` | Event classifications |
| Tags | `/configs/conf-tags` | Tag assignments |
| Lookup Definitions | `/data/lookup-table-files` | File metadata |
| Lookup Contents | Download via REST | CSV data |
| Field Extractions | `/data/transforms/extractions` | Regex extractions |
| Apps List | `/apps/local` | Installed apps |
| Index Settings | `/data/indexes` | Index configuration |

### ⚠️ Partially Available

| Data Type | Limitation | Workaround |
|-----------|------------|------------|
| Props.conf | No file access | Reconstructed from `/configs/conf-props` |
| Transforms.conf | No file access | Reconstructed from `/configs/conf-transforms` |
| Usage Analytics | Requires search | Run searches on `_audit` index |
| Index Sizes | Limited stats | Best-effort from API |

### ❌ Not Available

| Data Type | Why | Impact |
|-----------|-----|--------|
| Raw config files | No file system | Use REST reconstruction |
| $SPLUNK_HOME access | Cloud infrastructure | N/A |
| Audit.log file | No file system | Use `_audit` index search |
| License file | Cloud-managed | N/A |
| Deployment apps | Cloud-managed | N/A |

---

## IP Allowlisting (If Required)

Some Splunk Cloud instances require IP allowlisting for API access:

### Check If Required

Contact your Splunk Cloud admin or check:
- Splunk Cloud Admin Config (if you have access)
- Cloud Stack settings

### Adding Your IP

1. Log into Splunk Cloud Admin Config
2. Go to IP Allowlist settings
3. Add your machine's public IP:
   ```bash
   # Find your public IP
   curl ifconfig.me
   ```
4. Allow port 8089 (REST API)

---

## Running the Script

### Basic Usage

```bash
# Make executable
chmod +x dynabridge-splunk-cloud-export.sh

# Run interactively
./dynabridge-splunk-cloud-export.sh
```

### With Pre-set Values

```bash
# Set stack URL via environment
export SPLUNK_CLOUD_STACK="acme-corp.splunkcloud.com"

# Set token via environment (more secure than command line)
export SPLUNK_CLOUD_TOKEN="your-api-token"

./dynabridge-splunk-cloud-export.sh
```

### Non-Interactive Mode (for automation)

```bash
./dynabridge-splunk-cloud-export.sh \
  --stack "acme-corp.splunkcloud.com" \
  --token "$SPLUNK_CLOUD_TOKEN" \
  --all-apps \
  --output /path/to/output
```

---

## Command-Line Arguments (Updated in v4.1.0)

| Argument | Description | Example |
|----------|-------------|---------|
| `--stack` | Splunk Cloud stack URL | `--stack acme.splunkcloud.com` |
| `--token` | API token for authentication | `--token "xxxxx"` |
| `--user` | Username (if not using token) | `--user admin` |
| `--password` | Password (if not using token) | `--password "xxx"` |
| `--apps` | Comma-separated list of apps **(NEW v4.1.0)** | `--apps "search,myapp"` |
| `--all-apps` | Export all applications (default) | `--all-apps` |
| `--quick` | Quick mode - skip analytics **(TESTING ONLY - see warning)** | `--quick` |
| `--scoped` | Scope collections to selected apps only **(NEW v4.1.0)** | `--scoped` |
| `--no-usage` | Skip usage analytics collection | `--no-usage` |
| `--skip-internal` | Skip searches requiring _internal index | `--skip-internal` |
| `--output` | Output directory | `--output /path/to/output` |
| `-d, --debug` | Enable verbose debug logging **(NEW v4.1.0)** | `--debug` |
| `--help` | Show help message | `--help` |

### App-Scoped Export Mode (NEW in v4.1.0)

For large Splunk Cloud environments, dramatically reduce export time by targeting specific apps:

```bash
# Export only specific apps (fastest option)
./dynabridge-splunk-cloud-export.sh \
  --stack acme.splunkcloud.com \
  --token "$TOKEN" \
  --apps "search,myapp,security_essentials" \
  --quick

# Scoped mode - exports app configs + only users/searches related to those apps
./dynabridge-splunk-cloud-export.sh \
  --stack acme.splunkcloud.com \
  --token "$TOKEN" \
  --apps "myapp,otherapp" \
  --scoped
```

| Mode | What It Does | Use When |
|------|-------------|----------|
| `--quick` | App configs only, no global analytics | **Testing/validation only** - NOT for migration analysis |
| `--scoped` | App configs + app-filtered users/usage | You want usage data but only for selected apps |
| (default) | Full export of all apps + global analytics | **Recommended** - Full migration analysis |

> **⚠️ CRITICAL WARNING: Do NOT use `--quick` for Migration Analysis**
>
> The `--quick` flag is intended **ONLY for testing and script validation**, not for actual migration planning. Using `--quick` eliminates critical data needed for migration analysis:
>
> - **Usage Analytics**: Who uses which dashboards/alerts, how often, and when last accessed
> - **User & RBAC Data**: Migration audience identification, role mappings, permission structures
> - **Search Activity**: Which saved searches are actively used vs. abandoned
> - **Priority Assessment**: Data needed to determine migration priority and phasing
>
> **Without this data, you cannot:**
> - Identify which assets are actually being used vs. unused/abandoned
> - Understand who your migration audiences are
> - Prioritize which dashboards/alerts to migrate first
> - Make informed decisions about what may or may not be needed
>
> **Always use the default (full) export or `--scoped` for any export intended for migration analysis.**

### Debug Mode (NEW in v4.1.0)

When troubleshooting issues, enable debug mode to capture detailed logs:

```bash
./dynabridge-splunk-cloud-export.sh \
  --stack acme.splunkcloud.com \
  --token "$TOKEN" \
  --apps myapp \
  --debug
```

Debug mode provides:
- **Console output**: Color-coded messages by category (API, SEARCH, TIMING, ERROR, WARN)
- **Debug log file**: `export_debug.log` inside the export directory (included in the .tar.gz)
- **API call tracking**: Every REST API call with HTTP status and response size
- **Detailed timing**: Duration of each API call and search operation

---

## Enterprise Resilience Features

**NEW in v4.0.0**: The Cloud script now includes the same enterprise-scale features as the Enterprise script for environments with 4000+ dashboards and 10K+ alerts.

### Default Settings (Enterprise-Ready)

| Setting | Default | Description |
|---------|---------|-------------|
| `BATCH_SIZE` | 100 | Items per API request |
| `API_TIMEOUT` | 120s | Per-request timeout (2 min) |
| `MAX_TOTAL_TIME` | 14400s | Max runtime (4 hours) |
| `MAX_RETRIES` | 3 | Retry attempts with exponential backoff |
| `RATE_LIMIT_DELAY` | 0.1s | Delay between API calls (100ms) |
| `CHECKPOINT_ENABLED` | true | Enable checkpoint/resume capability |

### Checkpoint/Resume Capability

If the export is interrupted (timeout, network error, Ctrl+C), you can resume:

```bash
# Script detects previous incomplete export
./dynabridge-splunk-cloud-export.sh

# Output:
# Found checkpoint from 2025-01-06 14:30:00
# Would you like to resume? (Y/n): Y
# Resuming from: Dashboards (offset 500)...
```

### Export Timing Statistics

At completion, the script shows detailed timing:

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    EXPORT TIMING STATISTICS                              ║
╠══════════════════════════════════════════════════════════════════════════╣
║  Total Duration:        5 minutes 4 seconds                              ║
║  API Calls:             347                                              ║
║  API Retries:           2                                                ║
║  API Failures:          0                                                ║
║  Rate Limit Hits:       0                                                ║
║  Batches Completed:     52                                               ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### Environment Variable Overrides

For very large Splunk Cloud environments, tune via environment variables:

```bash
# Large environment (5000+ dashboards)
export BATCH_SIZE=50
export API_TIMEOUT=180
./dynabridge-splunk-cloud-export.sh

# Or inline
BATCH_SIZE=50 API_TIMEOUT=180 ./dynabridge-splunk-cloud-export.sh
```

---

## Troubleshooting

### Connection Refused

```
Error: curl: (7) Failed to connect to acme-corp.splunkcloud.com port 8089
```

**Solutions**:
1. Check if you're on VPN (if required)
2. Verify the stack URL is correct
3. Check corporate firewall rules
4. Verify Splunk Cloud IP allowlist includes your IP

### Authentication Failed (401)

```
Error: HTTP 401 Unauthorized
```

**Solutions**:
1. Verify credentials are correct
2. Check if token has expired
3. Try creating a new token
4. Verify user account is active

### Forbidden (403)

```
Error: HTTP 403 Forbidden for /services/authentication/users
```

**Solutions**:
1. User/token lacks required capabilities
2. Add `admin_all_objects` capability
3. Check role assignments
4. Some Cloud stacks restrict certain APIs

### Rate Limited (429)

```
Error: HTTP 429 Too Many Requests
```

**Solutions**:
1. Script will automatically back off and retry
2. If persistent, wait 5 minutes and try again
3. Contact Splunk Cloud support for limit increases

### SSL Certificate Error

```
Error: SSL certificate problem: unable to get local issuer certificate
```

**Solutions**:
1. Update CA certificates: `update-ca-certificates`
2. Script uses `-k` flag as fallback (warns user)
3. Download Splunk Cloud CA cert and specify

---

## Security Best Practices

### Token Security

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TOKEN SECURITY CHECKLIST                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ✓ Use API tokens instead of passwords                                  │
│  ✓ Set appropriate token expiration (7-30 days)                         │
│  ✓ Don't share tokens in chat, email, or tickets                       │
│  ✓ Use environment variables, not command-line args                    │
│  ✓ Delete token after export is complete                               │
│  ✓ Don't commit tokens to version control                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Export File Security

```bash
# Export file contains sensitive metadata
# Handle with appropriate care

# Secure transfer
scp export.tar.gz user@secure-server:/path/

# Delete after upload to DynaBridge
rm export.tar.gz
```

---

## What Gets Exported

The export creates a `.tar.gz` file compatible with DynaBridge containing:

```
dynabridge_cloud_export_[stack]_[timestamp]/
├── dynasplunk-env-summary.md      # Summary report
├── _metadata.json                  # Export metadata
├── _systeminfo/                    # Server info
├── _rbac/                         # Users and roles
├── _configs/                      # Reconstructed configs
├── _usage_analytics/              # Usage data
├── [app_name]/                    # Per-app data
│   ├── dashboards/
│   ├── savedsearches.json
│   └── macros.json
└── dashboard_studio/              # Dashboard Studio content
```

---

## Comparison with Enterprise Export

| Feature | Enterprise Export | Cloud Export |
|---------|------------------|--------------|
| Dashboards | ✅ Complete | ✅ Complete |
| Alerts | ✅ Complete | ✅ Complete |
| Users/RBAC | ✅ Complete | ✅ Complete |
| Props/Transforms | ✅ File-based | ⚠️ REST reconstruction |
| Usage Analytics | ✅ Audit log + search | ⚠️ Search only |
| Index Stats | ✅ Complete | ⚠️ Limited |
| Lookup Contents | ✅ Direct file | ✅ REST download |
| Custom Scripts | ✅ bin/ directory | ❌ Not accessible |
| Export Format | .tar.gz | .tar.gz (compatible) |

---

## Frequently Asked Questions

### Q: Can I run this on my laptop?

**A: Yes!** That's exactly where you should run it. You just need network access to your Splunk Cloud instance.

### Q: Do I need SSH access to anything?

**A: No.** This script is 100% REST API based. No SSH required.

### Q: Will this work with MFA enabled?

**A: Use an API token.** MFA typically doesn't apply to API token authentication.

### Q: How long does the export take?

**A: 5-30 minutes** depending on the size of your environment and network speed. Large environments with many dashboards may take longer.

### Q: Can I schedule this to run automatically?

**A: Yes.** Use the non-interactive mode with environment variables:
```bash
export SPLUNK_CLOUD_STACK="your-stack.splunkcloud.com"
export SPLUNK_CLOUD_TOKEN="your-token"
./dynabridge-splunk-cloud-export.sh --all-apps --output /exports/
```

### Q: What if I have multiple Splunk Cloud stacks?

**A: Run the script once per stack.** Each export will be labeled with the stack name.

---

## Support

If you encounter issues:

1. Check the troubleshooting section above
2. Review the export log file generated during the run
3. Contact the DynaBridge team with:
   - Error messages
   - Stack URL (without credentials)
   - Splunk Cloud type (Classic/Victoria)

---

## What to Expect: Step-by-Step Walkthrough

This section shows exactly what you'll see when running the script successfully.

### Step 1: Launch and Welcome Screen

When you run `./dynabridge-splunk-cloud-export.sh`, you'll see:

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                                                                                ║
║  ██████╗ ██╗   ██╗███╗   ██╗ █████╗ ██████╗ ██████╗ ██╗██████╗  ██████╗ ███████╗ ║
║  ██╔══██╗╚██╗ ██╔╝████╗  ██║██╔══██╗██╔══██╗██╔══██╗██║██╔══██╗██╔════╝ ██╔════╝ ║
║  ██║  ██║ ╚████╔╝ ██╔██╗ ██║███████║██████╔╝██████╔╝██║██║  ██║██║  ███╗█████╗   ║
║  ██║  ██║  ╚██╔╝  ██║╚██╗██║██╔══██║██╔══██╗██╔══██╗██║██║  ██║██║   ██║██╔══╝   ║
║  ██████╔╝   ██║   ██║ ╚████║██║  ██║██████╔╝██║  ██║██║██████╔╝╚██████╔╝███████╗ ║
║  ╚═════╝    ╚═╝   ╚═╝  ╚═══╝╚═╝  ╚═╝╚═════╝ ╚═╝  ╚═╝╚═╝╚═════╝  ╚═════╝ ╚══════╝ ║
║                                                                                ║
║                   ☁️  SPLUNK CLOUD EXPORT SCRIPT  ☁️                         ║
║                                                                                ║
║          Complete REST API-Based Data Collection for Migration              ║
║                        Version 4.1.0                                    ║
║                                                                                ║
╚══════════════════════════════════════════════════════════════════════════════╝

Do you want to continue? (Y/n):
```

**Action**: Press `Y` or Enter to continue.

### Step 2: Pre-Flight Checklist

After confirming, you'll see a checklist and system verification:

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                     PRE-FLIGHT CHECKLIST                                    ║
║         Please confirm you have the following before continuing            ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  SPLUNK CLOUD ACCESS:                                                      ║
║    □  Splunk Cloud stack URL (e.g., your-company.splunkcloud.com)          ║
║    □  Splunk username with admin privileges                                ║
║    □  Splunk password OR API token (sc_admin role recommended)             ║
║                                                                              ║
║  🔒 DATA PRIVACY & SECURITY:                                                ║
║                                                                              ║
║  We do NOT collect or export:                                              ║
║    ✗  User passwords or password hashes                                    ║
║    ✗  API tokens or session keys                                           ║
║    ✗  Private keys or certificates                                         ║
║    ✗  Your actual log data (only metadata/structure)                       ║
║                                                                              ║
║  We automatically REDACT:                                                  ║
║    ✓  password = [REDACTED] in all .conf files                             ║
║    ✓  secret = [REDACTED] in outputs.conf                                  ║
╚══════════════════════════════════════════════════════════════════════════════╝

  Quick System Check:
    ✓ bash: 5.2.15(1)-release
    ✓ curl: 8.1.2
    ✓ jq: jq-1.6
    ✓ tar: available

Ready to proceed? (Y/n):
```

**Action**: Press `Y` if all checks pass.

### Step 3: Enter Splunk Cloud Stack URL

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: SPLUNK CLOUD CONNECTION                                             │
└─────────────────────────────────────────────────────────────────────────────┘

  Your Splunk Cloud stack URL looks like:
    https://your-company.splunkcloud.com

  Enter your Splunk Cloud stack URL: acme-corp.splunkcloud.com

◐ Testing connection to https://acme-corp.splunkcloud.com:8089...
✓ Connection successful
```

**Action**: Enter your stack URL (without `https://` prefix).

### Step 4: Authentication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: AUTHENTICATION                                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  Required Permissions:
    • admin_all_objects - Access all knowledge objects
    • list_users, list_roles - Access RBAC data
    • search - Run analytics queries

  🔒 Security: Your credentials are used locally only and are NEVER stored,
     logged, or transmitted outside of this session. They are cleared on exit.

  Choose authentication method:

    1) API Token (recommended)
    2) Username/Password

  Select option [1]: 1

  Enter API token: ••••••••••••••••••••••••••••••••

◐ Testing authentication...
✓ Token authentication successful (user: admin)
```

**Action**: Choose auth method and enter credentials.

### Step 5: Select Data Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5: DATA CATEGORIES                                                     │
└─────────────────────────────────────────────────────────────────────────────┘

  Select data categories to collect:

    [✓] 1. Configurations (via REST - reconstructed from API)
    [✓] 2. Dashboards (Classic + Dashboard Studio)
    [✓] 3. Alerts & Saved Searches
    [✓] 4. Users & RBAC (usernames & roles only - NO passwords)
    [✓] 5. Usage Analytics (via search on _audit)
    [✓] 6. Index Statistics
    [✓] 7. Lookup Contents (may be large)
    [ ] 8. Anonymize Data (emails→fake, hosts→fake, IPs→redacted)

  🔒 Privacy: User data includes names/roles only. Passwords are NEVER collected.
  💡 Tip: Enable option 8 when sharing export with third parties.

  Accept defaults? (Y/n): Y
```

**Action**: Press `Y` to accept defaults or `n` to customize.

### Step 6: Data Collection Progress

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ COLLECTING DATA                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

  [1/7] Collecting system information...
✓ Server info collected
✓ Installed apps collected

  [2/7] Collecting configurations via REST API...
✓ Props configuration collected
✓ Transforms configuration collected
✓ Indexes configuration collected

  [3/7] Collecting dashboards...
[████████████████████████████████████████] 100% security_app/security_overview
✓ Collected 47 Classic dashboards
✓ Collected 12 Dashboard Studio dashboards

  [4/7] Collecting alerts and saved searches...
✓ Collected 89 saved searches (34 alerts)

  [5/7] Collecting users and roles...
✓ Collected 23 users
✓ Collected 8 roles

  [6/7] Collecting usage analytics...
◐ Running search: Dashboard views (last 30 days)...
✓ Dashboard usage collected
◐ Running search: User activity...
✓ User activity collected

  [7/7] Collecting index statistics...
✓ Index stats collected for 15 indexes
```

### Step 7: Export Complete

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                         EXPORT COMPLETE!                                     ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  Export Archive:                                                             ║
║    📦 dynabridge_cloud_export_acme-corp_20241203_143052.tar.gz               ║
║                                                                              ║
║  Summary:                                                                    ║
║    • Dashboards:        59 (47 Classic + 12 Studio)                          ║
║    • Alerts:            34                                                   ║
║    • Saved Searches:    89                                                   ║
║    • Users:             23                                                   ║
║    • Roles:             8                                                    ║
║    • Apps:              12                                                   ║
║    • Indexes:           15                                                   ║
║                                                                              ║
║  Duration: 4 minutes 23 seconds                                              ║
║  Archive Size: 2.3 MB                                                        ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  NEXT STEPS:                                                                 ║
║                                                                              ║
║  1. Upload to DynaBridge:                                                    ║
║     Open DynaBridge for Splunk app → Data Sources → Upload Export            ║
║                                                                              ║
║  2. Review the summary report:                                               ║
║     cat dynabridge_cloud_export_acme-corp_20241203_143052/                   ║
║         dynasplunk-env-summary.md                                            ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

### What Success Looks Like

After a successful export, you'll have a `.tar.gz` file. Extract it to see:

```bash
$ tar -tzf dynabridge_cloud_export_acme-corp_20241203_143052.tar.gz | head -20

dynabridge_cloud_export_acme-corp_20241203_143052/
dynabridge_cloud_export_acme-corp_20241203_143052/manifest.json
dynabridge_cloud_export_acme-corp_20241203_143052/dynasplunk-env-summary.md
dynabridge_cloud_export_acme-corp_20241203_143052/_export.log
dynabridge_cloud_export_acme-corp_20241203_143052/_systeminfo/
dynabridge_cloud_export_acme-corp_20241203_143052/_systeminfo/server_info.json
dynabridge_cloud_export_acme-corp_20241203_143052/_systeminfo/installed_apps.json
dynabridge_cloud_export_acme-corp_20241203_143052/_rbac/
dynabridge_cloud_export_acme-corp_20241203_143052/_rbac/users.json
dynabridge_cloud_export_acme-corp_20241203_143052/_rbac/roles.json
dynabridge_cloud_export_acme-corp_20241203_143052/_usage_analytics/
dynabridge_cloud_export_acme-corp_20241203_143052/_usage_analytics/dashboard_views.json
dynabridge_cloud_export_acme-corp_20241203_143052/_usage_analytics/users_most_active.json
dynabridge_cloud_export_acme-corp_20241203_143052/security_app/
dynabridge_cloud_export_acme-corp_20241203_143052/security_app/dashboards/
dynabridge_cloud_export_acme-corp_20241203_143052/security_app/savedsearches.json
```

### If Something Goes Wrong

If errors occur, you'll see a warning box:

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ⚠️  EXPORT COMPLETED WITH 3 ERRORS                                          ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  Some data could not be collected. See details below:                        ║
║                                                                              ║
║  Errors:                                                                     ║
║    • HTTP 403: Access denied to /services/data/lookup-table-files           ║
║    • Search timeout: Usage analytics query exceeded 5 minutes               ║
║    • HTTP 429: Rate limited - some data may be incomplete                   ║
║                                                                              ║
║  A troubleshooting report has been generated:                                ║
║    📄 TROUBLESHOOTING.md                                                      ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

Review `TROUBLESHOOTING.md` in the export directory for specific remediation steps.

---

## Sample Output Files

### Example: dynasplunk-env-summary.md

This human-readable summary report is generated in the export directory:

```markdown
# DynaBridge Splunk Cloud Environment Summary

**Export Date**: 2025-12-03 14:30:52 EST
**Export Script Version**: 4.1.0
**Export Type**: Splunk Cloud (REST API)

---

## Environment Overview

| Property | Value |
|----------|-------|
| **Stack URL** | acme-corp.splunkcloud.com |
| **Cloud Type** | Victoria Experience |
| **Splunk Version** | 9.1.3 |
| **Server GUID** | 8F4A2B1C-3D5E-6F7A-8B9C-0D1E2F3A4B5C |

---

## Collection Summary

| Category | Count | Status |
|----------|-------|--------|
| **Applications** | 12 | ✅ Collected |
| **Dashboards** | 59 | ✅ Collected |
| **Alerts** | 34 | ✅ Collected |
| **Users** | 23 | ✅ Collected |
| **Indexes** | 15 | ✅ Collected |

---

## Collection Statistics

| Metric | Value |
|--------|-------|
| **API Calls Made** | 347 |
| **Rate Limit Hits** | 2 |
| **Errors** | 0 |
| **Warnings** | 1 |

---

## Data Categories Collected

- ✅ Configurations (via REST API reconstruction)
- ✅ Dashboards (Classic and Dashboard Studio)
- ✅ Alerts and Saved Searches
- ✅ Users, Roles, and RBAC
- ✅ Usage Analytics (last 30d)
- ✅ Index Statistics
- ⏭️ Lookup Contents (skipped)
- ⏭️ Data Anonymization (available - enable with option 8)

---

## Applications Exported

- search
- security_app
- itsi
- splunk_app_for_aws
- enterprise_security
- phantom
- dashboard_studio
- user-prefs
- learned
- introspection_generator_addon
- alert_manager
- monitoring_console

---

## Cloud Export Notes

This export was collected via REST API from Splunk Cloud. Some differences from Enterprise exports:

1. **Configuration Files**: Reconstructed from REST API endpoints (not direct file access)
2. **Usage Analytics**: Collected via search queries on _audit and _internal indexes
3. **Index Statistics**: Limited to what's available via REST API
4. **No File System Access**: Cannot access raw bucket data, audit logs, etc.

---

## Errors and Warnings

### Errors (0)
No errors occurred.

### Warnings (1)
- Rate limit approached on dashboard collection; added 2s delay

---

## Next Steps

1. **Upload to DynaBridge**: Upload the `.tar.gz` file to DynaBridge in Dynatrace
2. **Review Dashboards**: Check the dashboard conversion preview
3. **Review Alerts**: Check alert conversion recommendations
4. **Plan Data Ingestion**: Use OpenPipeline templates for log ingestion

---

*Generated by DynaBridge Splunk Cloud Export Script v4.0.0*
```

### Example: manifest.json (Schema)

This machine-readable manifest is used by DynaBridge to process your export:

```json
{
  "schema_version": "3.3",
  "export_tool": "dynabridge-splunk-cloud-export",
  "export_tool_version": "4.0.0",
  "export_timestamp": "2025-12-03T19:30:52Z",
  "export_duration_seconds": 263,

  "source": {
    "hostname": "acme-corp.splunkcloud.com",
    "fqdn": "acme-corp.splunkcloud.com",
    "platform": "Splunk Cloud",
    "platform_version": "Victoria Experience"
  },

  "splunk": {
    "home": "cloud",
    "version": "9.1.3",
    "build": "cloud",
    "flavor": "cloud",
    "role": "search_head",
    "architecture": "cloud",
    "is_cloud": true,
    "cloud_type": "Victoria Experience",
    "server_guid": "8F4A2B1C-3D5E-6F7A-8B9C-0D1E2F3A4B5C"
  },

  "collection": {
    "configs": true,
    "dashboards": true,
    "alerts": true,
    "rbac": true,
    "usage_analytics": true,
    "usage_period": "30d",
    "indexes": true,
    "lookups": false
  },

  "statistics": {
    "apps_exported": 12,
    "dashboards_classic": 47,
    "dashboards_studio": 12,
    "dashboards_total": 59,
    "alerts": 34,
    "saved_searches": 89,
    "users": 23,
    "roles": 8,
    "indexes": 15,
    "api_calls_made": 347,
    "rate_limit_hits": 2,
    "errors": 0,
    "warnings": 1,
    "total_files": 234,
    "total_size_bytes": 2411724
  },

  "apps": [
    {
      "name": "security_app",
      "dashboards": 15,
      "alerts": 12,
      "saved_searches": 28
    },
    {
      "name": "itsi",
      "dashboards": 8,
      "alerts": 6,
      "saved_searches": 14
    }
  ],

  "usage_intelligence": {
    "summary": {
      "dashboards_never_viewed": 12,
      "alerts_never_fired": 8,
      "users_inactive_30d": 5,
      "alerts_with_failures": 2
    },
    "volume": {
      "avg_daily_gb": 45.7,
      "peak_daily_gb": 78.3,
      "total_30d_gb": 1371.2,
      "top_indexes_by_volume": [
        {"index": "main", "total_gb": 456.2},
        {"index": "security", "total_gb": 312.8},
        {"index": "web_logs", "total_gb": 198.4}
      ],
      "top_sourcetypes_by_volume": [
        {"sourcetype": "access_combined", "total_gb": 234.5},
        {"sourcetype": "syslog", "total_gb": 187.3}
      ]
    },
    "prioritization": {
      "top_dashboards": [
        {"dashboard": "security_overview", "views": 1523},
        {"dashboard": "executive_summary", "views": 892}
      ],
      "top_users": [
        {"user": "admin", "searches": 4521},
        {"user": "analyst1", "searches": 2134}
      ],
      "top_alerts": [
        {"alert": "High CPU Alert", "fires": 234},
        {"alert": "Failed Login", "fires": 156}
      ]
    },
    "elimination_candidates": {
      "dashboards_never_viewed_count": 12,
      "alerts_never_fired_count": 8,
      "note": "See _usage_analytics/ for full lists of candidates"
    }
  }
}
```

This manifest enables DynaBridge to:
- **Prioritize migration** based on actual usage data
- **Identify elimination candidates** (unused dashboards/alerts)
- **Estimate data volume** for Dynatrace ingestion planning
- **Map applications** to their respective assets

---

*For Splunk Enterprise (on-premises), use `dynabridge-splunk-export.sh` instead.*
