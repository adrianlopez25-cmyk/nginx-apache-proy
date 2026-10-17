# Memoria Técnica: Despliegue de Arquitectura Web Multicapa y Hardening con Docker Compose

## 1. Introducción y Aprendizaje
En esta práctica he migrado una arquitectura web sencilla a un entorno multicapa desacoplado en contenedores. A lo largo del ejercicio he aprendido a:
- Separa la capa pública (*Proxy*) de la capa privada (*Backend*) para mejorar la seguridad perimetral.
- Ocultar firmas de servidor y aplicar cabeceras de protección (*Hardening* con Nginx).
- Manejar módulos de Apache como `mod_remoteip` para mantener la trazabilidad de la IP real del usuario (cumpliendo con la normativa RGPD).
- Restringir zonas sensibles del servidor web mediante la comprobación de cabeceras de autorización personalizadas (`X-Auth-Token`).

---

## 2. Diagrama de Arquitectura

Durante el desarrollo del ejercicio, la red y la comunicación entre contenedores han quedado estructuradas de la siguiente manera:

```text
               +----------------------------------+
               |           HOST MACHINE           |
               |          (Puerto 8080)           |
               +----------------+-----------------+
                                |
                                v
+-----------------------------------------------------------------+
| Red Privada Docker (frontend-net - Bridge)                      |
|                                                                 |
|  +-----------------------+           +-----------------------+  |
|  |   Nginx Edge Proxy    | proxy_80  | Apache Backend Server |  |
|  |  (nginx_edge_proxy)   |---------->| (apache_backend_app)  |  |
|  |                       |           |                       |  |
|  | - Hardening Headers   |           | - Aislado (sin ports) |  |
|  | - Ocultacion versión  |           | - IP Real mod_remoteip|  |
|  | - Inyeccion X-Forw-*  |           | - Token en /privado/  |  |
|  +-----------------------+           +-----------------------+  |
+-----------------------------------------------------------------+
```

## 3. Justificación Técnica y Configuración

### Aislamiento de Red (Docker Compose)
Para que el servidor Apache no sea accesible directamente desde el exterior, solo he publicado el puerto `8080` en el contenedor de Nginx. Apache se encuentra únicamente dentro de la red puente interna `frontend-net`, lo que obliga a que todo el tráfico pase obligatoriamente por el proxy.

### Hardening en la Frontera (Nginx)
En el archivo `default.conf` de Nginx he configurado:
- `server_tokens off;`: Para evitar mostrar la versión de Nginx en las cabeceras HTTP.
- **Inyección de cabeceras de seguridad**:
  - `X-Frame-Options: SAMEORIGIN` (evita ataques de Clickjacking).
  - `X-Content-Type-Options: nosniff` (previene el formateo incorrecto de archivos/MIME Sniffing).
  - `X-XSS-Protection: 1; mode=block` (filtro básico para XSS).

### Control de Acceso y RGPD (Apache)
- **Trazabilidad IP**: He habilitado el módulo `mod_remoteip` para que Apache no registre la IP interna del proxy de Docker, sino la IP real del cliente que realiza la consulta (`X-Forwarded-For`), asegurando así el cumplimiento del RGPD.
- **Zona Privada**: Mediante la regla `Require expr "%{HTTP:X-Auth-Token} == 'DAW2026'"` en el archivo `httpd.conf`, he bloqueado el acceso a la carpeta `/privado/`, respondiendo con un `403 Forbidden` si no se envía la cabecera adecuada y con un `200 OK` cuando esta es correcta.

---
## 4. Estructura empleada

```text
nginx-apache-proy-main/
├── .dockerignore
├── docker-compose.yml
├── README.md
├── apache/
│   ├── conf/
│   │   └── httpd.conf
│   ├── html/
│   │   ├── css/
│   │   │   └── style.css
│   │   ├── img/
│   │   │   └── guadalquivir-recorrido...
│   │   ├── privado/
│   │   │   └── index.html
│   │   └── index.html
│   └── logs/
├── capturas/
│   ├── acceso a la raiz.png
│   ├── auth-token correcto.png
│   └── localhost-privado.png
└── nginx/
    └── conf/
        └── default.conf
```
## 5 pruebas de verifiacion necesarias
### 1 despliegue de contenedores
docker compose up -d

<img width="415" height="110" alt="Captura de pantalla 2026-10-16 102014" src="https://github.com/user-attachments/assets/ec442e2b-c0db-4e93-8a63-ced613c15615" />

### 2 Comprobación del Proxy y Hardening
curl.exe -I http://localhost:8080/

<img width="389" height="222" alt="acceso a la raiz" src="https://github.com/user-attachments/assets/c702311a-308c-4538-81b8-4f94ea383e12" />

### 3 Comprobación de la Zona Privada sin Token
curl.exe -I http://localhost:8080/privado/

<img width="539" height="247" alt="image" src="https://github.com/user-attachments/assets/8db14f67-149b-4ae0-9fb9-e741173d5cf3" />

### 4 Comprobación de la Zona Privada con Token
curl.exe -i -H "X-Auth-Token: DAW2026" http://localhost:8080/privado/

<img width="445" height="206" alt="auth-token correcto" src="https://github.com/user-attachments/assets/007cddc2-6e80-4d29-9330-00463249cb8b" />

### 5 Verificación de Trazabilidad e IP Real en Logs
docker compose logs apache-backend

<img width="528" height="387" alt="image" src="https://github.com/user-attachments/assets/087e5169-64a9-4fd4-a55f-d4203b24d42e" />



