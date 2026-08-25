# Guía de actualización en Hostinger

Esta guía describe cómo actualizar la aplicación **Club Carnaval Oruro** desde GitHub en el servidor de producción.

> No guardar contraseñas, claves SSH ni variables del archivo `.env` en este documento.

## Datos del despliegue

- Rama de producción: `deployhostinger`
- Proyecto en el servidor: `/home/guilder/projects/ClubCarnavalOruro`
- Entorno virtual: `/home/guilder/projects/ClubCarnavalOruro/venv`
- Servicio Gunicorn: `club-carnaval-oruro.service`
- Dominio: `https://carnavaldeoruro.club`

## Configuración única de HTTPS y URL canónica

La única URL pública del sitio debe ser:

```text
https://carnavaldeoruro.club/
```

Antes de aplicar esta configuración, comprueba en el panel DNS de Hostinger que los registros `A` de `carnavaldeoruro.club` y `www.carnavaldeoruro.club` apuntan a `187.127.59.36`. No dejes un reenvío web de Hostinger activo para `www`, porque Nginx debe realizar la redirección.

En el servidor, instala Certbot y prepara el directorio para la validación de Let's Encrypt:

```bash
sudo apt update
sudo apt install -y certbot
sudo mkdir -p /var/www/certbot
```

Crea temporalmente `/etc/nginx/sites-available/club-carnaval-oruro` con este contenido y recarga Nginx. Este bloque permite a Let's Encrypt validar ambos nombres mientras el resto del tráfico sigue llegando a la aplicación:

```nginx
server {
	listen 80;
	listen [::]:80;
	server_name carnavaldeoruro.club www.carnavaldeoruro.club;

	location /.well-known/acme-challenge/ {
		root /var/www/certbot;
	}

	location / {
		proxy_pass http://127.0.0.1:8001;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
	}
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo certbot certonly --webroot -w /var/www/certbot -d carnavaldeoruro.club -d www.carnavaldeoruro.club
```

Cuando Certbot confirme que creó el certificado, sustituye el archivo por esta configuración final. Conserva los bloques `location /static/` y `location /media/` si los usas actualmente, con las rutas reales del proyecto:

```nginx
upstream club_carnaval_oruro {
	server 127.0.0.1:8001;
}

server {
	listen 80;
	listen [::]:80;
	server_name carnavaldeoruro.club www.carnavaldeoruro.club;

	location /.well-known/acme-challenge/ {
		root /var/www/certbot;
	}

	location / {
		return 301 https://carnavaldeoruro.club$request_uri;
	}
}

server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	server_name www.carnavaldeoruro.club;

	ssl_certificate /etc/letsencrypt/live/carnavaldeoruro.club/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/carnavaldeoruro.club/privkey.pem;

	return 301 https://carnavaldeoruro.club$request_uri;
}

server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	server_name carnavaldeoruro.club;

	ssl_certificate /etc/letsencrypt/live/carnavaldeoruro.club/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/carnavaldeoruro.club/privkey.pem;
	client_max_body_size 50M;

	location /static/ {
		alias /home/guilder/projects/ClubCarnavalOruro/staticfiles/;
		expires 30d;
		add_header Cache-Control "public, immutable";
	}

	location /media/ {
		alias /home/guilder/projects/ClubCarnavalOruro/media/;
		expires 7d;
	}

	location / {
		proxy_pass http://club_carnaval_oruro;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto https;
		proxy_redirect off;
		proxy_connect_timeout 60s;
		proxy_send_timeout 60s;
		proxy_read_timeout 60s;
	}
}
```

Aplica y verifica las cuatro variantes. Deben terminar en `200` solamente para la URL canónica y en `301` para las otras tres:

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl enable --now certbot.timer
curl -I http://carnavaldeoruro.club/
curl -I http://www.carnavaldeoruro.club/
curl -I https://www.carnavaldeoruro.club/
curl -I https://carnavaldeoruro.club/
curl -I https://carnavaldeoruro.club/robots.txt
curl -I https://carnavaldeoruro.club/sitemap.xml
```

Las respuestas esperadas son `301` con `Location: https://carnavaldeoruro.club/...` para HTTP y `www`; `https://carnavaldeoruro.club/`, `robots.txt` y `sitemap.xml` deben responder `200`.

## 1. Verificar cambios localmente

Desde PowerShell, dentro del proyecto:

```powershell
git status --short --branch
git log -1 --oneline
python manage.py check
git diff --check
```

Confirma que los cambios estén publicados en GitHub:

```powershell
git push origin deployhostinger
git ls-remote origin deployhostinger
```

Guarda el identificador del commit remoto que aparece en la última línea.

## 2. Conectarse al servidor

Usa el usuario y la dirección del servidor configurados para el proyecto:

```powershell
ssh guilder@187.127.59.36
```

Escribe la contraseña SSH directamente en la terminal. No la pegues en documentos, comandos ni chats.

## 3. Guardar el commit actual

Antes de actualizar, entra al proyecto y registra el commit instalado:

```bash
cd /home/guilder/projects/ClubCarnavalOruro
git rev-parse --short HEAD
```

Este dato permite identificar la versión anterior si fuera necesario hacer rollback.

## 4. Descargar la versión de GitHub

```bash
cd /home/guilder/projects/ClubCarnavalOruro
git fetch origin deployhostinger
git reset --hard origin/deployhostinger
```

El `reset` solo debe ejecutarse en el servidor de producción, donde el código debe coincidir con GitHub.

## 5. Actualizar dependencias y Django

```bash
/home/guilder/projects/ClubCarnavalOruro/venv/bin/pip install -r requirements.txt
/home/guilder/projects/ClubCarnavalOruro/venv/bin/python manage.py migrate
```

Si no hay cambios de base de datos, Django mostrará `No migrations to apply.`

## 6. Publicar archivos estáticos

Este paso es obligatorio cuando cambian CSS, JavaScript o imágenes:

```bash
/home/guilder/projects/ClubCarnavalOruro/venv/bin/python manage.py collectstatic --noinput
```

## 7. Validar la aplicación

```bash
/home/guilder/projects/ClubCarnavalOruro/venv/bin/python manage.py check
```

La salida esperada es:

```text
System check identified no issues (0 silenced).
```

## 8. Reiniciar Gunicorn

```bash
sudo systemctl restart club-carnaval-oruro
sudo systemctl status club-carnaval-oruro --no-pager
```

El servicio debe mostrar:

```text
Active: active (running)
```

Es posible que el sistema solicite la contraseña `sudo`; escríbela directamente en la terminal.

## 9. Validar Nginx y recargarlo

```bash
sudo nginx -t
sudo systemctl reload nginx
```

La prueba correcta debe mostrar:

```text
syntax is ok
 test is successful
```

## 10. Verificar el sitio

```bash
curl -I https://carnavaldeoruro.club/
curl -I https://carnavaldeoruro.club/login/
```

Las dos rutas deben responder `HTTP 200 OK`.

Verifica también el commit instalado:

```bash
git rev-parse --short HEAD
```

Debe coincidir con el commit publicado en `origin/deployhostinger`.

## 11. Revisar errores si algo falla

Gunicorn:

```bash
journalctl -u club-carnaval-oruro -n 80 --no-pager
```

Estado del servicio:

```bash
sudo systemctl status club-carnaval-oruro --no-pager
```

Estado de Nginx:

```bash
sudo systemctl status nginx --no-pager
```

## 12. Cerrar la sesión

Cuando todas las verificaciones sean correctas:

```bash
exit
```

## Procedimiento resumido

```bash
cd /home/guilder/projects/ClubCarnavalOruro
git fetch origin deployhostinger
git reset --hard origin/deployhostinger
/home/guilder/projects/ClubCarnavalOruro/venv/bin/pip install -r requirements.txt
/home/guilder/projects/ClubCarnavalOruro/venv/bin/python manage.py migrate
/home/guilder/projects/ClubCarnavalOruro/venv/bin/python manage.py collectstatic --noinput
/home/guilder/projects/ClubCarnavalOruro/venv/bin/python manage.py check
sudo systemctl restart club-carnaval-oruro
sudo systemctl status club-carnaval-oruro --no-pager
sudo nginx -t
sudo systemctl reload nginx
curl -I https://carnavaldeoruro.club/
curl -I https://carnavaldeoruro.club/login/
git rev-parse --short HEAD
exit
```

## Resultado de la última actualización realizada

- Commit desplegado: `9172520`
- Migraciones: sin cambios pendientes.
- Archivos estáticos: recopilados correctamente.
- `manage.py check`: correcto.
- Gunicorn: activo.
- Nginx: configuración correcta y recargada.
- Página principal y login: `HTTP 200 OK`.
