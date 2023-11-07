Se plantea la necesidad de montar como volumen un bucket S3 dentro de un contenedor.

## Dockerfile
```Dockerfile
FROM debian:bullseye-slim

# Establece las variables de entorno para las credenciales de AWS
ENV AWS_ACCESS_KEY_ID 0
ENV AWS_SECRET_ACCESS_KEY 0
ENV AWS_BUCKET_NAME 0

# Actualiza el sistema y luego instala las dependencias necesarias
RUN apt-get update && apt-get upgrade -y && apt-get install -y s3fs

# Copia un script de entrada personalizado
COPY entrypoint.sh /entrypoint.sh

# Cambia los permisos del script
RUN chmod +x /entrypoint.sh

# Define el script de entrada
CMD ["/entrypoint.sh"]
```

## Docker

Destino:
> https://s3.console.aws.amazon.com/s3/buckets/prueba-bucket-docker-nico?region=us-east-2

```sh
docker build -t prueba-s3.x .

docker run -d -e AWS_ACCESS_KEY_ID=xxxxx -e AWS_SECRET_ACCESS_KEY=xxxxx --privileged -e AWS_BUCKET_NAME=prue
ba-bucket-docker-nico prueba-s3.x

docker exec -ti 859321ea387b /bin/bash
```
Una vez dentro del contenedor ejecutar:
```sh
ls /mnt/s3
```

## Openshift

Se necesita que el pod corra com root, para ello seguir la siguietne secuencia.

> Editar security context en el deploy.
```yaml
    securityContext:
      privileged: True
```

Levantar pod.
```sh
oc apply -f test-pod-deployment.yaml
```
Crear Service Acount.
```sh
oc create sa test-pod-sa
```
Aplicar los SCC a la SA.
```sh
oc amd policy add-scc-to-user priviliged -z test-pod-sa
```
Aplicar SA al deployment.
```sh
oc set sa deployment test-pod test-pod-sa
```
Verificar la regeneración del pod con `oc get pods`.
Ingresar al pod con `oc rsh NOMBRE-POD` y ejecutar comandos root, como por ejemplo `apt update`.

## Test pod deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-pod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-pod
  template:
    metadata:
      labels:
        app: test-pod
    spec:
      containers:
      - name: busybox-container
        image: busybox
          securityContext:
            privileged: True
        resources:
          limits:
            cpu: 50m
            memory: 100Mi
          requests:
            cpu: 10m
            memory: 50Mi
        command: ["sleep", "infinity"]
```

## Script
Script para levantar bucket S3 como FS.
```sh
#!/bin/sh

# Cargo credenciales AWS
echo "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" > /etc/passwd-s3fs
chmod 600 /etc/passwd-s3fs

# Crea un directorio de montaje
mkdir /mnt/s3

# Configura las credenciales para s3fs
echo "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" > /etc/passwd-s3fs

# Monta el sistema de archivos S3
s3fs $AWS_BUCKET_NAME /mnt/s3 -o passwd_file=/etc/passwd-s3fs

# Mantén el contenedor en ejecución
tail -f /dev/null
