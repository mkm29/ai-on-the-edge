# Let's Encrypt

In order for this to work, you must install `cert-manager`, otherwise the cluster will not have any of these resources. 

## Create Cluster Issuer

### Staging  

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: mitch@smigula.io
    privateKeySecretRef:
      name: letsencrypt-staging
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: traefik
```

### Prod

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: mitch@smigula.io
    privateKeySecretRef:
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: traefik
```

## Redirect HTTP

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
name: redirect-https
  namespace: default
spec:
  redirectScheme:
    permanent: true
    scheme: https
```

## Ingress

Apply these to your cluster (they will live in the default namespace). Once created you are ready to protect Ingress routes! This is an example of the `Ingress` that I created for my instance of Keycloak. The basic idea is that you need to specify a few annotations that will alert the mutating admission controller to reach out to Let's Encrypt, procure the TLS cert and then store it in a secret (in the same namespace as the Ingress resource). _Note_: in order to complete the ACME HTTP challenge you must not put the redirect annotations in the `Ingress` resource until after the challenge completes and the TLS secret is created.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    ingress.kubernetes.io/ssl-proxy-headers: X-Forwarded-Proto:https
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/redirect-entry-point: https
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
    traefik.ress.kubernetes.io/whitelist-x-forwarded-for: "true"
  name: keycloak-ingress
  namespace: keycloak
spec:
  rules:
  - host: sso.smigula.io
    http:
      paths:
      - backend:
          service:
            name: keycloak-svc
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - sso.smigula.io
    secretName: sso-smigula-io-tls
```

### Attach to Pod

Once the challenge is complete and the secret contains the key/cert, you will need to make sure that it is attached to the Pod. Note that this is not necessary for all deployments, take Keycloak for example, this Pod requires you to specify the secret containing the cert. The easiest way to do this is to mount the secret as a volume:  

```yaml
spec:
  volumes:
  - name: certs-volume
    secret:
      defaultMode: 420
      secretName: sso-smigula-io-tls
```

You then need to mount this volume into the Pod (the `mountPath` will vary depending on the application):  

```yaml
...
spec:
  containers:
  - name: keycloak-pod
    ...
    volumeMounts:
    - mountPath: /etc/x509/https
      name: certs-volume
```
