# Repository Guidelines

## Project Structure & Module Organization
- Root contains operational docs in Markdown (e.g., `AzureCommand.md`, `JP1-SEP-and-AMA-operation.md`).
- When adding assets or code, use:
  - `assets/` for images/diagrams referenced by docs.
  - `scripts/` for helper scripts (Bash/PowerShell) used in procedures.
  - `infra/` for IaC templates (Bicep/ARM/Terraform) related to Azure VM provisioning.

## Build, Test, and Development Commands
- No build step is required; Markdown renders in any viewer.
- Optional quality checks (run if you have the tools installed):
  - `markdownlint "**/*.md"` — lint Markdown style.
  - `prettier -w "**/*.md"` — format Markdown consistently.
  - `shellcheck scripts/*.sh` — static analysis for Bash scripts (if present).

## Coding Style & Naming Conventions
- Markdown: use ATX headings (`#`), sentence‑case titles, and numbered lists for ordered steps.
- Code/CLI: use fenced blocks with language hints, for example:
  
  ```bash
  az vm create --resource-group <rg> --name <vm> --image UbuntuLTS --only-show-errors
  ```
- Filenames: kebab‑case and descriptive (e.g., `azure-vm-create.md`, `jp1-ama-operations.md`).
- Keep lines readable (~100 chars); prefer concise, task‑oriented sections.

## Testing Guidelines
- Docs: verify each command with a dry run when possible (`--what-if`, `-WhatIf`, or equivalent). Include prerequisites and rollback notes.
- Scripts/IaC (if added): provide a quick validation path (e.g., `terraform validate`, `bicep build`, or `az deployment group what-if`).
- Place script samples under `scripts/` and show usage in the related doc.

## Commit & Pull Request Guidelines
- Commit style: bracketed prefixes seen in history (e.g., `[Add]`, `[Fix]`, `[Docs]`, `[Chore]`). Example: `[Docs] Clarify Azure VM create flags`.
- Subject: imperative, ≤72 chars; Body: what/why and notable context.
- PRs should include: clear summary, affected files, test evidence (logs/output snippets), and linked issues.

## Security & Configuration Tips
- Do not commit secrets, keys, or tenant/subscription IDs; use placeholders like `<SUBSCRIPTION_ID>` and redact outputs.
- Prefer `az login` with local credentials; store sensitive values in Key Vault or environment variables, not in this repo.
- Mask or remove command outputs that expose resources before committing.

