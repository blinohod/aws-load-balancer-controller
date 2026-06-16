# Proposal: External DNS Validation for ACM Certificate Management

## Summary

Add an opt-out mechanism that allows the `EnableCertificateManagement` feature to request
ACM certificates without managing Route53 DNS validation records, delegating that
responsibility to external tooling.

## Motivation

The current `EnableCertificateManagement` implementation always creates and deletes Route53
CNAME records as part of the certificate lifecycle. This design couples ACM certificate
issuance to Route53 and prevents adoption in several common environments:

- **Non-Route53 DNS providers**: Organizations using Cloudflare, Akamai, Infoblox, or other
  DNS providers cannot use this feature at all — the controller only supports Route53.
- **Cross-account Route53**: When hosted zones live in a separate AWS account from the EKS
  cluster, the controller has no mechanism to assume a role in the zone-owner account.
  Granting the controller cross-account Route53 write access is operationally complex and
  expands the blast radius of the controller's IAM role.
- **Centralized DNS automation**: Organizations with existing DNS automation platforms
  (internal tooling, external-dns, cert-manager ACME solvers, etc.) that already handle
  ACM DNS validation have no way to reuse that infrastructure — they must either duplicate
  it or forego the `EnableCertificateManagement` feature entirely.

In all of these cases, the desired behavior is identical: the controller requests the ACM
certificate and wires the ARN into the ALB listener, while DNS validation is handled
independently by whatever tooling the organization already operates.

## Proposed Interface

### Controller-level flag

A new flag that disables Route53 management for all certificates created by this controller
instance:

```
--acm-cert-skip-dns-validation=true  (default: false)
```

When this flag is set to `true`, it is a hard policy: per-Ingress annotations are ignored
and Route53 is never called. This is the correct mode for environments where Route53
permissions are not available on the controller's IAM role at all.

### Per-Ingress annotation

A new annotation that opts a specific Ingress out of Route53 management when the
controller-level flag is `false`:

```yaml
alb.ingress.kubernetes.io/acm-cert-manage-dns-validation: "false"
```

The annotation is only evaluated when the controller flag is `false`. When the controller
flag is `true`, this annotation has no effect — the controller never calls Route53
regardless. The annotation can only add restriction, never relax the controller-level
policy.

This allows mixed deployments where most Ingresses use external DNS validation but some
use the built-in Route53 integration, as long as the controller has Route53 permissions.

## Behavior Changes

### Certificate creation

| Mode | Behavior |
|---|---|
| Standard (flag=false, no annotation) | Pre-check hosted zones → `RequestCertificate` → create Route53 CNAMEs → wait for `ISSUED` |
| Skip DNS (flag=true, OR annotation=true) | `RequestCertificate` → wait for `ISSUED` (no Route53 calls) |

In skip-DNS mode, the pre-flight hosted zone check (`GetHostedZoneID` per domain) is also
skipped. External tooling is responsible for placing the validation CNAME within the
controller's wait window (see [Validation Timeout](#validation-timeout) below).

### Certificate re-issue (stuck in PENDING_VALIDATION)

When a cert has been in `PENDING_VALIDATION` for longer than `reissueWaitTime` (5 minutes),
the synthesizer deletes and recreates it. In skip-DNS mode: `Delete` (not
`DeleteWithValidationRecords`) is used for deletion, and `Create` (not
`CreateWithValidationRecords`) for recreation. No Route53 calls are made at any point.

Note: the re-issue on a 5-minute timeout can interfere with external DNS validation
pipelines that have longer propagation times. See [Validation Timeout](#validation-timeout).

### Certificate deletion

In skip-DNS mode: the ACM certificate is deleted directly. No Route53 record cleanup is
attempted.

Since orphaned certs in `PostSynthesize` have no associated Ingress resource (and therefore
no model-side `SkipDNSValidation` field available), the deletion path is determined by the
controller-level flag at the time of deletion. This is correct for the hard-policy case
(flag=true). For the per-annotation case (flag=false, annotation=true), the controller
will attempt `DeleteWithValidationRecords` for orphaned certs, which will result in
harmless no-ops if no Route53 records exist for those domains.

### Synthesizer branch points

The following 4 branch points in `certificate_synthesizer.go` must all be updated:

1. `Synthesize` → unmatchedResCerts loop: `CreateWithValidationRecords` → `Create`
2. `Synthesize` → matchedCerts re-issue loop: `DeleteWithValidationRecords` → `Delete`
3. `Synthesize` → matchedCerts re-issue loop: `CreateWithValidationRecords` → `Create`
4. `PostSynthesize` → orphaned cert delete loop: `DeleteWithValidationRecords` → `Delete`
   (conditioned on controller-level flag only, not per-cert state)

### Validation timeout

The controller waits up to 30 minutes (`validateWaitTime`) for a cert to reach `ISSUED`.
In standard mode this is generous since Route53 records are placed immediately. In
skip-DNS mode, the window includes the external tooling's discovery and propagation time.

If the timeout is exceeded, the reconcile fails and is re-queued. On the next reconcile,
if the cert has been in `PENDING_VALIDATION` for more than 5 minutes (`reissueWaitTime`),
the controller will delete and recreate it — resetting any progress the external tooling
has made. This is a known sharp edge: external tooling must complete validation within
5 minutes of the cert being requested to avoid this re-issue cycle.

Making `reissueWaitTime` configurable is out of scope for this proposal but is a natural
follow-on, particularly for environments with slower DNS propagation pipelines.

### IAM permissions

When the controller-level flag is `true`, Route53 permissions are not required and can be
omitted from the controller's IAM policy:

- `route53:ListHostedZones`
- `route53:ListHostedZonesByName`
- `route53:ChangeResourceRecordSets`

When the controller-level flag is `false` and only per-Ingress annotations are used,
Route53 permissions are still required on the controller's IAM role (for the Ingresses
that do not set the annotation).

## Implementation

Changes are required in 7 files, totaling approximately 50 lines of production code:

| File | Change |
|---|---|
| `pkg/model/acm/certificate.go` | Add `SkipDNSValidation bool` to `CertificateSpec` |
| `pkg/annotations/constants.go` | Add `IngressSuffixManageACMDNSValidation` constant |
| `pkg/config/ingress_config.go` | Add `--acm-cert-skip-dns-validation` controller flag |
| `pkg/ingress/model_builder.go` | Thread new flag through builder and task structs |
| `pkg/ingress/model_build_certificates.go` | Read annotation (when flag=false), set `SkipDNSValidation` |
| `pkg/deploy/acm/certificate_synthesizer.go` | Update all 4 branch sites; conditioned on `SkipDNSValidation` (sites 1-3) and controller flag (site 4) |
| `controllers/ingress/group_controller.go` | Pass new flag into `NewDefaultModelBuilder` |

## Limitations and Known Edges

- **Private (PCA) certificates are unaffected**: they already bypass Route53 validation.
- **No Gateway API support**: out of scope; Gateway API cert management tracked in #4774.
- **Mode change on existing cert**: changing the annotation on an existing Ingress from
  standard to skip-DNS mode does not take effect until the cert is recreated (either
  manually deleted or replaced by the re-issue timer). Route53 validation records created
  in standard mode are not cleaned up when switching to skip-DNS mode — they become
  orphaned in Route53 and must be removed manually.
- **5-minute re-issue timer**: external DNS pipelines must complete validation within
  5 minutes to avoid cert deletion and recreation. Document this prominently for operators.
- **External tooling discovery**: the controller emits no signal when a cert enters
  `PENDING_VALIDATION`. External tooling must discover pending certs independently
  (e.g., by polling `acm:ListCertificates`, reacting to EventBridge ACM state change
  events, or watching for a specific ACM tag). The required CNAME name and value are
  available via `acm:DescribeCertificate` → `DomainValidationOptions[].ResourceRecord`.

## Open Questions

1. Should `reissueWaitTime` (currently hardcoded at 5 minutes) be made configurable,
   specifically to support external DNS pipelines with longer propagation times?
2. Should this be implemented as a feature gate sub-option (consistent with how
   `EnableCertificateManagement` was introduced) rather than a plain `IngressConfig` flag?
3. Should `SkipDNSValidation` be factored into `isSDKCertificateRequiresReplacement` so
   that a mode change on an existing Ingress triggers immediate cert replacement rather
   than waiting for the re-issue timer?
