# OAuth2-Proxy Installation and Configuration


## Prerequisites

Add the required charts repo to Helm:
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
```


## Local

1. Generate a strong cookie secret:
```shell
dd if=/dev/urandom bs=32 count=1 2>/dev/null | base64 | tr -d -- '\n' | tr -- '+/' '-_'; echo
```
2. Modify the lcl.values.yaml configuration with the secret generated in both location.

3. From the Keycloak client admin page, copy-paste the client secret and modify the lcl.values.yaml configuration file with the client secret.


4. Create the namespace with Istio enabled:
```shell
kubectl apply -f namespace.yaml
```

5. The Redis secret is delivered by External Secrets, not created by hand.

   `base.values.yaml` sets `redis.auth.existingSecret: oauth2-proxy-redis-secret`, and
   the `boucio-oauth2-proxy-redis-secret` ExternalSecret in
   `infrastructure/external-secrets/externalsecrets/infra/<env>/` produces it (key
   `redis-password`, which is the Bitnami default when
   `redis.auth.existingSecretPasswordKey` is empty). Set the source value
   `boucio-oauth2proxy-redis-password` once: on local in the gitignored
   `boucio-local-external-credentials.yaml`, on sandbox in GCP Secret Manager. See
   `infrastructure/external-secrets/README.md`.

   The local-only `redis-credentials-secret.yaml` (gitignored) is the pre-ESO path and
   is no longer needed once the ExternalSecret is reconciling.

6. Install OAuth2-Proxy.
```shell
helm upgrade -install oauth2-proxy-rel -f base.values.yaml -f lcl.values.yaml bitnami/oauth2-proxy -n oauth2-proxy
```



## Sandbox

kubectl apply -f namespace.yaml


The Redis credentials come from External Secrets (see step 5 above); there is no
manual secret step here anymore.

Note the name: the chart consumes `oauth2-proxy-redis-secret`, which is what the
ExternalSecret produces. Earlier revisions of this README documented creating
`oauth2proxy-db-secret`, a name nothing actually reads. If you created that Secret on
a cluster, it is inert and can be removed.

For a standalone install outside this GitOps setup:

kubectl create secret generic oauth2-proxy-redis-secret \
  --from-literal='username=admin' \
  --from-literal='redis-password=<YOUR-REDIS-PASSWORD>' \
  -n oauth2-proxy

helm upgrade -install oauth2-proxy-rel -f base.values.yaml -f snbx.values.yaml bitnami/oauth2-proxy -n oauth2-proxy -kubeconfig <KUBECONFIG-FILE>



## Common error on Local setup

To resolve this error:

[2024/10/23 15:59:43] [provider.go:55] Performing OIDC Discovery...
[2024/10/23 15:59:43] [main.go:60] ERROR: Failed to initialise OAuth2 Proxy: error intiailising provider: could not create provider data: error building OIDC ProviderVerifier: could not get verifier builder: error while discovery OIDC configuration: failed to discover OIDC configuration: error performing request: Get "https://sso.docker.internal/realms/users/.well-known/openid-configuration": read tcp 10.1.16.77:45376->10.1.16.47:443: read: connection reset by peer

Validate the IP address in the hostAliases section of the lcl.values.yaml file.

*Section*:
hostAliases: #[]
  - ip: "10.96.145.240" ## TODO: this IP will change constantly technically (IP for the Istio ingress); how to make permanent??
    hostnames:
      - "sso.docker.internal"

To obtain the proper IP address for the Istio ingress, run the following command:

```shell
kubectl get svc istio-ingressgateway -n istio-system --cluster docker-desktop
```


kubectl patch deployment oauth2-proxy-rel -n oauth2-proxy -p '{"spec":{"template":{"spec":{"containers":[{"name":"oauth2-proxy","args":["--oidc-issuer-url=https://sso.docker.internal/auth"]}]}}}}'



# References

1. https://aaronparecki.com/oauth-2-simplified/
2. https://github.com/bitnami/charts/tree/master/bitnami/oauth2-proxy/#installing-the-chart
3. https://medium.com/@senthilrch/api-access-control-using-istio-ingress-gateway-44be659a087e
4. https://medium.com/@senthilrch/api-authentication-using-istio-ingress-gateway-oauth2-proxy-and-keycloak-a980c996c259
5. https://medium.com/@senthilrch/api-authentication-using-istio-ingress-gateway-oauth2-proxy-and-keycloak-part-2-of-2-dbb3fb9cd0d0
6. https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#keycloak-oidc-auth-provider

 

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
