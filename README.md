# oixio-reportapp-public

OIXIO ReportApp for users: work with dashboards and Power BI reports in ReportApp, without changing settings.

Published by OIXIO. **Don't edit this repository directly:**
the next publish overwrites any changes made here.

| Plugin | Version | What it does |
| --- | --- | --- |
| OIXIO ReportApp User (`reportapp-user`) | 0.1.2 | For ReportApp users: understand the numbers on a dashboard or Power BI report, fill in values and forms for you to save, and filter the view, without changing any settings. Configuration and setup are administrator work and not included. Don't install it next to OIXIO ReportApp Admin. |

## Install

**Claude Code** (terminal, VS Code, desktop):

```
/plugin marketplace add <this repository's URL, or owner/name on GitHub>
/plugin install reportapp-user@oixio-reportapp-public
```

Then turn on automatic updates: `/plugin` → **Marketplaces** → `oixio-reportapp-public` → **Enable auto-update**.
Without it, run `/plugin marketplace update oixio-reportapp-public` to get new versions.

**claude.ai, Claude Desktop, Cowork:** go to Customize → Plugins, add a marketplace from a Git repository
using this repository's URL, then install `reportapp-user`. Organization owners can instead sync this
repository for everyone under Organization settings → Plugins & skills.
