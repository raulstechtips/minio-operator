# MinIO Operator wrapper chart

## Purpose

This chart installs the upstream `operator` chart 7.0.1 without template changes.
It adds the STS Certificate and the operator CA secret sync. The tenant chart is
in `../minio` and is not part of this chart.

The dependency comes from `https://operator.min.io`. On October 30, 2026, the
public archive and the Harbor archive both had SHA-256
`41f4db5213edbd64c7a0d6faee502a01200cbbebb02296f858f96ab89b43231a`.
The wrapper is published to GHCR, so chart retrieval does not need Harbor.

## Before first install

1. Make sure cert-manager, the Infisical operator with the v1beta1 CRDs, and
   `raulstechtips-internal-ca-issuer` are ready.
2. Make sure `InfisicalAuth/default/infisical-prod-universal-auth` is ready.
   It supplies the Infisical connection and credentials. The values file contains
   no secret values.
3. Keep the release name and namespace `minio-operator`. Both CRDs are already
   tracked by this Application. Keep them rendered.
4. Complete the secret handover in Adoption notes before the first manual sync.
5. After the chart repository is merged and CI publishes the chart to GHCR, set
   `cluster-config/argo/dev/minio-operator.yaml` `targetRevision` to the first
   CI version, for example `1.0.1-main-<sha>-<run>`. Do not sync
   `PENDING_CI_VERSION`.
6. Review the rendered resources and then sync the Application manually.

## Values reference

| Value | Default | Function |
| --- | --- | --- |
| `infisical.authRef.name`, `.namespace` | empty | Existing InfisicalAuth reference |
| `infisical.projectSlug`, `.environmentSlug` | empty | Infisical source |
| `stsCertificate.enabled` | `false` | Create `sts-tls-certificate` |
| `stsCertificate.secretName` | `sts-tls` | Certificate output Secret expected by the operator |
| `stsCertificate.duration`, `.renewBefore` | `8760h`, `720h` | Certificate lifetime and renewal interval |
| `stsCertificate.organization` | `raulstechtips` | Certificate subject organization |
| `stsCertificate.issuerRef.name`, `.kind` | `raulstechtips-internal-ca-issuer`, `ClusterIssuer` | Certificate issuer |
| `operatorCaSecret.enabled` | `false` | Create the CA InfisicalStaticSecret |
| `operatorCaSecret.secretName` | empty | CR and target Secret name |
| `operatorCaSecret.secretPath` | `/certificates/internal` | Infisical source path |
| `operatorCaSecret.recursive` | `true` | Include child folders |
| `operatorCaSecret.refreshInterval` | `60s` | Secret sync interval |
| `operator.operator` | live overrides | Values passed to the upstream chart |

The repeated `operator.operator` is required: the first key names the dependency;
the second key is the upstream values root. Upstream defaults provide the live
security contexts, affinity, resource requests, and empty sidecar image override.
Only the changed image, replica count, environment list, and limits are set here.
Environment-specific names and the Infisical connection are in
`../cluster-config/apps/dev/minio-operator.yaml`.

The STS DNS names are `sts`, `sts.<namespace>.svc`, and
`sts.<namespace>.svc.cluster.local`. They use the release namespace.

## Upgrades

1. Change the dependency version in `charts/Chart.yaml` only after review of
   upstream changes, including both CRDs.
2. Use `/opt/homebrew/bin/helm` to update the dependency lock. Keep `Chart.lock`;
   do not check in `charts/charts/`.
3. Run the same checks as CI:

   ```sh
   export TMPDIR=/tmp/ulw-wrappers/minio-operator
   mkdir -p "$TMPDIR"
   ../.omo/wrapper-charts/tools/checks.sh "$PWD" "$TMPDIR"
   ```

4. Render the environment values and compare against live before adoption:

   ```sh
   /opt/homebrew/bin/helm template minio-operator charts -n minio-operator \
     -f ../cluster-config/apps/dev/minio-operator.yaml
   ```

5. After CI publishes a new version, update the Application revision and use a
   manual sync. The live Application has no sync options or ignore rules to carry
   over. This definition also has no automatic sync.

## Adoption notes

- The October 30, 2026 parity result is
  `SAME=8 CHANGED=1 NEW=1 PRUNE=1 HOOK=0 UNKNOWN-KIND=0`.
  All eight upstream objects are SAME. The only differences are the Certificate
  labels and the secret CR migration described below.
- The live Application was `Synced` and `Healthy` with 10 resources, including
  `tenants.minio.min.io` and `policybindings.sts.min.io`. The upstream chart places
  these CRDs in `templates/`, not `crds/`. No `skipCrds` is set; both remain managed.
- The Certificate keeps its live name and full spec. It gains wrapper labels
  (`app.kubernetes.io/name`, `app.kubernetes.io/instance`,
  `app.kubernetes.io/managed-by`, and `helm.sh/chart`). This is a metadata-only
  change, not a certificate reissue request.
- The new `InfisicalStaticSecret/operator-ca-tls-tenant-s3` replaces the old
  `InfisicalSecret/infisical-secret-operator-ca-tls-tenant-s3`. The new object and
  target use the live Secret name and namespace. It keeps `kubernetes.io/tls`,
  `/certificates/internal`, recursive fetch, all keys, and the 60-second interval.
  It uses the existing v1beta1 auth reference and `creationPolicy: Orphan`.
- In v1beta1, an absent target `template` copies all fetched keys as-is.
  `kubectl explain infisicalstaticsecret.spec.targets.template` shows that the
  optional template has only `data` and `engineVersion`; there is no
  `includeAllSecrets` switch. The official InfisicalStaticSecret documentation,
  under Templating, confirms the no-template behavior. Live key-name-only
  inspection found `ca.crt`, `tls.crt`, and `tls.key`. No Secret values were read.
  Omitting the template keeps the old copy-all behavior, including dotted key
  names and PEM content, rather than selecting or renaming keys.
- Before the first sync, the operator must orphan-delete the old CR so that its
  target Secret stays in place. This is an adoption step, not a command run by
  this task:

  ```sh
  kubectl delete infisicalsecret infisical-secret-operator-ca-tls-tenant-s3 \
    -n minio-operator --cascade=orphan
  ```

  Then sync the new chart manually. Check that the new InfisicalStaticSecret
  reports a successful sync. Check only the target Secret type and key names:
  `kubernetes.io/tls` with `ca.crt`, `tls.crt`, and `tls.key`. Keep the old source
  from reconciling during this handover.
- Chart 7.0.1 identifies its app version as `v7.0.1`, but the live operator image
  is `harbor.dev.raulstechtips.dev/raulstechtips/operator:v7.1.0`. This mismatch
  stays for parity. Tenant pods use `quay.io/minio/operator-sidecar:v7.0.1`.
  Do not add `OPERATOR_SIDECAR_IMAGE`: the live Deployment does not set it.
- The chart dependency no longer needs Harbor. The operator image still does.
  This bootstrap dependency is the remaining work in DEVSECOPS-185, "MinIO
  operator own chart + image from GHCR". Moving the image needs a push and is
  outside this task.
- The CA secret remains in this chart. The tenant chart is not changed.
- Local check results and the complete render-vs-live report are in
  `../.omo/evidence/wrappers/minio-operator/`.
