# OpenShift: Recovering Expired cert-manager Certificates (DigitalOcean DNS-01)

Runbook for when cert-manager fails to renew certificates (e.g. `api-certs`, `router-certs`) because the DNS-01 solver can no longer authenticate to DigitalOcean. Uses `oc` only; no `cmctl` required.

## Symptoms

- `oc get certificate -A` shows `READY=False`
- Certificates are well past their expected renewal date (Let's Encrypt certs last 90 days and renew at ~60)
- cert-manager logs show a 401 from the DigitalOcean API

## 1. Diagnose

```bash
oc get certificate -A
oc describe certificate <name> -n <ns>        # check Status.Conditions and events
oc get certificaterequest,order,challenge -A
oc get clusterissuer,issuer -A
oc logs -n cert-manager deploy/cert-manager --tail=200 | grep -iE 'error|fail'
```

Root cause in this case:

```
GET https://api.digitalocean.com/v2/domains/<domain>/records?type=TXT: 401 ... Unable to authenticate you
```

The DNS-01 solver could not write `_acme-challenge` TXT records, so every renewal failed.

## 2. Fix the credential

1. In DigitalOcean: **API → Tokens → Generate New Token**. Use a custom scope limited to the `domain` resource (create, read, update, delete) and a long expiry.

2. Find the secret the issuer reads:

   ```bash
   oc get clusterissuer -o yaml | grep -B2 -A6 digitalocean
   ```

   Note `tokenSecretRef.name` and `tokenSecretRef.key`. For a ClusterIssuer the secret lives in the `cert-manager` namespace (the cluster resource namespace). For a namespaced Issuer, it lives in the Issuer's namespace.

3. Replace the secret. No pod restart is needed; cert-manager reads it at solve time.

   ```bash
   read -rs DO_TOKEN
   oc create secret generic <secret-name> -n cert-manager \
     --from-literal=<key>=$DO_TOKEN --dry-run=client -o yaml | oc apply -f -
   unset DO_TOKEN
   ```

   (`read -s` keeps the token out of shell history. In a hand-written manifest, use `stringData` for plain text; `data` expects base64.)

## 3. Force reissue

Failed issuance triggers exponential backoff (up to 32h), so fixing the secret alone does not retry immediately. Without `cmctl renew`, recreate the Certificate to wipe its failure history.

For each Certificate (here: `api-certs` in `openshift-config`, `router-certs` in `openshift-ingress`):

```bash
# Save the spec first (or use the manifest from git)
oc get certificate router-certs -n openshift-ingress -o yaml > router-certs.yaml
# Edit the file: remove the status: block and metadata uid, resourceVersion,
# creationTimestamp, generation and managedFields

oc delete certificate router-certs -n openshift-ingress
oc delete secret router-certs -n openshift-ingress --ignore-not-found
oc apply -f router-certs.yaml
```

Clear any failed leftovers if needed:

```bash
oc delete certificaterequest,order,challenge --all -n <ns>
```

Watch it progress (expect CertificateRequest → Order → Challenge pending → valid):

```bash
oc get certificate,certificaterequest,order,challenge -n <ns> -w
```

### Gotcha: stale ACME order

If `oc describe certificate` shows:

```
order is in "errored" state: Failed to retrieve Order resource: 404 urn:ietf:params:acme:error:malformed: No order for ID ...
```

Let's Encrypt has pruned the old order. This leaves the Certificate idle in backoff with no CertificateRequest. Deleting the secret alone does not clear it; recreate the Certificate as above.

## 4. Verify

```bash
# Certificates ready
oc get certificate -A

# Fresh cert in the secret
oc get secret <name> -n <ns> -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -subject -dates

# Cluster operators healthy
oc get co ingress kube-apiserver

# What clients actually see
echo | openssl s_client -connect <app-route-host>:443 -servername <app-route-host> 2>/dev/null \
  | openssl x509 -noout -dates -issuer
echo | openssl s_client -connect <api-host>:6443 -servername <api-host> 2>/dev/null \
  | openssl x509 -noout -dates -issuer
```

Healthy: both Certificates `READY=True`, `notAfter` ~90 days out, Let's Encrypt issuer, and both operators Available with `Degraded=False`. The kube-apiserver rollout after a cert change can take several minutes (`Progressing=True`).

## 5. Router still serving the old cert?

The ingress operator normally rolls the router when the secret changes. Check the pods are newer than the secret:

```bash
oc get pods -n openshift-ingress
oc get secret router-certs -n openshift-ingress -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
```

If not, restart with a rollout (not by deleting pods):

```bash
oc rollout restart deployment/router-default -n openshift-ingress
oc rollout status deployment/router-default -n openshift-ingress
```

Confirm the ingresscontroller references the right secret:

```bash
oc get ingresscontroller default -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate.name}{"\n"}'
```

The API server needs no pod action; the kube-apiserver operator rolls its own static pods.

## 6. Prevent recurrence

- Delete stale `_acme-challenge` TXT records left in DigitalOcean by the failed attempts.
- Add Prometheus alerts on:
  - `certmanager_certificate_expiration_timestamp_seconds` (e.g. under 21 days remaining)
  - a Certificate stuck not-ready (`certmanager_certificate_ready_status{condition="False"}`)
- Record the new DigitalOcean token's expiry date and set a reminder well before it.
- Optional: if DNS self-checks hang, point cert-manager at working resolvers with `--dns01-recursive-nameservers=1.1.1.1:53,8.8.8.8:53` and `--dns01-recursive-nameservers-only`.
