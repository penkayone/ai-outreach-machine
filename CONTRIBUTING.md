# Contributing

Thanks for your interest in improving this project. This is a real production system, so contributions are welcome — especially:

- **New niches**: Scout source configurations for industries beyond real estate (legal, dental, consulting, e-commerce, etc.)
- **New languages**: prompt refinements for Copywriter and Tracker in languages not yet supported
- **Bug fixes**: edge cases in IMAP handling, Claude response parsing, rate limiting
- **Documentation**: clearer setup steps, troubleshooting, niche-specific guides

## How to contribute

1. Fork the repo
2. Create a branch: `git checkout -b feature/your-feature`
3. Test your changes on a sanitized n8n instance
4. Submit a pull request with description of what changed and why

## Code style for Code nodes

n8n Code nodes have specific quirks. Follow these conventions:

- Always access cross-node data via `$('Node Name').item.json.field`, not via `$json`
- Always handle missing fields with `||` defaults
- Always return `[{ json: {...} }]` array shape, even for single items
- For loops over candidates, use SplitInBatches v3
- For Claude API calls, build the full payload in a Code node and pass via `{{ JSON.stringify($json._claude_payload) }}` to keep HTTP node simple

## Sensitive data

**Never commit credentials, API keys, real lead emails, real client data, or company-specific configurations.** The repo's sanitization process is documented in the README.

## License

By contributing you agree your contributions are MIT licensed.
