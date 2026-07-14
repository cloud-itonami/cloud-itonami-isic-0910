# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in cloud-itonami-isic-0910, please
**do not open a public issue**. Instead, email the cloud-itonami project team
with a detailed description of the vulnerability, affected versions, and
recommended remediation.

Security disclosures will be reviewed promptly and a patch released if warranted.

## Security Considerations

### Scope Boundary Enforcement

This project is designed to enforce a hard boundary between contractor and
operator authority:

- **Contractor-side operations** (service order intake, crew dispatch, logistics)
  are within scope.
- **Operator-side operations** (drilling, well-control, hazmat authorization)
  are hard-blocked and cannot be overridden.

If a vulnerability is discovered that allows an operator-class operation to
bypass the governor's enforcement, it is treated as a critical security issue.

### Audit Trail

All decisions are logged to an append-only audit ledger. Any attempt to tamper
with ledger entries is a critical issue.

### LLM Safety

The built-in `llm-advisor` never fabricates confidence scores; LLM parse
failures default to confidence 0.0, triggering escalation to human review.
If you discover a code path that fabricates confidence, report it as a security issue.

## License

This project is released under AGPL-3.0-or-later. See LICENSE for details.
