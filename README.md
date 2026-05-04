# 🖥️ Servidor Debian 12 — DNS (BIND9) + Nginx

![Debian](https://img.shields.io/badge/Debian-12%20Bookworm-A81D33?style=flat&logo=debian&logoColor=white)
![BIND9](https://img.shields.io/badge/DNS-BIND9-0078D7?style=flat&logo=cloudflare&logoColor=white)
![Nginx](https://img.shields.io/badge/Web-Nginx-009639?style=flat&logo=nginx&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VM-VirtualBox-183A61?style=flat&logo=virtualbox&logoColor=white)
![Estado](https://img.shields.io/badge/Estado-Funcional-brightgreen?style=flat)

> Proyecto de infraestructura en entorno de laboratorio. Implementación de un servidor Debian 12 con IP estática, servicio DNS autoritativo mediante BIND9 (zonas directa e inversa) y servidor web Nginx, todo desplegado sobre VirtualBox y verificado desde un cliente Windows.

---

## 📋 Tabla de contenidos

1. [Descripción del proyecto](#-descripción-del-proyecto)
2. [Entorno y tecnologías](#-entorno-y-tecnologías)
3. [Arquitectura de red](#-arquitectura-de-red)
4. [Configuración de IP estática](#-configuración-de-ip-estática)
5. [Servicio DNS con BIND9](#-servicio-dns-con-bind9)
6. [Servidor web con Nginx](#-servidor-web-con-nginx)
7. [Verificación desde cliente Windows](#-verificación-desde-cliente-windows)
8. [Estructura del repositorio](#-estructura-del-repositorio)
9. [Capturas de pantalla](#-capturas-de-pantalla)
10. [Resolución de problemas](#-resolución-de-problemas)

---

## 📌 Descripción del proyecto

Este proyecto documenta el despliegue completo de un servidor de red en entorno de laboratorio virtualizado. Los objetivos técnicos alcanzados son:

- **Asignación de IP estática** al servidor dentro de la red local.
- **Servicio DNS autoritativo** con BIND9: zona directa (`lab.local`) y zona inversa (PTR) para la red `192.168.1.0/24`.
- **Servidor web Nginx** configurado con un Virtual Host accesible mediante nombre DNS.
- **Resolución de nombres** verificada desde un cliente Windows apuntando al servidor DNS.

---

## 🛠️ Entorno y tecnologías

| Componente | Detalle |
|---|---|
| **Sistema Operativo** | Debian 12 Bookworm |
| **Plataforma de virtualización** | Oracle VirtualBox |
| **Modo de red VM** | Adaptador puente *(Bridge)* |
| **Servicio DNS** | BIND9 |
| **Servidor Web** | Nginx |
| **Cliente de pruebas** | Windows (mismo segmento de red) |

---

## 🌐 Arquitectura de red

```
┌────────────────────────────────────┐
│         Red local: 192.168.1.0/24  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Servidor (VirtualBox VM)    │  │
│  │  Hostname : srv-debian       │  │
│  │  IP       : 192.168.1.14     │  │
│  │  DNS      : BIND9            │  │
│  │  Web      : Nginx            │  │
│  └──────────────────────────────┘  │
│              ▲                     │
│              │ DNS + HTTP          │
│  ┌───────────┴──────────────────┐  │
│  │  Cliente Windows             │  │
│  │  DNS configurado: 192.168.1.14│  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
```

### Registros DNS configurados

| Tipo | Nombre | Valor |
|---|---|---|
| A | `ns1.lab.local` | `192.168.1.14` |
| A | `srv-debian.lab.local` | `192.168.1.14` |
| A | `www.lab.local` | `192.168.1.14` |
| NS | `lab.local` | `ns1.lab.local` |
| PTR | `14.1.168.192.in-addr.arpa` | `srv-debian.lab.local.` |

---

## 🔧 Configuración de IP estática

La IP estática se configura editando el archivo `/etc/network/interfaces`:

```bash
sudo nano /etc/network/interfaces
```

```ini
# Interfaz principal (adaptador puente de VirtualBox)
auto enp0s3
iface enp0s3 inet static
    address 192.168.1.14
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 192.168.1.14
```

Aplicar los cambios:

```bash
sudo systemctl restart networking
ip addr show enp0s3   # Verificar la IP asignada
```

---

## 🔍 Servicio DNS con BIND9

### Instalación

```bash
sudo apt update && sudo apt install -y bind9 bind9utils bind9-doc
sudo systemctl enable --now named
```

### Archivo de zonas — `named.conf.local`

Ruta: `/etc/bind/named.conf.local`

```bind
// Zona directa
zone "lab.local" {
    type master;
    file "/etc/bind/zones/db.lab.local";
};

// Zona inversa (PTR)
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

### Zona directa — `db.lab.local`

Ruta: `/etc/bind/zones/db.lab.local`

```dns
$TTL    604800
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Servidor de nombres
@               IN      NS      ns1.lab.local.

; Registros A
ns1             IN      A       192.168.1.14
srv-debian      IN      A       192.168.1.14
www             IN      A       192.168.1.14
```

### Zona inversa (PTR) — `db.192.168.1`

Ruta: `/etc/bind/zones/db.192.168.1`

```dns
$TTL    604800
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Servidor de nombres
@       IN      NS      ns1.lab.local.

; Registro PTR
14      IN      PTR     srv-debian.lab.local.
```

### Verificar la configuración

```bash
# Comprobar sintaxis de los archivos de zona
sudo named-checkconf
sudo named-checkzone lab.local /etc/bind/zones/db.lab.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1

# Reiniciar y comprobar estado
sudo systemctl restart named
sudo systemctl status named

# Prueba de resolución local
dig @192.168.1.14 www.lab.local
dig @192.168.1.14 -x 192.168.1.14
```

---

## 🌍 Servidor web con Nginx

### Instalación

```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

### Virtual Host — `www.lab.local`

Crear el archivo de configuración del sitio:

```bash
sudo nano /etc/nginx/sites-available/lab.local
```

```nginx
server {
    listen 80;
    server_name www.lab.local srv-debian.lab.local;

    root /var/www/lab.local;
    index index.html;

    access_log /var/log/nginx/lab.local-access.log;
    error_log  /var/log/nginx/lab.local-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Activar el sitio y crear el contenido web:

```bash
# Activar el Virtual Host
sudo ln -s /etc/nginx/sites-available/lab.local /etc/nginx/sites-enabled/

# Crear directorio raíz y página de inicio
sudo mkdir -p /var/www/lab.local
sudo nano /var/www/lab.local/index.html
```

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Lab.local - Servidor Debian</title>
</head>
<body>
    <h1>¡Servidor Nginx funcionando!</h1>
    <p>Hostname: srv-debian | IP: 192.168.1.14 | Dominio: lab.local</p>
</body>
</html>
```

```bash
# Verificar configuración y recargar
sudo nginx -t
sudo systemctl reload nginx
```

---

## ✅ Verificación desde cliente Windows

### 1. Configurar el DNS del cliente

En Windows, ir a:
`Panel de control → Centro de redes → Cambiar configuración del adaptador → IPv4 → DNS preferido: 192.168.1.14`

### 2. Pruebas de resolución DNS

Abrir `cmd` o `PowerShell`:

```powershell
# Resolución directa
nslookup www.lab.local
nslookup srv-debian.lab.local

# Resolución inversa (PTR)
nslookup 192.168.1.14

# Ping por nombre
ping www.lab.local
```

### 3. Prueba del servidor web

Abrir el navegador y acceder a:

```
http://www.lab.local
```

---

## 📁 Estructura del repositorio

```
debian-dns-nginx-server/
├── README.md
├── configs/
│   ├── network/
│   │   └── interfaces              # Configuración de IP estática
│   ├── dns/
│   │   ├── named.conf.local        # Declaración de zonas
│   │   ├── db.lab.local            # Zona directa
│   │   └── db.192.168.1            # Zona inversa (PTR)
│   └── nginx/
│       ├── lab.local               # Virtual Host de Nginx
│       └── index.html              # Página web de prueba
├── screenshots/
│   ├── 01-ip-estatica.png
│   ├── 02-bind9-status.png
│   ├── 03-dig-resolucion.png
│   ├── 04-nginx-status.png
│   └── 05-web-windows.png
└── docs/
    └── troubleshooting.md          # Errores comunes y soluciones
```

---

## 📸 Capturas de pantalla

| Paso | Descripción | Archivo |
|---|---|---|
| 1 | IP estática configurada (`ip addr show`) | `screenshots/01-ip-estatica.png` |
| 2 | Estado del servicio BIND9 (`systemctl status named`) | `screenshots/02-bind9-status.png` |
| 3 | Resolución DNS con `dig` desde el servidor | `screenshots/03-dig-resolucion.png` |
| 4 | Estado del servicio Nginx (`systemctl status nginx`) | `screenshots/04-nginx-status.png` |
| 5 | Acceso a `http://www.lab.local` desde Windows | `screenshots/05-web-windows.png` |

> 📂 Las capturas se encuentran en la carpeta [`screenshots/`](./screenshots/)

---

## 🔎 Resolución de problemas

Ver el archivo completo: [`docs/troubleshooting.md`](./docs/troubleshooting.md)

| Síntoma | Causa probable | Solución rápida |
|---|---|---|
| `named` no arranca | Error de sintaxis en zona | `sudo named-checkconf && sudo named-checkzone ...` |
| `dig` no resuelve | Servicio detenido o firewall | `sudo systemctl restart named` |
| Nginx devuelve 404 | Ruta del `root` incorrecta | Verificar `/var/www/lab.local/index.html` |
| Windows no resuelve | DNS no apunta al servidor | Comprobar DNS preferido en adaptador de red |
| PTR no responde | Zona inversa mal configurada | Revisar `db.192.168.1` y serial number |

---

## 👤 Autor

**Arian Araujo Arnaiz**  
📧 arianaraujoarnaiz@email.com  
🔗 [github.com/arianaraujo](https://github.com/tu-usuario)

---

## 📄 Licencia

Este proyecto está bajo la licencia [MIT](LICENSE).
