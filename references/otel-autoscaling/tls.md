# TLS and source credentials

Treat each connection separately: app/source → collector, collector → add-on
OTLP receiver, and KEDA → add-on external scaler. TLS on one does not enable it
on another. Reuse the user's trust and secret-management conventions. Keep
private keys/tokens out of chart values, manifests committed to Git, logs, and
PR descriptions; use existing Secret references or the repository's approved
encrypted/external-secret source.

## Collector → add-on

In the inspected chart, `settings.tls.certFile` and `keyFile` configure the
add-on's OTLP server certificate. `settings.tls.caFile` supplies the client CA
for verifying client certificates when mutual TLS is required.
`settings.tls.secrets` mounts existing Secrets into the scaler Pod. Verify the
selected version's field behavior and that the cert/key pair is present.

Configure the collector's `otlp/keda` exporter with matching `tls.ca_file` and,
for mTLS, `tls.cert_file` and `tls.key_file`, using collector-mounted Secrets.
Mount fields depend on the owner:

- Bundled collector chart: `otelCollector.extraVolumes`/`extraVolumeMounts`.
- Chart-generated operator CR: the selected `otelOperatorCrs` item's
  `tls.secrets`, with its TLS paths or explicit exporter configuration.
- Existing collector/operator CR: its actual chart or CR volume/mount interface.

Secrets must exist in the namespace of the consuming Pod. A matching Secret name
in the scaler namespace does not mount it into a collector in another namespace.
Use a certificate whose SAN matches the actual Service DNS name and a trusted
CA. Preserve source-endpoint authentication when adding a scaling pipeline.

## KEDA → add-on

The chart's `settings.tls.keda.enabled` enables TLS on the separate KEDA gRPC
server, with its own `certFile`/`keyFile` or the documented fallback to the
shared pair. The inspected server presents a certificate but does not configure
client-certificate verification for this connection. Do not claim end-to-end
mTLS solely because the collector connection uses mTLS.

Use the installed KEDA version's external-scaler TLS interface. Set
`enableTLS: "true"` in trigger metadata and use a `TriggerAuthentication`
`secretTargetRef` for `caCert` and client credentials if needed/supported.
Check version-specific parameter names against
[KEDA's external scaler documentation](https://keda.sh/docs/2.18/scalers/external/)
and the installed version. Associate the authentication resource through the
trigger's `authenticationRef`, using the workload namespace for a namespaced
TriggerAuthentication and its Secrets.

The diagnostic REST endpoint is separate from these gRPC TLS settings. Prefer
local port-forwarding rather than exposing it publicly.

## Verification and rotation

Check certificate SANs, CA chain, expiry, mounted paths, and protocol on both
sides. Diagnose `unknown authority`, hostname mismatch, expired certs, missing
client certs, and plaintext/TLS mismatch at the failing connection. Do not copy
demo `unsafeSsl` or `insecure_skip_verify` settings as fixes. Plaintext
`tls.insecure: true` examples describe a deliberate deployment choice, not a
fallback for failed TLS verification.

The inspected OTLP server supports certificate reload configuration through
`settings.tls.reloadInterval`; the separate KEDA TLS server loads its certificate
at startup. Verify rotation behavior for the deployed version and plan a scoped
rollout where required; do not promise that one reload interval covers all links.
