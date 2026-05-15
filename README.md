Sistema de Gestión de Ventas y Despachos (Microservicios)

Este proyecto es una plataforma web para la gestión de órdenes de compra y asignación de despachos logísticos. Está construido bajo una arquitectura de **Microservicios**, utilizando contenedores Docker, y está diseñado para ser desplegado en la nube (AWS EC2) mediante un entorno de red segmentado y un Proxy Inverso.

Arquitectura y Tecnologías

El ecosistema está dividido en tres capas principales:

1.  Frontend (UI): Desarrollado en React.js (Vite). Actúa como la interfaz de usuario para listar las compras pendientes y gestionar el ciclo de vida de los despachos.
2.  Backend (Microservicios): Desarrollado en Java con Spring Boot.
    api-ventas: Gestiona las órdenes de compra.
    api-despachos: Gestiona los envíos, patentes de camiones y estados de entrega.
3.  Base de Datos: MySQL 8.0, centralizando la persistencia de datos (esquema tienda_db).

Infraestructura y DevOps
Contenedores: Docker y Docker Compose.
Servidor Web / Proxy Inverso: Nginx.
CI/CD: GitHub Actions / Docker Hub (Generación automática de imágenes).
Cloud: Despliegue en 2 instancias AWS EC2 (Subred Pública para Frontend y Subred Privada/Aislada para Backend).

---

Cómo funciona el sistema (Flujo de red)

Para garantizar la seguridad, el Frontend nunca expone la IP privada del Backend al cliente.
1. El usuario accede a la IP pública del Frontend.
2. Las peticiones de la API desde React se hacen mediante rutas relativas (ej. /api/v1/ventas).
3. El Nginx intercepta estas peticiones (Proxy Inverso) y las enruta a través de la red interna de AWS hacia la instancia del Backend en los puertos 8081 y 8082.

---

Guía de Despliegue (Cómo utilizar el proyecto)

Requisitos Previos
Dos servidores o instancias (ej. AWS EC2) con Docker y Docker Compose instalados.
Conexión de red entre ambas instancias (Security Groups en AWS configurados para permitir tráfico TCP en los puertos 8081 y 8082 desde el Frontend hacia el Backend).

Paso 1: Despliegue del Backend e Infraestructura de Datos
En la máquina designada para el Backend (ej. IP 10.0.3.153):

1. Crear el archivo docker-compose.yml:
   yaml
   version: '3.8'

   services:
     db:
       image: mysql:8.0
       container_name: mysql-db
       restart: always
       environment:
         MYSQL_ROOT_PASSWORD: root
         MYSQL_DATABASE: tienda_db
       ports:
         - "3306:3306"
       networks:
         - devops-network

     back-ventas:
       image: alexanderduoc/back-ventas:latest
       container_name: api-ventas
       restart: always
       ports:
         - "8081:8080"
       environment:
         SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/tienda_db?createDatabaseIfNotExist=true
         SPRING_DATASOURCE_USERNAME: root
         SPRING_DATASOURCE_PASSWORD: root
       networks:
         - devops-network

     back-despachos:
       image: alexanderduoc/back-despachos:latest
       container_name: api-despachos
       restart: always
       ports:
         - "8082:8080"
       environment:
         SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/tienda_db?createDatabaseIfNotExist=true
         SPRING_DATASOURCE_USERNAME: root
         SPRING_DATASOURCE_PASSWORD: root
       networks:
         - devops-network

   networks:
     devops-network:
       driver: bridge


Levantar los servicios con: docker-compose up -d

Paso 2: Despliegue del Frontend y Proxy Inverso

En la máquina designada para el Frontend (expuesta a Internet):
Asegurarse de que la configuración interna de Nginx (nginx.conf) en la imagen Docker contenga el enrutamiento correcto hacia la IP privada del Backend:

server {
    listen 8080; 

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html; 
    }

    location /api/v1/ventas {
        proxy_pass [http://10.0.3.153:8081](http://10.0.3.153:8081); 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/v1/despachos {
        proxy_pass [http://10.0.3.153:8082](http://10.0.3.153:8082); 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

Levantar el contenedor del Frontend mapeando el puerto 80 del servidor al 8080 de Nginx:

docker run -d --name front-despacho -p 80:8080 alexanderduoc/front-despacho:latest

Paso 3: Verificación
Acceder a la IP pública de la instancia Frontend desde un navegador web.

El sistema debería cargar la interfaz de usuario en React y comunicarse exitosamente con la base de datos a través del puente establecido por Nginx.
