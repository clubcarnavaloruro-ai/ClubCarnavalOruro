# Configuracion inicial del servidor

Servidor: `187.127.59.36`
Hostname: `srv1904721`
Sistema operativo: Ubuntu 24.04.4 LTS

## Acceso inicial

- Se inicio una conexion SSH como `root`.
- Se verifico la huella ED25519 del servidor y se acepto.
- La huella guardada es `SHA256:VKXRzxRx2JCz2dt6q5EbK2WMVNHbt7gmEouPvOe70kA`.

## Configuracion ejecutada

- Se ejecuto `apt update`.
- Se instalaron las actualizaciones disponibles.
- Se instalaron herramientas base:
  - `sudo`
  - `git`
  - `python3`
  - `python3-venv`
  - `python3-pip`
  - `nginx`
  - `ufw`
  - `fail2ban`
- Se instalo el kernel `6.8.0-138-generic`.

## Usuario para el deploy

Usuario: `guilder`
Contrasena: la define el administrador directamente en el terminal SSH; no se registra en este archivo.

El alta del usuario esta pendiente de completar porque el terminal solicita una contrasena valida. Cuando aparezca `New password:`, escribe una contrasena segura, repitela en `Retype new password:` y confirma los datos opcionales con Enter. No compartas la contrasena por chat ni la guardes en el repositorio.

## Pendiente

- Crear o confirmar `guilder` y agregarlo al grupo `sudo`.
- Eliminar `deploy` solamente después de confirmar que no ejecuta servicios:

```bash
id guilder >/dev/null 2>&1 || adduser --disabled-password --gecos "" guilder
usermod -aG sudo guilder
install -d -o guilder -g guilder /home/guilder/projects
systemctl list-units --type=service --all | grep -E 'club|django|gunicorn' || true
id deploy >/dev/null 2>&1 && userdel -r deploy || true
```

- Verificar el acceso con `su - guilder`.
- Reiniciar el servidor para cargar el nuevo kernel, cuando sea conveniente.
- Configurar la aplicacion Django, PostgreSQL, Nginx, HTTPS y el firewall con las reglas definitivas.