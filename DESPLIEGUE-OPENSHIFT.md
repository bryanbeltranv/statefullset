# Despliegue de PostgreSQL StatefulSet en OpenShift

## Prerrequisitos

1. Tener acceso a un cluster de OpenShift
2. Tener `oc` CLI instalado
3. Estar autenticado en el cluster

## Pasos para el despliegue

### 1. Autenticarse en OpenShift

```bash
# Obtener el token desde la consola web de OpenShift
# Web Console → Click en tu usuario → Copy login command
oc login --token=<tu-token> --server=<url-del-servidor>
```

### 2. Crear un nuevo proyecto (namespace)

```bash
# Crear proyecto
oc new-project postgres-db

# O usar un proyecto existente
oc project mi-proyecto-existente
```

### 3. Configurar permisos de seguridad (IMPORTANTE)

OpenShift tiene Security Context Constraints (SCC) más estrictas que Kubernetes vanilla. Necesitamos dar permisos adicionales:

```bash
# Dar permisos al ServiceAccount default para ejecutar como anyuid
# Esto es necesario porque PostgreSQL necesita ejecutarse con un UID específico
oc adm policy add-scc-to-user anyuid -z default

# Alternativamente, si quieres usar un ServiceAccount específico:
oc create serviceaccount postgres-sa
oc adm policy add-scc-to-user anyuid -z postgres-sa
```

### 4. Desplegar el StatefulSet

```bash
# Aplicar el archivo YAML
oc apply -f postgres-statefulset.yaml
```

### 5. Verificar el despliegue

```bash
# Ver los pods creándose
oc get pods -w

# Ver el estado del StatefulSet
oc get statefulset postgres

# Ver los servicios
oc get svc

# Ver los PersistentVolumeClaims
oc get pvc

# Ver logs del pod maestro
oc logs -f postgres-0

# Ver logs de una réplica
oc logs -f postgres-1
```

### 6. Verificar la replicación

```bash
# Conectarse al pod maestro
oc exec -it postgres-0 -- psql -U postgres

# Dentro de psql, verificar el estado de replicación:
SELECT * FROM pg_stat_replication;

# Debería mostrar 2 réplicas conectadas
```

### 7. Probar la conectividad

#### Desde dentro del cluster:

```bash
# Crear un pod temporal para probar
oc run -it --rm --restart=Never postgres-client --image=postgres:15-alpine -- psql -h postgres-master -U postgres -d mydb

# Para lecturas (balancea entre réplicas)
oc run -it --rm --restart=Never postgres-client --image=postgres:15-alpine -- psql -h postgres-read -U postgres -d mydb
```

#### Exponer fuera del cluster (OPCIONAL):

```bash
# Crear una ruta para el servicio maestro (solo si es necesario)
oc expose svc postgres-master --port=5432

# O crear un servicio NodePort
oc expose svc postgres-master --type=NodePort --name=postgres-external
```

### 8. Monitoreo y mantenimiento

```bash
# Ver eventos
oc get events --sort-by='.lastTimestamp'

# Describir un pod
oc describe pod postgres-0

# Ver configuración del StatefulSet
oc describe statefulset postgres

# Escalar réplicas (agregar más nodos de lectura)
oc scale statefulset postgres --replicas=4
```

### 9. Acceso desde aplicaciones

En tus aplicaciones dentro del mismo proyecto, usa estos hostnames:

- **Para escrituras**: `postgres-master:5432`
- **Para lecturas**: `postgres-read:5432`

Ejemplo de cadena de conexión:
```
# Escritura
postgresql://postgres:mypassword@postgres-master:5432/mydb

# Lectura
postgresql://postgres:mypassword@postgres-read:5432/mydb
```

### 10. Limpieza (cuando ya no lo necesites)

```bash
# Eliminar todos los recursos
oc delete -f postgres-statefulset.yaml

# Eliminar PVCs (los datos se perderán)
oc delete pvc -l app=postgres

# Eliminar el proyecto completo
oc delete project postgres-db
```

## Solución de problemas comunes

### Los pods no inician por permisos

```bash
# Verificar SCC
oc describe scc anyuid

# Verificar que el ServiceAccount tenga permisos
oc get scc anyuid -o jsonpath='{.users}'
```

### Las réplicas no se sincronizan

```bash
# Verificar logs de la réplica
oc logs postgres-1 | grep -i replication

# Verificar conectividad de red
oc exec -it postgres-1 -- ping postgres-0.postgres-master
```

### Problemas con PersistentVolumes

```bash
# Ver el estado de PV y PVC
oc get pv,pvc

# Si usas almacenamiento dinámico, verifica el StorageClass
oc get storageclass
```

## Notas importantes para OpenShift

1. **Security Context Constraints**: OpenShift requiere permisos explícitos para ejecutar contenedores con usuarios específicos
2. **Routes vs Services**: OpenShift usa Routes en lugar de Ingress
3. **Image Streams**: Puedes usar Image Streams de OpenShift en lugar de imágenes directas
4. **Monitoreo**: OpenShift incluye Prometheus y Grafana integrados para monitoreo
5. **Backups**: Considera usar OpenShift Backup Operators para respaldos automáticos

## Configuración avanzada

### Usar StorageClass específico

Si necesitas un StorageClass particular:

```bash
# Ver StorageClasses disponibles
oc get storageclass

# Editar el YAML para especificar storageClassName
# En volumeClaimTemplates, agregar:
# storageClassName: <nombre-del-storage-class>
```

### Variables de entorno desde Secrets externos

```bash
# Crear secret desde línea de comandos
oc create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD='mi-password-seguro' \
  --from-literal=REPLICATION_PASSWORD='password-replicacion'
```
