1. Init container to install plugin secret in ArgoCD
repoServer:
  env:
    - name: HELM_PLUGINS
      value: /gitops-tools/helm-plugins/
    - name: HELM_SECRETS_CURL_PATH
      value: /gitops-tools/curl
    - name: HELM_SECRETS_SOPS_PATH
      value: /gitops-tools/sops
    - name: HELM_SECRETS_VALS_PATH
      value: /gitops-tools/vals
    - name: HELM_SECRETS_AGE_PATH
      value: /gitops-tools/age
    - name: HELM_SECRETS_KUBECTL_PATH
      value: /gitops-tools/kubectl
    - name: HELM_SECRETS_BACKEND
      value: sops
    # https://github.com/jkroepke/helm-secrets/wiki/Security-in-shared-environments
    - name: HELM_SECRETS_VALUES_ALLOW_SYMLINKS
      value: "false"
    - name: HELM_SECRETS_VALUES_ALLOW_ABSOLUTE_PATH
      value: "true"
    - name: HELM_SECRETS_VALUES_ALLOW_PATH_TRAVERSAL
      value: "false"
    - name: HELM_SECRETS_WRAPPER_ENABLED
      value: "true"
    - name: HELM_SECRETS_DECRYPT_SECRETS_IN_TMP_DIR
      value: "true"
    - name: HELM_SECRETS_HELM_PATH
      value: /usr/local/bin/helm
    - name: HELM_SECRETS_LOAD_GPG_KEYS
      # Multiple keys can be separated by space
      value: /helm-secrets-private-keys/key.asc
  volumes:
    - name: gitops-tools
      emptyDir: {}
    # kubectl create secret generic helm-secrets-private-keys --from-file=key.asc=assets/gpg/private2.gpg
    - name: helm-secrets-private-keys
      secret:
        secretName: helm-secrets-private-keys
  volumeMounts:
    - mountPath: /gitops-tools
      name: gitops-tools
    - mountPath: /usr/local/sbin/helm
      subPath: helm
      name: gitops-tools
    - mountPath: /helm-secrets-private-keys/
      name: helm-secrets-private-keys
  initContainers:
    - name: download-tools
      image: alpine:latest
      imagePullPolicy: IfNotPresent
      command: [sh, -euc]
      env:
        - name: HELM_SECRETS_VERSION
          value: "4.6.10"
        - name: KUBECTL_VERSION
          value: "1.34.1"
        - name: VALS_VERSION
          value: "0.42.4"
        - name: SOPS_VERSION
          value: "3.11.0"
        - name: AGE_VERSION
          value: "1.2.1"
        - name: HELM_PLUGINS
          value: /gitops-tools/helm-plugins/
        - name: HELM_SECRETS_CURL_PATH
          value: /gitops-tools/curl
        - name: HELM_SECRETS_SOPS_PATH
          value: /gitops-tools/sops
        - name: HELM_SECRETS_VALS_PATH
          value: /gitops-tools/vals
        - name: HELM_SECRETS_AGE_PATH
          value: /gitops-tools/age
        - name: HELM_SECRETS_KUBECTL_PATH
          value: /gitops-tools/kubectl
      args:
        - |
          mkdir -p "${HELM_PLUGINS}"
          export CURL_ARCH=$(uname -m | sed -e 's/x86_64/amd64/')
          wget -qO "${HELM_SECRETS_CURL_PATH}" https://github.com/moparisthebest/static-curl/releases/latest/download/curl-${CURL_ARCH}
          export GO_ARCH=$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')
          wget -qO "${HELM_SECRETS_KUBECTL_PATH}" https://dl.k8s.io/release/v${KUBECTL_VERSION}/bin/linux/${GO_ARCH}/kubectl
          wget -qO "${HELM_SECRETS_SOPS_PATH}" https://github.com/getsops/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.${GO_ARCH}
          wget -qO- https://github.com/helmfile/vals/releases/download/v${VALS_VERSION}/vals_${VALS_VERSION}_linux_${GO_ARCH}.tar.gz | tar zxv -C "${HELM_SECRETS_VALS_PATH%/*}" vals
          wget -qO- https://github.com/jkroepke/helm-secrets/releases/download/v${HELM_SECRETS_VERSION}/helm-secrets.tar.gz | tar -C "${HELM_PLUGINS}" -xzf-
          wget -qO- "https://github.com/FiloSottile/age/releases/download/v${AGE_VERSION}/age-v${AGE_VERSION}-linux-amd64.tar.gz" | tar -xzf- --strip-components=1 -C "${HELM_SECRETS_AGE_PATH%/*}" age/age
          chmod +x \
            "${HELM_SECRETS_CURL_PATH}" \
            "${HELM_SECRETS_SOPS_PATH}" \
            "${HELM_SECRETS_KUBECTL_PATH}" \
            "${HELM_SECRETS_VALS_PATH}" \
            "${HELM_SECRETS_AGE_PATH}"
          cp "${HELM_PLUGINS}/helm-secrets/scripts/wrapper/helm.sh" /gitops-tools/helm
      volumeMounts:
        - mountPath: /gitops-tools
          name: gitops-tools
2. Allow helm-secrets schemes in argocd-cm ConfigMap
In argocd.yaml , we add schemes of helm-secrets
configs:
  cm:
    helm.valuesFileSchemes: >-
      secrets+gpg-import, secrets+gpg-import-kubernetes,
      secrets+age-import, secrets+age-import-kubernetes,
      secrets, secrets+literal,
      https
3.Generating the key 
Run this command to create GPG Key 
gpg --full-generate-key --rfc4880
When asked to enter a password, you need to omit it.
4. Creating the kubernetes secret holding the exported private key
Run this command to get your private key:
gpg --export-secret-keys --armor YOUR_FINGERPRINT > private-key-no-pass.asc
Create a kubernetes secret
kubectl -n argocd create secret generic helm-secrets-private-keys --from-file=key.asc=./private-key-no-pass.asc
5. Write application file for app 
This is an example file application:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-app-secret
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/KhacThien0808/helm-secret.git
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - secrets://values.yaml
  destination:
    server: https://kubernetes.default.svc 
    namespace: test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
6. Run this command to encrypt file (or specific field):
sops --encrypt --in-place --encrypted-regex '^(mySecretPassword)$' values.yaml
It will look like that after I run command to encrypt field mySecretPassword
# values.yaml
replicaCount: 1
image:
    repository: nginx
    pullPolicy: IfNotPresent
    tag: 1.21.6
mySecretPassword: ENC[AES256_GCM,data:+JvLri9zN9f1FQf4AUpK9g==,iv:EpyQuy4kxjhm/hjX1KTr3AYkY6cxE3tNWtqHIUvYU6E=,tag:fJFMwX3n4wpKuU0RxTryYw==,type:str]
sops:
    lastmodified: "2025-10-15T03:23:45Z"
    mac: ENC[AES256_GCM,data:J+mzvCiIcAhUWGpizQHXgwKh7isI/axg/fdmIXmTw/IjGp16zwZcChXz/uRtxa4iZGJ6fAkuH51jGiRiCDN6NaiKfREpF75hqCmmC09kNJO3B7xbVbaWm1ggIlQeH5fXdJivgaEntH1rdNmFBWxYeGmXbT1mNY82dltRPD5Iob8=,iv:HaeUVXvYH2f+TtB5Cu/klPVZER2MzQuOVY8n12qsg/g=,tag:9bGRdE7i7iUVRNdG/WRocw==,type:str]
    pgp:
        - created_at: "2025-10-15T03:23:45Z"
          enc: |-
            -----BEGIN PGP MESSAGE-----
            hF4DHICC/VIf2eASAQdAa2Yhb5qTmzrX9pHNU0IgmnDovWfJl/CoM1Lk6vBk9mMw
            9++W0NOXoWlEPE+ooaWtDw2tFzt+Q6TQz94ntMfcH5bFquzA5nmbtJKbw7Qzq86i
            1GgBCQIQ3+zg0+uDgPTyG3Si+DJW3jYbos3p4zv+P7Hlach45We6p63LT79xE1Eh
            sTdqri2J9c9tLgDqM2z8pX8GZhKoTbTsydgDDnF50UKkn0AtyUbd7MGh5UxiTMrR
            ZzG97+9BAQTJtQ==
            =VrLq
            -----END PGP MESSAGE-----
          fp: 7E379DCF9EB4EF47B7396B37F75F5CA1FB03EDE9
    encrypted_regex: ^(mySecretPassword)$
    version: 3.11.0
