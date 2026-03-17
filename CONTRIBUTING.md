# Contributing

Thank you for your interest in contributing to this project. This document provides guidelines for submitting changes.

## Getting Started

1. Fork the repository
2. Create a feature branch from `main`: `git checkout -b feature/your-change`
3. Make your changes
4. Validate the template: `aws cloudformation validate-template --template-body file://your-template.yaml`
5. Submit a pull request

## What We Accept

- New CloudFormation templates for common AWS architectures
- Improvements to existing templates (security hardening, cost optimization, new features)
- Documentation improvements
- Bug fixes

## Guidelines

### Templates

- All templates must be valid CloudFormation YAML
- Use `AWS::CloudFormation::Interface` metadata to organize parameters with clear labels
- Every parameter should have a `Description` and a `Default` value where possible
- Required parameters (no default) must have clear descriptions explaining how to obtain the value
- Group parameters by importance — required fields first, advanced/rarely-changed fields last
- Use `AllowedValues` or `AllowedPattern` constraints to prevent invalid input
- Include `Tags` on all resources that support them
- Use `!Sub` and `!Ref` for dynamic values — never hardcode account IDs, regions, or ARNs

### Security

- Never open security groups to `0.0.0.0/0` except for HTTP/HTTPS on load balancers
- Enable encryption at rest on all data stores (RDS, EFS, S3, etc.)
- Enable encryption in transit where available
- Use IAM instance roles, not access keys
- Enable KMS key rotation
- Set `DeletionPolicy: Snapshot` on databases and `DeletionPolicy: Retain` on audit log buckets
- Never suppress cfn-lint or cfn_nag rules for encryption or deletion checks

### SOC 2 Compliance

Changes to the SOC 2 template (`cloudformation-launchtemplates-soc2.yaml`) have additional requirements:

- Do not remove or weaken any existing security control
- Update the [SOC 2 Control Matrix](docs/SOC2-CONTROL-MATRIX.md) if controls are added or modified
- Update the relevant procedure documents in `docs/` if operational procedures change
- Any change to security groups, IAM policies, encryption settings, or logging must include a justification in the PR description

### Documentation

- Update `docs/` if your template change affects any compliance procedure
- Use relative links to template files with line numbers (e.g., `[ResourceName](../vpc-standard-2privatesubnets/template.yaml#L100)`)
- Keep the README in sync with template capabilities

## Pull Request Process

1. Fill out the pull request template completely
2. Ensure the template validates without errors
3. Describe what changed, why, and what the impact is
4. For security-impacting changes, note which SOC 2 controls are affected
5. Wait for at least one review before merging

## Reporting Issues

Use the GitHub issue templates:

- **Bug Report** — Something in a template doesn't work as expected
- **Feature Request** — Suggest a new template or improvement
- **Security Vulnerability** — Report a security issue (use private reporting if sensitive)

## Code of Conduct

Be respectful, constructive, and professional. We're all here to build better infrastructure.
