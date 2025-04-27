# lanwflowARODeploy
# Langflow OpenShift Deployment

Este repositorio contiene los archivos necesarios para desplegar Langflow en OpenShift, utilizando una base de datos PostgreSQL en Azure.

## Descripción

Langflow es una UI para LangChain que facilita la creación de flujos de trabajo de procesamiento de lenguaje natural. Este despliegue utiliza:

- Langflow como frontend y backend
- Base de datos PostgreSQL en Azure para almacenamiento persistente
- OpenShift como plataforma de orquestación de contenedores

## Requisitos previos

- Acceso a un clúster de OpenShift
- Cliente de OpenShift (`oc`) instalado
- Base de datos PostgreSQL en Azure (ya configurada)

## Configuración de la base de datos

Ya tenemos una instancia de PostgreSQL en Azure con los siguientes parámetros:

```
Host: microsweeper-30d70f389agbbpwd.postgres.database.azure.com
Usuario: myAdmin
Contraseña: PASSWORD
String de conexión: postgres://myAdmin%40microsweeper-30d70f389agbbpwd:adminUserGBB-30d70f389agbbpwd@microsweeper-30d70f389agbbpwd.postgres.database.azure.com/postgres?sslmode=require
```

## Instrucciones de despliegue

### 1. Iniciar sesión en OpenShift

```bash
oc login <URL-del-cluster>
```

### 2. Crear un nuevo proyecto

```bash
oc new-project langflow-project
```

### 3. Crear Secret para la conexión a la base de datos

```bash
oc create secret generic postgres-azure-secret \
  --from-literal=POSTGRES_USER=myAdmin \
  --from-literal=POSTGRES_PASSWORD=PASSWORD \
  --from-literal=POSTGRES_DB=postgres \
  --from-literal=POSTGRES_HOST=microsweeper-30d70f389agbbpwd.postgres.database.azure.com \
  --from-literal=LANGFLOW_DATABASE_URL="postgres://myAdmin%40microsweeper-30d70f389agbbpwd:adminUserGBB-30d70f389agbbpwd@microsweeper-30d70f389agbbpwd.postgres.database.azure.com/postgres?sslmode=require"
```

### 4. Crear PersistentVolumeClaim para Langflow

```bash
oc create -f langflow-pvc.yaml
```

### 5. Desplegar Langflow

```bash
oc create -f langflow-deployment.yaml
oc create -f langflow-service.yaml
oc create -f langflow-route.yaml
```

## Archivos de despliegue

### langflow-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: langflow-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### langflow-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: langflow
  labels:
    app: langflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: langflow
  template:
    metadata:
      labels:
        app: langflow
    spec:
      containers:
      - name: langflow
        image: langflowai/langflow:latest
        ports:
        - containerPort: 7860
        env:
        - name: LANGFLOW_DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: postgres-azure-secret
              key: LANGFLOW_DATABASE_URL
        - name: LANGFLOW_CONFIG_DIR
          value: "/app/langflow"
        volumeMounts:
        - name: langflow-data
          mountPath: /app/langflow
      volumes:
      - name: langflow-data
        persistentVolumeClaim:
          claimName: langflow-data-pvc
```

### langflow-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: langflow
spec:
  selector:
    app: langflow
  ports:
  - port: 7860
    targetPort: 7860
  type: ClusterIP
```

### langflow-route.yaml

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: langflow
spec:
  to:
    kind: Service
    name: langflow
  port:
    targetPort: 7860
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

## Para desplegar imagen personalizada con flujos precargados

Si deseas desplegar una imagen personalizada que contenga tus flujos de trabajo, crea un Dockerfile como el siguiente:

```Dockerfile
FROM langflowai/langflow:latest
RUN mkdir -p /app/flows
COPY ./flows/*.json /app/flows/
ENV LANGFLOW_LOAD_FLOWS_PATH=/app/flows
```

Luego construye y sube la imagen:

```bash
docker build -t miusuario/langflow-custom:1.0.0 .
docker push miusuario/langflow-custom:1.0.0
```

Y modifica el archivo `langflow-deployment.yaml` para usar tu imagen personalizada.

## Verificación

Para verificar que el despliegue se ha realizado correctamente:

```bash
# Ver la URL de la ruta
oc get route langflow

# Verificar que los pods están funcionando
oc get pods
```

## Solución de problemas

Si tienes problemas con la conexión a la base de datos, verifica:

1. Los logs del pod:
   ```bash
   oc logs $(oc get pods -l app=langflow -o name)
   ```

2. Que los secretos estén correctamente configurados:
   ```bash
   oc describe secret postgres-azure-secret
   ```

3. Que la ruta de conexión a PostgreSQL sea accesible desde OpenShift.
