# GitOps Setup — Mi App (Spring Boot + Jenkins + ArgoCD + GitLab)

## Estructura de repos

```
argo-central/                        ← repo ArgoCD (este)
└── apps/
    └── mi-app/
        ├── base/                    ← Helm chart base
        │   ├── Chart.yaml
        │   └── templates/
        │       ├── deployment.yaml
        │       └── service-ingress.yaml
        └── staging/
            ├── application.yaml     ← Application de ArgoCD
            └── values-staging.yaml  ← Jenkins actualiza image.tag aquí

app-repo/                            ← repo de la aplicación
├── jenkins/
│   └── Jenkinsfile                  ← pipeline CI/CD
├── k8s/                             ← manifiestos extra si es necesario
├── src/                             ← código Spring Boot
├── Dockerfile
└── .gitlab-ci.yml
```

---

## Orden de configuración

### 1. Crear los repos en GitLab

- `tu-grupo/mi-app` → repo de la aplicación
- `tu-grupo/argo-central` → repo de ArgoCD (este)

### 2. Configurar accesos (ver docs/gitlab-access-setup.md en el repo app)

```bash
# Generar llave para Jenkins → argo-central
ssh-keygen -t ed25519 -C "jenkins-argo-write" -f ~/.ssh/jenkins_argo_key

# Generar llave para ArgoCD → argo-central (solo lectura)
ssh-keygen -t ed25519 -C "argocd-read" -f ~/.ssh/argocd_read_key
```

Subir claves públicas como Deploy Keys en GitLab.
Subir clave privada de Jenkins a: Jenkins → Credentials → SSH Username with private key (ID: `argo-repo-ssh-key`)

### 3. Configurar Jenkins

En Jenkins → New Item → Multibranch Pipeline:
- Branch Sources: GitLab → URL del repo app
- Build Configuration: by Jenkinsfile → Path: `jenkins/Jenkinsfile`
- Scan Triggers: activar webhook (o polling como fallback)
- Credentials: agregar token de GitLab y credencial SSH para argo-central

### 4. Configurar webhook en GitLab

Settings → Webhooks:
- URL: `http://<jenkins-host>/project/mi-app`
- Trigger: Push events → rama `develop`

### 5. Aplicar Application en ArgoCD

```bash
# Conectar ArgoCD al repo argo-central (si es privado)
argocd repo add git@gitlab.com:tu-grupo/argo-central.git \
  --ssh-private-key-path ~/.ssh/argocd_read_key

# Aplicar el Application de staging
kubectl apply -f apps/mi-app/staging/application.yaml -n argocd

# Verificar
argocd app get mi-app-staging
```

### 6. Crear el namespace de staging en Kubernetes

```bash
kubectl create namespace staging
```

### 7. Crear el secret de la app en staging

```bash
kubectl create secret generic mi-app-staging-secrets \
  --from-literal=DB_URL="jdbc:postgresql://host:5432/staging_db" \
  --from-literal=DB_PASSWORD="cambiar-esto" \
  -n staging
```

---

## Flujo automático tras el setup

```
Desarrollador → git push → develop
        ↓
GitLab dispara webhook
        ↓
Jenkins: build → test → docker build → docker push (registry GitLab)
        ↓
Jenkins: git clone argo-central
         sed -i "tag: <nuevo-tag>" staging/values-staging.yaml
         git commit + push
        ↓
ArgoCD detecta el commit en argo-central (polling cada 3 min o webhook)
        ↓
ArgoCD sync automático → Deployment actualizado en namespace staging
```

---

## Comandos útiles

```bash
# Ver estado del sync
argocd app get mi-app-staging

# Forzar sync manual
argocd app sync mi-app-staging

# Ver historial de deploys
argocd app history mi-app-staging

# Rollback a versión anterior
argocd app rollback mi-app-staging <ID>

# Ver pods en staging
kubectl get pods -n staging

# Logs de la app
kubectl logs -f deploy/mi-app-staging -n staging
```

---

## Consideraciones de seguridad

- El desarrollador NUNCA tiene acceso al repo `argo-central`.
- El desarrollador NUNCA puede hacer push directo a `main`.
- Jenkins es el único que escribe en `argo-central` (via deploy key con write).
- ArgoCD solo lee `argo-central` (deploy key sin write).
- Los secrets de la app viven en Kubernetes, no en git.
- Rotar las deploy keys y tokens cada 6-12 meses.
