# MongoDB OpenShift Deployment

Despliegue automatizado de MongoDB en OpenShift usando Helm y GitHub Actions.

## Características

- ✅ Despliegue automático con GitHub Actions
- ✅ 2 réplicas de MongoDB
- ✅ Persistencia de datos habilitada
- ✅ Configuración mediante Helm values

## Uso

### Despliegue automático
Haz push a la rama `main` para desplegar automáticamente:
```bash
git add .
git commit -m "Deploy MongoDB"
git push origin main
```

### Despliegue manual
Ve a **Actions** → **Deploy MongoDB to OpenShift** → **Run workflow**

## Configuración

Edita `helm/mongodb-values.yaml` para cambiar la configuración:
```yaml
replicaCount: 2  # Cambia el número de réplicas
```

## Verificación local
```bash
oc get pods -l app=mongodb
helm status my-mongodb
```
