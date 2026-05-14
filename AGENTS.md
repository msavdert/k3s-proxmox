# AI Agent Guidelines (AGENTS.md)

Welcome, AI Agent. When interacting with this repository, please adhere to the following guidelines to maintain a high-quality, consistent "Hard Way" documentation standard.

## 1. Documentation Standards
- Write all documentation in clear, professional English.
- Use explicit, step-by-step instructions. Avoid skipping "trivial" steps. This is a "Hard Way" guide.
- Provide the exact terminal commands required. Always use fenced code blocks with the appropriate language tag (e.g., `bash`, `yaml`).
- Highlight important information, prerequisites, or destructive actions using GitHub Alerts (`> [!NOTE]`, `> [!WARNING]`, `> [!CAUTION]`).
- Do not use placeholder values in documentation without clearly defining them using angle brackets (e.g., `<PUBLIC_IP>`, `<YOUR_PASSWORD>`).

## 2. Directory Structure
- `docs/`: Contains all the ordered markdown chapters (`00-`, `01-`, etc.). Keep the numbering logical and sequential.
- `scripts/`: If any helper scripts are added, they must be placed here and clearly referenced in the docs.
- `manifests/`: Kubernetes YAML files should be placed here, structured logically by component.

## 3. Code & Secret Management
- **Never commit secrets.** Always use placeholder values (`<SECRET_NAME>`) in documentation and manifests.
- If an agent generates passwords, API tokens, or IP addresses during a local session, it must sanitize them before writing them to any file in the repository.

## 4. Markdown Formatting
- Use `#` for document titles (only one per file).
- Use `##` and `###` for sections.
- Keep lines relatively short where possible, but do not artificially break long URLs or command strings.

## 5. Interactions
- When making large structural changes, create an `implementation_plan.md` first and ask for the USER's approval.
