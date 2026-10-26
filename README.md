# Ansible para openSUSE Leap 16.0

Proyecto para organizar la configuración de servidores openSUSE Leap 16.0 con Ansible. El despliegue FTP ejecuta `preflight`, `packages`, `accounts`, `ftp`, `firewall` y `verify`; los demás servicios siguen como esqueleto.

## Estructura

```text
.
├── ansible.cfg                 # Opciones del controlador Ansible
├── site.yml                    # Coordina cuenta, FTP, firewall y verificación
├── requirements.yml            # Colecciones para Zypper y firewalld
├── .gitignore                  # Exclusiones de Git que se definirán
├── README.md                   # Descripción y mapa del proyecto
├── inventory/
│   └── hosts.ini               # Grupo destinos y dirección del servidor FTP
├── group_vars/
│   ├── all.yml                 # Cuentas comunes a todos los hosts
│   └── destinos.yml            # Hash de contraseña del grupo «destinos»
└── roles/
    ├── preflight/              # Validaciones previas
    ├── packages/               # Paquetes centralizados
    ├── accounts/               # Cuentas y directorios
    ├── apache_php/             # Apache y PHP
    ├── mariadb/                # MariaDB
    ├── phpmyadmin/             # Administración web de MariaDB
    ├── samba/                  # Recursos compartidos Samba
    ├── ftp/                    # Servidor vsftpd
    ├── postgresql/             # PostgreSQL y TLS
    ├── firewall/               # Reglas de acceso
    └── verify/                 # Verificación de servicios
```

`inventory/hosts.ini` define el grupo `destinos` con la dirección del servidor del ejemplo; ajusta la IP si cambió el destino. `site.yml` ejecuta `preflight`, `packages`, `accounts`, `ftp`, `firewall` y `verify` en ese orden y con privilegios de administrador. `group_vars/all.yml` define la cuenta `publica`; `group_vars/destinos.yml` proporciona el hash de su contraseña. `requirements.yml` declara `community.general` para Zypper y `ansible.posix` para firewalld.

Cada rol tiene `tasks/main.yml` como punto de entrada, `defaults/main.yml` para sus valores predeterminados y `meta/main.yml` para sus metadatos. Los roles de servicios incluyen `handlers/main.yml` para recargas o reinicios. Donde corresponde, `tasks/` separa instalación, configuración, usuarios, directorios, bases de datos, TLS o reglas; `templates/` contiene archivos Jinja2 `.j2`. Los seis roles del despliegue FTP ya tienen tareas funcionales.

## Roles

| Rol | Responsabilidad prevista | Relación con los demás |
| --- | --- | --- |
| `preflight` | Validar openSUSE Leap 16.0 y la disponibilidad de Zypper. La conexión SSH se establece antes de ejecutar sus tareas. | Se ejecuta antes de instalar o configurar FTP. |
| `packages` | Centralizar la instalación de `vsftpd`, `firewalld` y `python3-firewall` y refrescar el catálogo de Zypper. | Prepara los paquetes requeridos por `ftp` y `firewall`. |
| `accounts` | Crear usuarios locales y establecer sus contraseñas mediante hashes. | Crea `publica` con `/home/publica` y configura la contraseña solicitada antes de ejecutar `ftp`. |
| `apache_php` | Instalar y configurar Apache y PHP mediante sus plantillas. | Proporcionará el entorno web que necesita `phpmyadmin`. |
| `mariadb` | Configurar el servidor MariaDB, sus usuarios y bases de datos. | Proporcionará la base de datos administrada desde `phpmyadmin`. |
| `phpmyadmin` | Instalar y configurar la interfaz web de administración de MariaDB. | Dependerá de Apache/PHP y MariaDB. |
| `samba` | Configurar Samba, sus usuarios, directorios y la plantilla `smb.conf.j2`. | Podrá reutilizar cuentas y rutas de `accounts`. |
| `ftp` | Instalar `vsftpd.conf.j2`, preparar `/home/publica` y `/home/publica/archivos`, y habilitar el servicio. | Verifica la cuenta creada por `accounts` y prepara un directorio escribible por `publica`. |
| `postgresql` | Configurar PostgreSQL, usuarios, bases de datos y TLS. | Usará los paquetes centralizados; no contempla pgAdmin ni phpPgAdmin. |
| `firewall` | Permitir SSH, `21/TCP` y `30000–30100/TCP` en firewalld. | Aplica reglas permanentes antes de iniciar el servicio y respeta el puerto SSH configurado. |
| `verify` | Comprobar servicios, escucha IPv4 local y permisos de los directorios FTP. | Se ejecuta al final, tras `firewall`. |

## Despliegue FTP

1. Confirma que la IP de `inventory/hosts.ini` corresponde al servidor openSUSE Leap 16.0 y que el usuario SSH tiene privilegios de administrador. Se recomienda usar clave SSH; si usas contraseña, solicita la entrada con `--ask-pass` y, si corresponde, `--ask-become-pass`.
2. Instala la colección en el controlador: `ansible-galaxy collection install -r requirements.yml`.
3. Ejecuta `ansible-playbook site.yml` desde este directorio.

El rol `accounts` crea `publica` y `/home/publica`. Luego establece la contraseña solicitada mediante un hash SHA-512, equivalente al resultado de `passwd publica` y apto para reejecutar el playbook sin interacción. El texto claro no se guarda en el repositorio. El hash está en `group_vars/destinos.yml`; antes de publicar el repositorio o usarlo fuera de un laboratorio, cambia la contraseña y protege el hash con Ansible Vault.

La raíz FTP `/home/publica` tendrá propietario `root`, grupo `publica` y modo `0750`. El usuario podrá acceder y leer allí, pero deberá subir archivos dentro de `/home/publica/archivos`, con propietario y grupo `publica` y modo `0750`. En un cliente FTP, el directorio de subida se verá como `/archivos`.

La plantilla fija el rango pasivo TCP `30000–30100`. `firewall` abre ese rango, `21/TCP` y el puerto SSH en la zona `public`. Antes de ejecutar, confirma que la interfaz del nodo pertenece a esa zona; si no, cambia `ftp_firewall_zone` en `roles/firewall/defaults/main.yml` o sobrescríbela desde `group_vars/destinos.yml`. Las opciones `listen=NO` y `listen_ipv6=YES` proceden de la configuración solicitada. `verify` comprueba que el puerto 21 responde en IPv4 local, pero la conexión y transferencia desde otro equipo deben probarse tras el despliegue. Si existe NAT, configura la dirección pasiva anunciada en la plantilla y el reenvío de puertos.

La configuración usa FTP **sin cifrado**: las credenciales y los datos viajan en texto claro. Úsala solo en una red de confianza y configura TLS antes de exponerla a otras redes. El servicio acepta usuarios locales mediante PAM; `anonymous_enable=NO` desactiva el acceso anónimo.

El orden de los demás roles es el **diseño previsto**; todavía no se ejecutan desde `site.yml`.
