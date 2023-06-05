# ArgoCD with Helm Sops

ArgoCD image with Helm-Sops support. Helm Sops is a Helm wrapper that decrypts SOPs encrypted value files before invoking Helm.

The following tools have been added to the image:

- GnuPG
- [Helm Sops](https://github.com/camptocamp/helm-sops)

ArgoCD repository server binary is wrapped by a shell script which can import a GPG private key if it exists. The key must be located at `/app/config/gpg/privkey.asc`.


### Custom image

To use this custom sops supported image when deploying ArgoCD using the [Helm chart](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd), add the following lines to the chart value file:

```yaml
global:
  image:
    repository: "thenaim/argocd"
    tag: "v2.6.9"
```

### Sops with an AWS KMS key

#### Method 1: [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
This is an example values file for the ArgoCD Server [Helm chart](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd):

```bash
repoServer:
  serviceAccount:
    create: true
    name: "argocd-repo-server"
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/iam-role-name
    automountServiceAccountToken: true
```

#### Method 2: [instance profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)

#### Method 3: If IRSA/Instance profiles are not available

Add the following lines to the chart value file:

```yaml
repoServer:
  env:
    - name: "AWS_ACCESS_KEY_ID"
      valueFrom:
        secretKeyRef:
          name: "argocd-secret"
          key: "aws.accessKeyId"
    - name: "AWS_SECRET_ACCESS_KEY"
      valueFrom:
        secretKeyRef:
          name: "argocd-secret"
          key: "aws.secretAccessKey"
```

and add the following lines to an encrypted value file (create a dedicated IAM Access Key):

```yaml
configs:
  secret:
    extra:
      aws.accessKeyId: <Access Key ID>
      aws.secretAccessKey: <Secret Access Key>
```

### Sops with a GPG key

In order to use Sops with a GPG key, add the following lines to the chart value file:

```yaml
global:
  securityContext:
    fsGroup: 2000

repoServer:
  volumes:
    - name: "gpg-private-key"
      secret:
        secretName: "argocd-secret"
        items:
          - key: "gpg.privkey.asc"
            path: "privkey.asc"
        defaultMode: 0600
  volumeMounts:
    - name: "gpg-private-key"
      mountPath: "/app/config/gpg/privkey.asc"
      subPath: "privkey.asc"
```

and add the following lines to an encrypted value file (the GPG private key can be exported by running `gpg --export-secret-keys --armor <key ID>`:

```yaml
configs:
  secret:
    extra:
      gpg.privkey.asc: |
        -----BEGIN PGP PRIVATE KEY BLOCK-----
        
        ...
        -----END PGP PRIVATE KEY BLOCK-----
```
