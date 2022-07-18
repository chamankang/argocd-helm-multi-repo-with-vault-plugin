# argocd-helm-multi-repo-with-vault-plugin
'helm.argocd.yaml' is a helm override values file for [argo-cd chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd).

It configures an Argo CD custom plugin that enables Applications to build helm template with a helm chart and an override values file from two separated repositories.

It also enables to use [Argo CD Vault Plugin](https://argocd-vault-plugin.readthedocs.io/en/stable/). It configures AWS Secrets Manager as a secrets backend.

Deploying Argo CD with this override values file, 'argocd' Application is created. It makes Argo CD to manage itself as an Application, downloads argo-cd chart from [argo-cd chart repo](https://argoproj.github.io/argo-helm) and syncs with the override values in [this repository](https://github.com/chamankang/argocd-helm-multi-repo-with-vault-plugin). It also uses argocd-vault-plugin to get and protect secrets.

## helm.argocd.yaml
The custom plugin name is argocd-vault-plugin-helm. It builds helm template with env variables set in its Application and replaces placeholders with the secrets in AWS Secrets Manager.
```
server:
  config:
    configManagementPlugins: |
      - name: argocd-vault-plugin-helm
        generate:
          command: ["sh", "-c"]
          args: [
            "helm template $ARGOCD_APP_NAME ${ARGOCD_ENV_CHART_NAME:=$ARGOCD_APP_NAME} --repo $ARGOCD_ENV_CHART_REPO --version $ARGOCD_ENV_CHART_VERSION --values $ARGOCD_ENV_VALUES $ARGOCD_ENV_EXTRA_HELM_ARGS | argocd-vault-plugin generate -"
          ]
```

'argocd' Application syncs with [this repository](https://github.com/chamankang/argocd-helm-multi-repo-with-vault-plugin). It is used to manage Argo CD itself after the initial deployment.
```
server:
  additionalApplications:
    - name: argocd
      namespace: argocd
      project: default
      source:
        repoURL: 'git@github.com:chamankang/argocd-helm-multi-repo-with-vault-plugin.git'
        path: .
        targetRevision: HEAD
        plugin:
          name: argocd-vault-plugin-helm
          env:
            - name: CHART_NAME
              value: argo-cd
            - name: CHART_REPO
              value: https://argoproj.github.io/argo-helm
            - name: CHART_VERSION
              value: 4.9.8
            - name: VALUES
              value: helm.argocd.yaml
            - name: EXTRA_HELM_ARGS
              value: --include-crds
      destination:
        name: in-cluster
        namespace: argocd
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

The secrets are stored in AWS Secrets Manager in ap-northeast-2 region. Argo CD Repo Server can access to the secrets using AWS IAM Role for Service Account. With this override values file, the role's ARN should be stored in AWS secret named 'argocd'. If Argo CD isn't to be deployed to EKS cluster or IRSA is not usable for the EKS cluster, annotations should be removed and AWS credential should be passed to repoServer as env vars.
```
repoServer:
  replicas: 2
  env:
  - name: AVP_TYPE
    value: "awssecretsmanager"
  - name: AWS_REGION
    value: "ap-northeast-2"
  serviceAccount:
    create: true
    annotations:
      eks.amazonaws.com/role-arn: <path:argocd#repoServer.serviceAccount.role-arn>
```
The role for service account must be an IAM OIDC federated role and have permissions to read secret values like this;
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "*"
        }
    ]
}
```
The ssh key set in credentialTemplate is used to access to any repo starting with 'git@github.com:chamankang/'. The value of 'sshPrivateKey' is a placeholder to be replaced by argocd-vault-plugin with the secret value corresponding to the secret key 'credentialTemplate.chamankang.sshPrivateKey' of 'argocd' secret in AWS Secrets Manager. The values of 'argocdServerAdminPassword' and 'server.secretkey' are also replaced with 'argocd' secret values.
```
configs:
  credentialTemplates:
    chamankang:
      url: git@github.com:chamankang/
      sshPrivateKey: |
        <path:argocd#credentialTemplate.chamankang.sshPrivateKey>
  repositories:
    argocd-helm-multi-repo-with-vault-plugin:
      url: git@github.com:chamankang/argocd-helm-multi-repo-with-vault-plugin
      name: argocd-helm-multi-repo-with-vault-plugin
      project: default
  secret:
    createSecret: true
    argocdServerAdminPassword: <path:argocd#admin.password>
    argocdServerAdminPasswordMtime: "2022-07-03T16:05:17Z"
    extra:
      server.secretkey: <path:argocd#server.secretkey>
```

## How to deploy
```
kubectl create namespace argocd
git clone https://github.com/chamankang/argocd-helm-multi-repo-with-vault-plugin
cd argocd-helm-multi-repo-with-vault-plugin
export AVP_TYPE=awssecretsmanager
export AWS_REGION=ap-northeast-2
export AWS_PROFILE=<AWS account profile to access to Secrets Manager>
export ARGOCD_APP_NAME=argocd
export ARGOCD_ENV_CHART_NAME=argo-cd
export ARGOCD_ENV_CHART_REPO=https://argoproj.github.io/argo-helm
export ARGOCD_ENV_CHART_VERSION=4.9.8
export ARGOCD_ENV_VALUES=helm.argocd.yaml
export ARGOCD_ENV_EXTRA_HELM_ARGS=--include-crds
helm template $ARGOCD_APP_NAME ${ARGOCD_ENV_CHART_NAME:=$ARGOCD_APP_NAME} --repo $ARGOCD_ENV_CHART_REPO --version $ARGOCD_ENV_CHART_VERSION --values $ARGOCD_ENV_VALUES $ARGOCD_ENV_EXTRA_HELM_ARGS -n argocd | argocd-vault-plugin generate - | kubectl apply -f- -n argocd
```

## How to connect to ArgoCD UI
```
sudo -E kubefwd svc -n argocd
```
Access to http://argocd-server and login as the admin user.
