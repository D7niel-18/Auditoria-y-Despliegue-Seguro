# Despliegue de Nginx + Apache con Docker

**Despliegue de Aplicaciones Web** (2º DAW).

En esta práctica he montado con Docker Compose dos servidores web: **Nginx**, que hace de proxy inverso, y **Apache**, que es el servidor interno. También he añadido algunas medidas de seguridad y he comprobado que todo funciona con `curl`.

---

## 1. ¿Cómo funciona?

Hay dos contenedores conectados por una red privada de Docker llamada `red-corp`:

- **Nginx**: es el único que se puede usar desde fuera (puerto 80). Hace dos cosas:
  - Sirve directamente la web estática de `portal.empresa.local`.
  - Reenvía las peticiones de `app-gestion.empresa.local` al contenedor de Apache.
- **Apache**: es el servidor interno. No tiene puerto abierto al exterior, así que solo se puede llegar a él a través de Nginx. Se encarga de la zona `/privado` y de guardar en los logs la IP real del cliente.
- **Red `red-corp`**: es una red de tipo `bridge` que comunica los dos contenedores entre sí sin exponer Apache al host.

---

## 2. Estructura del proyecto

```text
Auditoria-y-Despliegue/
├── apache/
│   ├── conf/
│   │   └── httpd.conf         # Configuración de Apache
│   ├── htdocs/
│   │   └── index.html         # Página principal de Apache
│   └── logs/                  # Logs de Apache
├── nginx/
│   ├── conf.d/
│   │   └── default.conf       # Hosts virtuales, proxy inverso y cabeceras de seguridad
│   ├── html/
│   │   └── index.html         # Web estática que sirve Nginx
│   └── logs/                  # Logs de Nginx
├── .gitignore                 # Para no subir los logs a Git
├── docker-compose.yml         # Define los servicios y la red
└── README.md                  # Este documento
```

---

## 3. Medidas de seguridad

### A. Aislar Apache y usar lo mínimo necesario

- En el servicio `apache-app` **no he puesto la directiva `ports`**, así que su puerto no se publica en el host. Solo se llega a él desde la red interna con el nombre `apache-app`.
- Las carpetas de configuración y las de la web se montan como **solo lectura** (`:ro`). Así, aunque alguien consiguiera ejecutar código dentro del contenedor, no podría modificar esos ficheros.

### B. Ocultar información del servidor (OWASP A05)

Cuanta menos información damos, menos pistas tiene un atacante.

- **Nginx**: `server_tokens off;` oculta la versión de Nginx.
- **Apache**:
  - `ServerTokens Prod`: en las cabeceras solo sale `Server: Apache`, sin versión.
  - `ServerSignature Off`: quita la firma con la versión de las páginas de error.
  - `TraceEnable Off`: desactiva el método HTTP TRACE, que se puede usar en ataques XST (Cross-Site Tracing).

### C. Guardar la IP real del cliente (RGPD)

Si no se hace nada, en los logs de Apache siempre aparece la IP del contenedor de Nginx y no sabemos quién ha entrado de verdad.

Para solucionarlo, Nginx le pasa la IP del cliente a Apache en estas cabeceras:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Y Apache usa el módulo `mod_remoteip` para leerlas:

```apache
LoadModule remoteip_module modules/mod_remoteip.so
RemoteIPHeader X-Forwarded-For
RemoteIPInternalProxy 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
```

Con esto, el `%h` del `LogFormat` ya muestra la IP real del cliente en vez de la del proxy. Solo se confía en la cabecera cuando la petición viene de una IP privada (la red de Docker), para que nadie de fuera pueda falsearla.

### D. Control de acceso a `/privado`

Para entrar en `/privado` hay que mandar la cabecera `X-Auth-Token` con el valor correcto:

```apache
<Location "/privado">
    Require expr "%{HTTP:X-Auth-Token} == 'DAW2026'"
</Location>
```

Esto es una protección sencilla para la práctica. En un proyecto real no se dejaría la contraseña escrita en el fichero de configuración.

---

## 4. Cómo desplegarlo

### Lo que hace falta

- Windows 11 con Docker Desktop (con WSL2), o bien Linux/macOS.
- Git y `curl` instalados.

### Pasos

1. Levantar los contenedores:

   ```bash
   docker compose up -d
   ```

2. Comprobar que están funcionando:

   ```bash
   docker compose ps
   ```

---

## 5. Pruebas

Estos son los comandos `curl` que he usado para comprobar que todo funciona. Con `-H "Host: ..."` le indicamos a qué sitio queremos entrar (también se podría añadir los nombres al fichero `hosts`).

### 1. Web estática de Nginx (`portal.empresa.local`)

```bash
curl -I -H "Host: portal.empresa.local" http://localhost
```

**Resultado esperado:** `HTTP/1.1 200 OK`, con las cabeceras de seguridad y `Server: nginx` (sin versión).

### 2. Proxy inverso hacia Apache (`app-gestion.empresa.local`)

```bash
curl -I -H "Host: app-gestion.empresa.local" http://localhost
```

**Resultado esperado:** `HTTP/1.1 200 OK` y `Server: Apache`, lo que demuestra que la respuesta viene de Apache.

### 3. `/privado` sin token

```bash
curl -I -H "Host: app-gestion.empresa.local" http://localhost/privado/
```

**Resultado esperado:** `HTTP/1.1 403 Forbidden`.

### 4. `/privado` con el token correcto

```bash
curl -I -H "Host: app-gestion.empresa.local" -H "X-Auth-Token: DAW2026" http://localhost/privado/
```

**Resultado esperado:** `HTTP/1.1 200 OK`.

### 5. Comprobar la IP real en los logs

Para ver que Apache guarda la IP del cliente y no la del contenedor de Nginx, miramos el log de accesos:

```bash
cat apache/logs/access.log
```