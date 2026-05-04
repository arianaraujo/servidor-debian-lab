# 🔎 Resolución de problemas

Guía de errores comunes encontrados durante el despliegue del servidor Debian 12 con BIND9 y Nginx.

---

## BIND9 / DNS

### El servicio `named` no arranca

```bash
sudo systemctl status named
sudo journalctl -xe -u named
```

**Causa más frecuente:** error de sintaxis en algún archivo de zona.

```bash
sudo named-checkconf
sudo named-checkzone lab.local /etc/bind/zones/db.lab.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1
```

Corregir el error indicado y reiniciar:

```bash
sudo systemctl restart named
```

---

### `dig` no devuelve respuesta desde el propio servidor

Verificar que el servicio está escuchando en el puerto 53:

```bash
ss -tulnp | grep :53
```

Si hay un conflicto con `systemd-resolved`:

```bash
sudo systemctl disable --now systemd-resolved
sudo systemctl restart named
```

---

### Los cambios en la zona no tienen efecto

BIND9 cachea las zonas. Cada vez que se modifica un archivo de zona hay que **incrementar el número de serial** (campo `Serial` del registro SOA) y recargar:

```bash
sudo systemctl reload named
# o bien:
sudo rndc reload
```

---

### La resolución inversa (PTR) no funciona

1. Comprobar que el archivo `db.192.168.1` existe y tiene el registro PTR correcto.
2. Verificar que `named.conf.local` declara la zona inversa exactamente como `1.168.192.in-addr.arpa`.
3. Probar:

```bash
dig @192.168.1.14 -x 192.168.1.14
```

---

## Nginx

### Nginx no arranca o da error

```bash
sudo nginx -t          # Comprobar sintaxis de configuración
sudo journalctl -xe -u nginx
```

---

### El sitio devuelve error 403 (Forbidden)

Causa: permisos incorrectos en el directorio raíz del sitio.

```bash
sudo chown -R www-data:www-data /var/www/lab.local
sudo chmod -R 755 /var/www/lab.local
```

---

### El sitio devuelve error 404 (Not Found)

Verificar que existe el archivo `index.html` en la ruta definida como `root` en el Virtual Host:

```bash
ls -la /var/www/lab.local/
```

---

### El Virtual Host no está activo

Comprobar que existe el enlace simbólico en `sites-enabled`:

```bash
ls -la /etc/nginx/sites-enabled/
```

Si no existe:

```bash
sudo ln -s /etc/nginx/sites-available/lab.local /etc/nginx/sites-enabled/
sudo systemctl reload nginx
```

---

## Cliente Windows

### `nslookup` no resuelve `www.lab.local`

1. Confirmar que el DNS preferido del adaptador de red es `192.168.1.14`.
2. Vaciar la caché DNS de Windows:

```powershell
ipconfig /flushdns
```

3. Probar la resolución forzando el servidor:

```powershell
nslookup www.lab.local 192.168.1.14
```

---

### El navegador no carga `http://www.lab.local`

1. Verificar que Nginx está activo: `sudo systemctl status nginx`
2. Verificar que el puerto 80 no está bloqueado por el firewall del servidor:

```bash
sudo ufw status
# Si está activo, permitir Nginx:
sudo ufw allow 'Nginx HTTP'
```

3. Probar conectividad básica desde Windows:

```powershell
ping 192.168.1.14
curl http://www.lab.local
```
