# Paperclip Skills

A collection of skills for AI coding agents to manage [Paperclip](https://github.com/paperclipai/paperclip) instances. Skills are packaged instructions that extend agent capabilities for setup, administration, and day-to-day operations.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### paperclip-setup

Set up, configure, and maintain a local Paperclip instance — onboarding, diagnostics, worktrees, authentication, and context profiles.

**Use when:**
- Setting up a new Paperclip instance
- Running diagnostics or repairs
- Managing worktrees for isolated environments
- Configuring authentication and context profiles

### paperclip-status

Check Paperclip instance health, dashboard summary, live agent runs, activity feed, and sidebar alert badges.

**Use when:**
- Checking instance health
- Viewing dashboard summary
- Monitoring live agent runs
- Reviewing activity feed

### paperclip-manage-access

Manage Paperclip access control — create invites, handle join requests, manage company members and permissions, configure CLI authentication, and administer users.

**Use when:**
- Creating invites for agents or users
- Approving or rejecting join requests
- Managing member permissions
- Bootstrapping the first admin account

### paperclip-manage-adapters

Manage Paperclip adapters — list built-in and external adapters, install new ones from npm, hot-reload during development, and inspect config schemas.

**Use when:**
- Listing available adapters
- Installing new adapters
- Hot-reloading adapters during development

### paperclip-manage-agents

Manage Paperclip agents — hire new agents, configure adapters, pause/resume/terminate, trigger heartbeats, manage API keys, and set instructions.

**Use when:**
- Hiring or configuring agents
- Managing agent lifecycle (pause, resume, terminate)
- Setting agent instructions or API keys

### paperclip-manage-approvals

Manage Paperclip approvals — review pending requests, approve/reject agent hires and actions, request revisions, and add comments.

**Use when:**
- Reviewing pending approval requests
- Approving or rejecting agent actions
- Adding comments to approval workflows

### paperclip-manage-companies

Manage Paperclip companies — create, list, update settings, archive, import/export portable bundles, and configure branding.

**Use when:**
- Creating or archiving companies
- Updating company settings or branding
- Importing/exporting company bundles

### paperclip-manage-costs

Monitor Paperclip costs and budgets — view spending summaries and breakdowns, set budget policies with hard stops, and resolve budget incidents.

**Use when:**
- Viewing spending summaries
- Setting budget policies
- Resolving budget incidents

### paperclip-manage-goals

Manage Paperclip goals — create strategic objectives, organize them hierarchically, and track their status.

**Use when:**
- Creating or updating strategic goals
- Organizing goal hierarchies
- Tracking goal progress

### paperclip-manage-issues

Manage Paperclip issues (tasks) — create, assign, checkout/release, update status, add comments, manage labels, documents, work products, and attachments.

**Use when:**
- Creating or assigning issues
- Updating issue status or labels
- Managing work products and attachments

### paperclip-manage-plugins

Manage Paperclip plugins — install from npm or local path, configure, enable/disable, monitor health, manage scheduled jobs and webhooks.

**Use when:**
- Installing or configuring plugins
- Enabling/disabling plugins
- Monitoring plugin health

### paperclip-manage-projects

Manage Paperclip projects — create projects with workspaces, configure environments, manage execution workspaces, and control runtime services.

**Use when:**
- Creating projects and workspaces
- Configuring project environments
- Managing runtime services

### paperclip-manage-routines

Manage Paperclip routines — create scheduled automations with cron/webhook/API triggers, configure concurrency policies, and manually trigger runs.

**Use when:**
- Creating scheduled automations
- Configuring triggers and concurrency
- Manually triggering routine runs

### paperclip-manage-secrets

Manage Paperclip secrets — store API keys and credentials securely, rotate values, and configure secret providers for agent environments.

**Use when:**
- Storing or rotating secrets
- Configuring secret providers
- Managing credentials for agents

## Installation

```bash
npx skills add albertobaselga/paperclip-skills
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant Paperclip tasks are detected.

**Examples:**
```
Set up a new Paperclip instance
```
```
Create an invite for a new agent
```
```
Check the instance health
```
```
Set a budget policy for the engineering company
```

## Skill Structure

Each skill contains:
- `SKILL.md` - Instructions for the agent
- `references/` - Supporting documentation (optional)

## License

MIT
