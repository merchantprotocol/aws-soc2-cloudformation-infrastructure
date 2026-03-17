## What

Brief description of the change.

## Why

Motivation — what problem does this solve or what requirement does it meet?

## Changes

-
-
-

## SOC 2 Impact

- [ ] No compliance impact
- [ ] Adds new security controls
- [ ] Modifies existing security controls
- [ ] Updates compliance documentation

If modifying controls, list which TSC criteria are affected:

## Checklist

- [ ] Template validates: `aws cloudformation validate-template --template-body file://template.yaml`
- [ ] Tested deployment in a non-production stack
- [ ] No security groups opened to `0.0.0.0/0` on non-web ports
- [ ] Encryption at rest and in transit preserved
- [ ] Updated relevant `docs/` if procedures or controls changed
- [ ] Updated `SOC2-CONTROL-MATRIX.md` if controls were added or modified
- [ ] README updated if template capabilities changed

## Testing

Describe how this was tested:

## Rollback Plan

How to revert if something goes wrong:
