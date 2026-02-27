# Security Policy

## Reporting Security Vulnerabilities

If you discover a security vulnerability in `claude-spend`, please email security concerns to the repository maintainers rather than opening a public GitHub issue.

Please include:
- Description of the vulnerability
- Steps to reproduce (if applicable)
- Potential impact
- Suggested fix (if you have one)

We will acknowledge receipt within 48 hours and work on a fix promptly.

## Data Privacy

**`claude-spend` processes data locally only.** No data is sent to external servers:

- ✅ Reads from `~/.claude/` on your machine
- ✅ Runs an Express server on `localhost` only
- ✅ No network requests to analytics, telemetry, or tracking services
- ✅ All processing happens in-memory on your computer

Your Claude Code session data and token metrics remain private.

## Security Best Practices

### Using `claude-spend`

1. **Trust the source** — Install only from the official NPM registry:
   ```bash
   npx claude-spend
   ```

2. **Keep it updated** — Periodically update to get security patches:
   ```bash
   npx claude-spend@latest
   ```

3. **Run locally** — This tool is designed for local machines only. Do not expose the dashboard to the internet.

### Installation

This tool requires Node.js 14+. No additional configuration or environment variables are needed.

### Dependencies

`claude-spend` has minimal dependencies:
- `express` — Lightweight web server
- `open` — Auto-open browser (platform-native, no external calls)

All dependencies are regularly scanned for vulnerabilities.

## Scope

This security policy applies to the `claude-spend` npm package and this GitHub repository only.

## Questions?

If you have questions about the security of this tool, please reach out to the maintainers.
