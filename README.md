# virtualization-lab

Workshop "Containerizing and Deploying a Java Web Application" — una pequeña API REST en Spring Boot, empaquetada como imagen Docker, ejecutada en contenedores aislados, orquestada con Docker Compose junto a MongoDB, publicada en Docker Hub y desplegada en una instancia AWS EC2.

## Propósito

Explorar la virtualización (contenedores) como mecanismo arquitectónico de modularidad, aislamiento, portabilidad y despliegue: construir una app Java, empaquetarla como imagen Docker, correrla en contenedores aislados localmente, definir un entorno multi-contenedor con Docker Compose, publicarla en Docker Hub y desplegarla en una máquina virtual de AWS.

## Arquitectura

```
Client
  │ HTTP request
  ▼
EC2 virtual machine (Amazon Linux 2023, t3.micro)
  │
  ▼
Docker Engine
  │
  ▼
Java web application container (Spring Boot)
```

- **Spring Boot app:** expone `GET /greeting?name=X` → responde `Hello, X!`. Lee su puerto de la variable de entorno `PORT` (default `9000`).
- **Docker container:** empaqueta el `.jar` sobre la imagen base `amazoncorretto:21`.
- **Docker Compose:** define dos servicios — `web` (la app) y `db` (MongoDB `mongo:8`) — conectados por una red interna creada automáticamente por Compose. La app aún no persiste datos en Mongo; el servicio `db` existe para practicar cómo Compose maneja múltiples servicios, redes, mapeo de puertos y volúmenes.
- **EC2 + Security Group:** la instancia corre el contenedor de la app; el Security Group solo permite SSH (22) desde la IP del desarrollador y el puerto de la app (8080) desde cualquier origen.

## Stack

- Java 21 LTS
- Maven 3.9+
- Spring Boot 4.1.1
- Docker Desktop + Docker Compose v2
- Docker Hub
- Amazon Linux 2023 en AWS EC2 (AWS Academy Learner Lab)
- Imagen base `amazoncorretto:21`

## Estructura del proyecto

```
virtualization-lab/
├── pom.xml
├── Dockerfile
├── compose.yaml
├── src/main/java/co/edu/escuelaing/virtualizationlab/
│   ├── RestServiceApplication.java
│   └── HelloRestController.java
└── README.md
```

## Build y ejecución local

```bash
mvn clean package
java -jar target/virtualization-lab-1.0.0.jar
```

Verificar:
```
http://localhost:9000/greeting?name=Pedro
```
Respuesta esperada: `Hello, Pedro!`

*(Nota: el puerto por defecto es 9000. Si tu navegador bloquea algún puerto por seguridad, puedes sobreescribirlo con la variable de entorno `PORT` antes de correr el jar.)*

## Construcción y ejecución de la imagen Docker

```bash
docker build -t <tu-usuario-dockerhub>/virtualization-lab:1.0 .
docker images
```

Correr un contenedor:
```bash
docker run -d --name virtualization-lab-1 -e PORT=9000 -p 34000:9000 <tu-usuario-dockerhub>/virtualization-lab:1.0
docker ps
```

Probar: `http://localhost:34000/greeting?name=Container`

### Aislamiento de contenedores

Se corrieron 3 instancias del mismo image en puertos distintos, demostrando que cada contenedor es un proceso aislado e independiente:

```bash
docker run -d --name virtualization-lab-2 -p 34001:9000 <tu-usuario-dockerhub>/virtualization-lab:1.0
docker run -d --name virtualization-lab-3 -p 34002:9000 <tu-usuario-dockerhub>/virtualization-lab:1.0
```

Cada uno responde de forma independiente en `34000`, `34001` y `34002`.

## Docker Compose (web + MongoDB)

```bash
docker compose up -d --build
docker compose ps
docker compose logs web
docker compose logs db
```

Probar: `http://localhost:8087/greeting?name=Compose`

Conectarse a MongoDB dentro de su contenedor:
```bash
docker compose exec db mongosh
```
```javascript
show dbs
use workshop
db.messages.insertOne({ message: "Hello from Docker Compose" })
db.messages.find()
exit
```

Detener conservando los volúmenes: `docker compose down`
Detener eliminando los datos: `docker compose down -v`

La app se comunica con Mongo a través del hostname `db` (el nombre del servicio en `compose.yaml`); Compose crea automáticamente la red necesaria. Los volúmenes `mongodb` y `mongodb_config` preservan los datos independientemente del ciclo de vida del contenedor `db`.

## Publicación en Docker Hub

```bash
docker login
docker tag <tu-usuario-dockerhub>/virtualization-lab:1.0 <tu-usuario-dockerhub>/virtualization-lab:latest
docker push <tu-usuario-dockerhub>/virtualization-lab:1.0
docker push <tu-usuario-dockerhub>/virtualization-lab:latest
```

**Repositorio en Docker Hub:** https://hub.docker.com/r/keyserna/virtualization-lab

## Despliegue en AWS EC2

1. Instancia Amazon Linux 2023, tipo `t3.micro` (AWS Academy Learner Lab).
2. Security Group: SSH (22) solo desde la IP del desarrollador; puerto de la app (8080) abierto.
3. Conexión SSH e instalación de Docker:
   ```bash
   ssh -i <key>.pem ec2-user@<ec2-public-ip>
   sudo yum update -y
   sudo yum install -y docker
   sudo service docker start
   sudo usermod -a -G docker ec2-user
   ```
4. Pull y ejecución de la imagen:
   ```bash
   docker pull <tu-usuario-dockerhub>/virtualization-lab:1.0
   docker run -d --name virtualization-lab --restart unless-stopped \
     -e PORT=9000 -p 8080:9000 <tu-usuario-dockerhub>/virtualization-lab:1.0
   ```
5. Verificación:
   ```bash
   docker ps
   docker logs virtualization-lab
   ```

**URL pública del despliegue (verificar antes de entregar, la IP de instancias del Learner Lab cambia entre sesiones):**
```
http://<ec2-public-ip>:8080/greeting?name=AWS
```
Respuesta esperada: `Hello, AWS!`

> **Nota:** al terminar el workshop, la instancia EC2 se detiene/termina para no generar cargos adicionales en el Learner Lab.

## Modelo de despliegue y análisis de costos

### Diagrama

```
Client
  │ HTTP request
  ▼
EC2 virtual machine
  │
  ▼
Docker Engine
  │
  ▼
Java web application container
```

### Responsabilidad de cada capa

- **EC2 virtual machine:** cómputo, memoria, almacenamiento y red aislados, facturados por hora.
- **Docker container:** entorno de ejecución portable con la app y sus dependencias de runtime.
- **Java web application:** recibe peticiones HTTP y ejecuta la lógica de negocio.
- **Security group:** controla qué tráfico entrante llega a la máquina virtual.

### Supuestos

Región `us-east-1`. Tráfico HTTP plano. Tamaño promedio de request+response ≈ 2 KB. Sin alta disponibilidad (instancia única, sin balanceador de carga).

### Tabla de costos

| Escenario | Requests/mes | Instancia | # instancias | Horas/mes | EBS | Transferencia saliente | Costo EC2 | Costo EBS | Costo transferencia | **Total/mes** | **Costo/request** |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Pequeño | 10,000 | t3.micro | 1 | 730 (continuo) | 8 GB gp3 | ~20 MB | $7.59 | $0.64 | ~$0.00 | **$8.23** | **$0.000823** |
| Mediano | 100,000 | t3.micro | 1 | 730 (continuo) | 8 GB gp3 | ~200 MB | $7.59 | $0.64 | ~$0.02 | **$8.25** | **$0.0000825** |
| Grande | 1,000,000 | t3.small | 1 | 730 (continuo) | 20 GB gp3 | ~2 GB | $15.18 | $1.60 | $0.18 | **$16.96** | **$0.0000170** |

*(Estimado con la [AWS Pricing Calculator](https://calculator.aws), región US East (N. Virginia) — ver captura de pantalla adjunta.)*

**Estimated cost per request = monthly infrastructure cost / monthly requests**

### Discusión arquitectónica

**¿Por qué hay un costo base aunque lleguen pocas peticiones?**
EC2 cobra por el tiempo que la instancia está encendida (capacidad reservada de CPU, RAM y disco), no por petición atendida — es capacidad alquilada, exista o no tráfico.

**¿En qué nivel de carga el costo fijo deja de pesar tanto?**
El costo fijo (~$8/mes) se reparte entre más peticiones a medida que crece el tráfico. Entre el escenario pequeño y el grande, el costo por petición cae de $0.0008 a $0.000017 (casi 50x menos), porque el costo variable empieza a pesar más solo con volumen alto.

**¿Qué obligaría a pasar de una a varias instancias?**
Saturación de CPU/memoria bajo carga concurrente alta, necesidad de alta disponibilidad (una instancia es un punto único de falla), o necesidad de servir desde varias regiones por latencia.

**¿Qué servicios adicionales necesitaría producción?**
Load balancer (ALB) para distribuir tráfico y habilitar multi-AZ, base de datos gestionada (RDS/DocumentDB en vez de Mongo en contenedor), monitoreo (CloudWatch + alarmas), backups automatizados, y un registro de contenedores (ECR) en vez de Docker Hub.

**¿Sería mejor serverless para el escenario pequeño?**
Sí — 10,000 requests/mes es un tráfico muy bajo e intermitente (~0.004 req/seg en promedio). Con EC2 se pagan 730 horas encendidas aunque esté inactiva la mayoría del tiempo; con Lambda + API Gateway solo se paga el tiempo de cómputo realmente usado, y ese volumen cabría dentro de la capa gratuita de Lambda. El patrón de esta app (stateless, petición/respuesta simple) es ideal para serverless.

### Conclusión

Para el escenario pequeño, EC2 es técnicamente viable pero económicamente ineficiente frente a serverless, porque se paga capacidad ociosa. Para el escenario grande, EC2 se vuelve más eficiente por request porque el costo fijo se diluye entre mucho más tráfico — ahí un modelo de servidor dedicado (o con auto-scaling) empieza a tener sentido frente a la sobrecarga por invocación de un modelo serverless.

## Evidencia

*(Agregar aquí las capturas de pantalla: ejecución local, `docker ps` con los 3 contenedores aislados, `docker compose ps` con web+db, el repositorio en Docker Hub, la conexión SSH y `docker ps`/`docker logs` en EC2, la respuesta `Hello, AWS!`, y la estimación de la AWS Pricing Calculator.)*

## Extensión del framework propio

La extensión del framework de curso (sin Spring, con concurrencia y graceful shutdown) vive en un repositorio separado: **https://github.com/KeySerna/webframework-extension**
