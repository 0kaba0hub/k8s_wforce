# k8s_wforce

Helm chart deploying [PowerDNS weakforced](https://github.com/PowerDNS/weakforced)
into the sandbox Kubernetes cluster as the auth policy server for the mail
servers there. Every server pointed at it is judged by the same policy, and
each request body is logged field by field, so what two senders put in a body
for the same login can be compared in one log.

Not a production deployment: single replica, no replication, no TLS on the
API (cluster-internal only).

## What it does

- Deploys `powerdns/wforce:v3.2.0` with a `wforce.conf` rendered from
  `helm/files/wforce.conf`: the image's example policy, counting different
  failed passwords per IP, per IP+login and per login over one hour.
- `allow` answers a tarpit past `policy.tarpitAfter` per IP+login, a
  refusal with `policy.refuseMessage` for the client past `policy.refuseAfter`
  per IP, and a tarpit past `policy.loginTarpitAfter` per login from any IP.
  `report` feeds the counts from failed logins.
- With `policy.logRequests`, every `allow` and `report` logs every field it
  received (`login`, `remote`, `pwhash`, `protocol`, `tls`, `device_id`,
  `session_id`, `fail_type`, `success`, `policy_reject`, `attrs`): what a sender put in
  the body is read from `kubectl -n wforce logs deploy/wforce`.
- The API listens on `:8084`, behind Service `wforce`; wforce's ACL admits the
  private ranges by default (`acl` narrows it).

## Deploy

The Secret is made once, outside git; the chart only references it:

```sh
kubectl --kubeconfig ~/.kube/ihorru-sbox-nc.yaml create namespace wforce
kubectl --kubeconfig ~/.kube/ihorru-sbox-nc.yaml -n wforce create secret generic wforce \
  --from-literal=password="$(openssl rand -hex 24)" \
  --from-literal=key="$(openssl rand -base64 32)"

helm upgrade --install wforce helm -n wforce \
  --kubeconfig ~/.kube/ihorru-sbox-nc.yaml \
  -f helm_values/values-sandbox.yaml
```

A client reads the password from the same Secret and sends it as HTTP basic
auth (`Authorization: Basic base64(wforce:<password>)`).

## Values

| Key | Default | Description |
|:---|:---|:---|
| `image.repository` / `image.tag` | `powerdns/wforce` / `v3.2.0` | wforce image |
| `replicas` | `1` | Pods; more than one would need wforce replication, not configured |
| `port` | `8084` | API port, container and Service |
| `existingSecret.name` / `passwordKey` / `keyKey` | `wforce` / `password` / `key` | Secret with the API password and the wforce key |
| `acl` | `[]` | Networks allowed to call the API; empty keeps wforce's private-range default |
| `policy.tarpitAfter` / `policy.tarpitSecs` | `3` / `3` | Different failed passwords per IP+login before `allow` tarpits, and for how long |
| `policy.refuseAfter` / `policy.refuseMessage` | `50` / `Too many failed logins, try again later` | Different failed passwords per IP before `allow` refuses, and the message for the client |
| `policy.loginTarpitAfter` / `policy.loginTarpitSecs` | `20` / `10` | Different failed passwords per login, from any IP, before `allow` tarpits, and for how long |
| `policy.logRequests` | `true` | Log every field of every `allow` and `report` |
| `resources` | `50m/64Mi` request, `512Mi` limit | Pod sizing |

## Probing it

```sh
kubectl -n wforce port-forward svc/wforce 18084:8084 &
PW=$(kubectl -n wforce get secret wforce -o go-template='{{.data.password|base64decode}}')
curl -s -u wforce:"$PW" -H 'Content-Type: application/json' \
  -d '{"login":"probe@test.invalid","remote":"192.0.2.10","pwhash":"ab","protocol":"imap","tls":true,"session_id":"s1","device_id":"","attrs":{}}' \
  'http://127.0.0.1:18084/?command=allow'
# {"msg": "", "r_attrs": {}, "status": 0}
curl -s -u wforce:"$PW" -H 'Content-Type: application/json' \
  -d '{"login":"probe@test.invalid","ip":"192.0.2.10"}' 'http://127.0.0.1:18084/?command=reset'
```

`status` is `0` to allow, a positive number of tarpit seconds, or `-1` to
refuse with `msg`. Without the password the API answers `401`.
