# Guía Despliegue

Para el desarrollo de la plataforma de congresos se solicita una arquitectura de microservicios 
desplegada en la nube con pipelines de CI/CD, a continuación se explican los pasos que se siguieron 
para implementarla.

Actualmente, es solo una versión inicial simplificada, sin funcionalidad real, más bien como un 
prototipo inicial que permita hacer pruebas e ir estableciendo la infraestructura poco a poco.

## 1. Obtener y configurar las máquinas
Ya que el proyecto requiere ambientes de producción y desarrollo, 
se instanció una EC2 para cada ambiente, con las características:

| Componente:       | Valor:              |
|-------------------|---------------------|
| Sistema Operativo | Ubuntu Server 24.04 |
| Tipo de instancia | t3.medium           |
| vCPUs:            | 2                   |
| RAM:              | 4 GB                |
| Almacenamiento:   | 16 GB               |

Configurando reglas de seguridad (permitir acceso por ssh y http),
claves de acceso y direcciones IP elásticas para cada una.

### 1.1 Conectar a la maquina
Primero modificar permisos de las llaves:
```bash
chmod 400 "ruta/hacia/las/llaves.pem" 
```

Luego conectar por ssh
```bash
ssh -i /ruta/hacia/las/llaves.pem <usuario_remoto>@<ip_elastica>  
```
### 1.1 Actualizar e instalar Docker
```bash
sudo apt update && sudo apt upgrade -y
```
Instalar Docker y añadir usuario al grupo de docker
```bash
sudo apt install docker.io
sudo apt install docker-compose-v2
sudo usermod -aG docker $USER
```

## 2. Estructura de carpetas
Por ser la primera versión solo se buscó lograr funcionamiento de los 
servicios y el api-gateway con docker.

Se propuso y preparó la siguiente estructura de carpetas:
```bash
~/cnb_congress/
├── docker-compose.yml
├── api-gateway
│   └── nginx.conf
├── attendance-service
│   ├── Dockerfile
│   ├── attendance_service-0.0.1-SNAPSHOT.jar
│   ├── db.env
│   └── service.env
└── reports-service
    ├── Dockerfile
    ├── db.env
    ├── reports_service-0.0.1-SNAPSHOT.jar
    └── service.env
```
Teniendo en la raíz el docker-compose.yml y luego una carpeta para el 
api-gateway y una por cada servicio, conteniendo su Dockerfile, 
el jar del servicio y los .env para los contenedores del servicio y de la db.

El docker-compose.yml funciona con base en esta estructura.

## 3. docker-compose.yml
### API Gateway
Representa el punto de entrada, se enlaza al puerto 80 del host y redirige el tráfico a los servicios correspondientes, 
para ello se monta el volumen con la configuración
```yaml
api-gateway:
    image: nginx:alpine
    ports:
    - "80:80"
    volumes:
    - ./api-gateway/nginx.conf:/etc/nginx/nginx.conf:ro
```

### Microservicios (Spring Boot)
Cada servicio es una API REST con responsabilidades diferentes dentro de la lógica dle negocio
```yaml
reports-service:
    build: ./reports-service
    container_name: reports-service
    env_file:
      - ./reports-service/service.env
    environment:
      JAVA_OPTS: -Xms128m -Xmx256m
    depends_on:
      - reports-db
    restart: unless-stopped
```
Se construye la imagen localmente desde Dockerfile y se le pasa el archivo service.env para 
separar los parámetros de configuración, además se configura el restart para que el contenedor se reinicie
en caso de fallas, o que el Docker daemon o el host se reinicien.

### Bases de datos
Base de datos para cada servicio.
```yaml
reports-db:
    image: postgres:16
    container_name: reports-db
    env_file:
      - ./reports-service/db.env
    volumes:
      - reports-db-data:/var/lib/postgresql/data
    restart: unless-stopped
```
Se montan volúmenes específicos para cada base de datos para permitir persistencia independiente del contenedor,
así los datos sobreviven reinicios y reconstrucciones de los contenedores

## 3. nginx.conf (API Gateway)
```conf
    upstream reports-service {
        server reports-service:8080;
    }
```
Para definir las rutas del DNS interno de Docker usando el nombre del servicio definido en el `docker-compose.yml`
y así poder usarlas en la redirección a continuación.

```conf
    location /reports/ {
        proxy_pass http://reports-service;
    }
```
Para enrutar según el prefijo de la URL

## 4. Dockerfile de cada servicio 
Se encarga de configurar el contenedor responsable de ejecutar el servicio

```dockerfile
FROM eclipse-temurin:21-jre-jammy

WORKDIR /reports_service

COPY reports_service-0.0.1-SNAPSHOT.jar reports_service.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "reports_service.jar"]
```
Para optimizar, usa solo el jre, ya que tiene el jar, no necesita el jdk completo, solo lo copia
del host al contenedor y lo ejecuta.

Aunque de momento todos los servicios usan un dockerfile con la misma estructura, se decidió 
tener uno específico para cada uno para permitir modificaciones futuras sin afectar a los demás.

## 5. Archivos .env
### service.env
De forma general debe seguir la siguiente estructura, pero realmente dependerá del servicio específico:
```env
# APP
SERVER_PORT=8080
SERVICE_BASE_URL=reports

# DATABASE
DB_HOST=reports-db
DB_NAME=congress_reports_db
DB_USERNAME=postgres
DB_PASSWORD=postgres

# FRONTEND / BACKEND
FRONTEND_ORIGIN=http://localhost:3000
BACKEND_ORIGIN=http://localhost:8080

# JWT
JWT_SECRET=your_super_secret_key_here
JWT_REFRESH_SECRET=your_refresh_secret_key_here

# EMAIL
EMAIL_USERNAME=mail@mail.com
EMAIL_PASSWORD=email_app_password

# =========================
# AWS S3
# =========================
BUCKET_NAME=your_bucket_name
AWS_REGION=us-east-1
AWS_ACCESS_KEY=your_access_key
AWS_SECRET=your_secret_key
                       
```

### db.env
Para el contenedor de la base de datos se necesitan los siguientes valores:
```env
POSTGRES_DB: congress_reports_db
POSTGRES_USER: postgres
POSTGRES_PASSWORD: postgres                       
```

## 6. Ejecutar
De momento se ejecuta manualmente desde la raiz del proyecto con
```bash
docker compose up
```
Al inicio construirá las imagenes, al haber terminado ya se podrían consultar los endpoints básicos y health checks expuestos por los servicios

