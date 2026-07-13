# Blue/Green Deployment avec Argo Rollouts

## Architecture

Le microservice `front` utilise une strategie blue/green via Argo Rollouts :

- **Active Service** (`front`) : pointe vers la version live (blue)
- **Preview Service** (`front-preview`) : pointe vers la nouvelle version (green)
- **Promotion manuelle** : `autoPromotionEnabled: false`
- **Rollback rapide** : l'ancienne version reste disponible 10 min (`scaleDownDelaySeconds: 600`)

## Flux de deploiement

### 1. Push d'une nouvelle version

Mettre a jour l'image tag dans :
```
environments/prod/40-applications/front/values.yaml
```

ArgoCD synchronise automatiquement. Argo Rollouts :
- Cree un nouveau ReplicaSet (green) avec la nouvelle image
- Le Service `front-preview` pointe vers les pods green
- Le Service `front` (active) continue de pointer vers les pods blue

### 2. Validation QA

Acceder a la version preview via l'Ingress dedie :
```
http://preview.formation.local/
```

Ou via port-forward :
```bash
kubectl port-forward svc/front-preview -n prod 8080:80
```

### 3. Promotion

Apres validation, promouvoir la version green en active :
```bash
kubectl argo rollouts promote front -n prod
```

Argo Rollouts :
- Bascule le Service `front` vers les pods green
- L'ancienne version (blue) reste disponible pendant 10 minutes

### 4. Rollback

Si un probleme est detecte apres promotion :
```bash
kubectl argo rollouts undo front -n prod
```

### 5. Surveillance

Etat du rollout :
```bash
kubectl argo rollouts get rollout front -n prod
kubectl describe rollout front -n prod
```

Dashboard Argo Rollouts :
```bash
kubectl port-forward svc/argo-rollouts-dashboard -n argo-rollouts 3100:3100
```

## Configuration

Le blue/green est active par environnement via les values gitops :

```yaml
rollout:
  enabled: true
```

Quand `rollout.enabled: false` (defaut), le chart deploie un Deployment standard avec RollingUpdate.

## Services et Ingress

| Ressource | Nom | Role |
|-----------|-----|------|
| Service active | `front` | Trafic live |
| Service preview | `front-preview` | Trafic QA/test |
| Ingress active | `front-ingress-prod` | Route `/` vers le service active |
| Ingress preview | `front-preview-ingress-prod` | Route `preview.formation.local` vers le service preview |
