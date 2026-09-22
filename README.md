# docker1
documentacion de contenedores

# Práctica 1 - Entorno de Desarrollo con Docker

## 📋 Descripción del Proyecto

Este proyecto implementa un entorno de desarrollo profesional dockerizado que replica la estructura de XAMPP pero con cada componente en su propio contenedor independiente. Es el modelo estándar en entornos de desarrollo reales.

### Arquitectura de Contenedores

```
┌─────────────────────────────────────────────┐
│           RED: app-network                  │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │   web    │  │   db     │  │phpmyadmin│ │
│  │ (Apache  │  │ (MySQL)  │  │(Gestor)  │ │
│  │  + PHP)  │  │          │  │  de BD   │ │
│  │ :8080    │  │ :3306    │  │  :8081   │ │
│  └──────────┘  └──────────┘  └──────────┘ │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 🚀 Estructura del Proyecto

```
docker/
├── docker-compose.yml          # Configuración de los 3 servicios
├── php/
│   └── Dockerfile              # Imagen personalizada Apache + PHP 8.3
├── src/
│   └── index.php               # Aplicación PHP de demostración
├── .gitignore                  # Archivos a ignorar en Git
└── README.md                   # Este archivo
```

---

## 📦 Componentes

### 1. **web** - Servidor Apache + PHP 8.3
- **Imagen base**: `php:8.3-apache`
- **Extensiones instaladas**: 
  - `pdo` - Acceso a bases de datos
  - `pdo_mysql` - Driver MySQL para PDO
  - Módulo Apache `rewrite`
- **Puerto**: `8080`
- **Volumen**: `./src:/var/www/html` (código PHP sincronizado)

### 2. **db** - Base de Datos MySQL 8.0
- **Imagen**: `mysql:8.0`
- **Puerto**: `3306`
- **Base de datos creada**: `app_db`
- **Usuario root**: `rootpassword`
- **Volumen**: `db_data` (persistencia de datos)

### 3. **phpmyadmin** - Gestor Web de MySQL
- **Imagen**: `phpmyadmin:latest`
- **Puerto**: `8081`
- **Acceso**: http://localhost:8081
- **Credenciales**: 
  - Usuario: `root`
  - Contraseña: `rootpassword`

---

## 🔧 Proceso de Creación - Paso a Paso

### Paso 1: Crear la Estructura de Directorios

```bash
# Crear directorios principales
mkdir docker
cd docker
mkdir php
mkdir src
```

### Paso 2: Crear el Dockerfile para Apache + PHP

**Archivo**: `php/Dockerfile`

```dockerfile
FROM php:8.3-apache

# Instalar extensiones necesarias
RUN apt-get update && apt-get install -y \
    mariadb-client \
    && docker-php-ext-install pdo pdo_mysql \
    && a2enmod rewrite

WORKDIR /var/www/html

EXPOSE 80

CMD ["apache2-foreground"]
```

**¿Qué hace?**
- Descarga la imagen base `php:8.3-apache` de Docker Hub
- Actualiza los repositorios del sistema
- Instala `mariadb-client` (cliente MySQL)
- Instala extensiones PHP: `pdo` y `pdo_mysql` para acceder a MySQL
- Activa el módulo Apache `rewrite` (necesario para URLs amigables)
- Define el directorio de trabajo: `/var/www/html`
- Expone el puerto 80 (Apache)
- Inicia Apache en primer plano

### Paso 3: Crear el docker-compose.yml

**Archivo**: `docker-compose.yml`

```yaml
version: '3.8'

services:
  web:
    build:
      context: ./php
      dockerfile: Dockerfile
    container_name: web
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - db
    networks:
      - app-network
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: rootpassword
      DB_NAME: app_db

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    ports:
      - "8081:80"
    depends_on:
      - db
    networks:
      - app-network
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: rootpassword

  db:
    image: mysql:8.0
    container_name: db
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: app_db
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app-network

volumes:
  db_data:

networks:
  app-network:
    driver: bridge
```

**Explicación de cada sección**:

#### **web (Apache + PHP)**
- `build`: Construye la imagen usando el Dockerfile de `./php`
- `container_name: web`: Nombre único del contenedor
- `ports: "8080:80"`: Mapea puerto 8080 de tu PC al 80 del contenedor
- `volumes`: Sincroniza carpeta local `./src` con `/var/www/html` del contenedor
- `depends_on: db`: Espera a que `db` esté listo antes de iniciar
- `networks`: Conecta a la red `app-network` para comunicarse con otros servicios
- `environment`: Variables de entorno con credenciales de BD

#### **phpmyadmin**
- `image: phpmyadmin:latest`: Descarga la imagen oficial de phpMyAdmin
- `ports: "8081:80"`: Interfaz web en http://localhost:8081
- `PMA_HOST: db`: Se conecta al servicio llamado `db`
- Credenciales del usuario root

#### **db (MySQL)**
- `image: mysql:8.0`: Versión oficial de MySQL 8.0
- `MYSQL_ROOT_PASSWORD`: Contraseña del usuario root
- `MYSQL_DATABASE`: Base de datos creada automáticamente
- `volumes: db_data:/var/lib/mysql`: Persiste datos en volumen (no se pierden al parar contenedor)

#### **networks**
- `app-network`: Red interna donde se comunican los contenedores
- Los servicios se llaman por su nombre (ej: `db`, `web`)

### Paso 4: Crear la Aplicación PHP

**Archivo**: `src/index.php`

```php
<?php
// Conexión a MySQL con PDO
$host = getenv('DB_HOST') ?: 'db';
$user = getenv('DB_USER') ?: 'root';
$password = getenv('DB_PASSWORD') ?: 'rootpassword';
$db_name = getenv('DB_NAME') ?: 'app_db';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$db_name", $user, $password);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $connection = "✓ Conectado a MySQL";
} catch (PDOException $e) {
    $connection = "✗ Error: " . $e->getMessage();
}
?>

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Práctica 1 - Docker</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 40px;
            background-color: #f5f5f5;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: white;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
        }
        .status {
            display: flex;
            gap: 20px;
            margin: 20px 0;
        }
        .service {
            padding: 15px;
            border-radius: 5px;
            flex: 1;
        }
        .service.ok {
            background-color: #d4edda;
            border: 1px solid #c3e6cb;
        }
        .service h3 {
            margin: 0 0 10px 0;
        }
        .info-box {
            background: #e7f3ff;
            border-left: 4px solid #2196F3;
            padding: 15px;
            margin: 20px 0;
        }
        a {
            color: #2196F3;
            text-decoration: none;
        }
        a:hover {
            text-decoration: underline;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Práctica 1 - Entorno Docker</h1>
        
        <div class="status">
            <div class="service ok">
                <h3>🌐 Apache</h3>
                <p>Puerto: 8080</p>
                <p>Estado: ✓ Activo</p>
            </div>
            <div class="service ok">
                <h3>📊 phpMyAdmin</h3>
                <p>Puerto: 8081</p>
                <p><a href="http://localhost:8081" target="_blank">Acceder →</a></p>
            </div>
            <div class="service ok">
                <h3>🗄️ MySQL</h3>
                <p>Puerto: 3306</p>
                <p>Usuario: root</p>
            </div>
        </div>

        <div class="info-box">
            <strong>Estado de conexión MySQL:</strong><br>
            <?php echo $connection; ?>
        </div>

        <h2>ℹ️ Información</h2>
        <ul>
            <li><strong>Versión PHP:</strong> <?php echo phpversion(); ?></li>
            <li><strong>Servidor:</strong> <?php echo $_SERVER['SERVER_SOFTWARE']; ?></li>
            <li><strong>Hostname:</strong> <?php echo gethostname(); ?></li>
        </ul>

        <h2>🔗 Enlaces útiles</h2>
        <ul>
            <li><a href="http://localhost:8080" target="_blank">Apache (Puerto 8080)</a></li>
            <li><a href="http://localhost:8081" target="_blank">phpMyAdmin (Puerto 8081)</a></li>
        </ul>
    </div>
</body>
</html>
```

### Paso 5: Crear .gitignore

**Archivo**: `.gitignore`

```
# No versionamos datos de MySQL
db_data/

# Archivos de sistema
.DS_Store
Thumbs.db

# IDEs
.vscode/
.idea/

# Docker
__pycache__/
```

---

## 🏗️ Construcción y Ejecución

### 1. Construir las Imágenes

```bash
docker compose build
```

**¿Qué hace?**
- Lee el `Dockerfile` de `./php`
- Descarga la imagen base `php:8.3-apache`
- Instala las extensiones y dependencias
- Crea la imagen personalizada `default-web:latest`

### 2. Iniciar los Contenedores

```bash
docker compose up -d
```

**Flags**:
- `-d`: Ejecutar en segundo plano (detached mode)

**Salida esperada**:
```
Creating app-network ... done
Creating volume "default_db_data" ... done
Creating db ... done
Creating web ... done
Creating phpmyadmin ... done
```

### 3. Verificar que los Contenedores Están Corriendo

```bash
docker compose ps
```

**Salida**:
```
NAME        IMAGE               PORTS
web         default-web:latest  0.0.0.0:8080->80/tcp
phpmyadmin  phpmyadmin:latest   0.0.0.0:8081->80/tcp
db          mysql:8.0           0.0.0.0:3306->3306/tcp
```

### 4. Ver Logs

```bash
# Ver logs de todos los servicios
docker compose logs -f

# Ver logs de un servicio específico
docker compose logs -f web
docker compose logs -f db
```

---

## 🌐 Acceder a los Servicios

### Apache + PHP
```
URL: http://localhost:8080
```
- Verás la página `index.php`
- Mostrará versión de PHP, servidor y estado de conexión a MySQL

### phpMyAdmin
```
URL: http://localhost:8081
Usuario: root
Contraseña: rootpassword
```
- Gestiona las bases de datos MySQL desde la web
- Crea tablas, inserta datos, ejecuta consultas SQL

### MySQL (desde terminal)
```bash
docker compose exec db mysql -u root -prootpassword app_db
```
- `-u root`: usuario
- `-prootpassword`: contraseña (sin espacio)
- `app_db`: nombre de la base de datos

---

## 📝 Comandos Útiles

### Detener los Contenedores
```bash
docker compose down
```
- Detiene e **elimina** los contenedores
- **Mantiene** los datos en el volumen `db_data`
- La próxima ejecución de `up` reactivará los datos

### Detener sin Eliminar
```bash
docker compose stop
```
- Solo pausa los contenedores
- `docker compose start` los reinicia

### Eliminar TODO (incluyendo datos)
```bash
docker compose down -v
```
- `-v`: Elimina también los volúmenes
- **CUIDADO**: Borra todos los datos de MySQL

### Acceder a un Contenedor
```bash
# Shell del contenedor web
docker compose exec web bash

# Shell del contenedor db
docker compose exec db bash
```

### Reconstruir sin caché
```bash
docker compose build --no-cache
```
- Descarga todas las capas nuevamente
- Útil si hay cambios en el sistema base

### Ver consumo de recursos
```bash
docker stats
```

---

## 🔍 Troubleshooting

### Puerto ya en uso
```bash
# Cambiar puerto en docker-compose.yml
ports:
  - "8888:80"  # En lugar de 8080:80
```

### MySQL no responde
```bash
# Verificar logs de MySQL
docker compose logs db

# El contenedor puede estar iniciando, espera 10 segundos
```

### Cambios en PHP no se reflejan
```bash
# El volumen sincroniza automáticamente
# Si ves caché, limpia el navegador (Ctrl+Shift+Del)

# O reconstruye la imagen
docker compose build --no-cache
docker compose up -d
```

### Permisos de carpetas en Linux
```bash
# Si hay errores de permisos
sudo chown -R $USER:$USER src/
```

---

## 📊 Flujo de Funcionamiento

```
1. Usuario accede a http://localhost:8080
                    ↓
2. Apache (web) recibe la petición
                    ↓
3. PHP procesa index.php
                    ↓
4. PHP intenta conectar a MySQL (db)
   - Usa PDO con credenciales del environment
   - Usa hostname interno "db" (resuelto por Docker)
                    ↓
5. MySQL (db) responde
                    ↓
6. PHP genera HTML
                    ↓
7. Apache envía HTML al navegador
```

---

## 🎓 Conceptos Docker Aprendidos

### 1. **Imágenes vs Contenedores**
- **Imagen**: Plantilla (código ejecutable + dependencias)
- **Contenedor**: Instancia ejecutando la imagen

### 2. **Docker Compose**
- Orquesta múltiples contenedores
- Define relaciones entre servicios
- Persiste configuración en `docker-compose.yml`

### 3. **Volúmenes**
- Sincronización bidireccional de archivos (`./src:/var/www/html`)
- Persistencia de datos (`db_data:/var/lib/mysql`)

### 4. **Redes Docker**
- Contenedores en la misma red pueden comunicarse por nombre
- `web` llama a `db` simplemente usando hostname `db`

### 5. **Variables de Entorno**
- Se pasan a los contenedores en `environment:`
- Accesibles en PHP con `getenv()`

### 6. **Dependencias entre Servicios**
- `depends_on: db` asegura que `db` inicie antes que `web`

---

## 📋 Checklist de Verificación

- [ ] Los 3 contenedores están corriendo (`docker compose ps`)
- [ ] Apache responde en http://localhost:8080
- [ ] phpMyAdmin accesible en http://localhost:8081
- [ ] MySQL conecta desde PHP (muestra ✓ en la página)
- [ ] Carpeta `src/` sincroniza cambios a Apache
- [ ] Datos de MySQL persisten tras `docker compose down`
- [ ] Commits incremental en Git con mensajes descriptivos

---

## 🚀 Próximos Pasos

1. **Agregar base de datos real**: Crea tablas en phpMyAdmin
2. **Conectar a formularios**: Modifica `index.php` para insertar/leer datos
3. **Agregar más servicios**: Redis, Memcached, Elasticsearch
4. **Usar Docker Swarm o Kubernetes**: Para orquestar en producción

---

## 📚 Referencias

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [PHP Official Images](https://hub.docker.com/_/php)
- [MySQL Official Images](https://hub.docker.com/_/mysql)
- [phpMyAdmin Documentation](https://www.phpmyadmin.net/)

---

## 📄 Información de la Práctica

- **Asignatura**: Desarrollo Web en Entorno Servidor
- **Curso**: 2º DAW A
- **Código**: 0613
- **Práctica**: UD1 - Arranque del entorno de desarrollo con Docker
- **Tipo**: Trabajo Individual
- **Entrega**: URL del repositorio público en GitHub

---

**Última actualización**: Septiembre 2026
