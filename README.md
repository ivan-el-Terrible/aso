# Ansible para openSUSE Leap 16.0

Proyecto para organizar la configuración de servidores openSUSE Leap 16.0 con Ansible. Ya están implementados los roles `preflight`, `packages`, `accounts` y `ftp`; los demás roles siguen como esqueleto.

## Estructura

```text
.
├── ansible.cfg                 # Opciones del controlador Ansible
├── site.yml                    # Ejecuta preflight, packages, accounts y ftp
├── requirements.yml            # Colección community.general para Zypper
├── .gitignore                  # Exclusiones de Git que se definirán
├── README.md                   # Descripción y mapa del proyecto
├── inventory/
│   └── hosts.ini               # Grupo destinos y dirección del servidor FTP
├── group_vars/
│   ├── all.yml                 # Cuentas comunes a todos los hosts
│   └── destinos.yml            # Variables del grupo «destinos» del inventario
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

`inventory/hosts.ini` define el grupo `destinos` con la dirección del servidor del ejemplo; ajusta la IP si cambió el destino. `site.yml` ejecuta `preflight`, `packages`, `accounts` y `ftp` en ese grupo con privilegios de administrador. `group_vars/all.yml` define las cuentas comunes y `group_vars/destinos.yml` queda disponible para variables específicas del grupo. `requirements.yml` declara `community.general`, necesaria para instalar paquetes con Zypper.

Cada rol tiene `tasks/main.yml` como punto de entrada, `defaults/main.yml` para sus valores predeterminados y `meta/main.yml` para sus metadatos. Los roles de servicios incluyen `handlers/main.yml` para recargas o reinicios. Donde corresponde, `tasks/` separa instalación, configuración, usuarios, directorios, bases de datos, TLS o reglas; `templates/` contiene archivos Jinja2 `.j2`. Solo `preflight`, `packages`, `accounts` y `ftp` tienen tareas funcionales por ahora.

## Roles

| Rol | Responsabilidad prevista | Relación con los demás |
| --- | --- | --- |
| `preflight` | Validar openSUSE Leap 16.0 y la disponibilidad de Zypper. La conexión SSH se establece antes de ejecutar sus tareas. | Se ejecuta antes de instalar o configurar FTP. |
| `packages` | Centralizar las listas y la instalación de paquetes; actualmente instala `vsftpd` y refresca el catálogo de Zypper. | Prepara el paquete requerido por `ftp`. |
| `accounts` | Crear usuarios locales y sus directorios personales. | Crea `publica` con `/home/publica`; Samba y FTP podrán reutilizar esta cuenta. |
| `apache_php` | Instalar y configurar Apache y PHP mediante sus plantillas. | Proporcionará el entorno web que necesita `phpmyadmin`. |
| `mariadb` | Configurar el servidor MariaDB, sus usuarios y bases de datos. | Proporcionará la base de datos administrada desde `phpmyadmin`. |
| `phpmyadmin` | Instalar y configurar la interfaz web de administración de MariaDB. | Dependerá de Apache/PHP y MariaDB. |
| `samba` | Configurar Samba, sus usuarios, directorios y la plantilla `smb.conf.j2`. | Podrá reutilizar cuentas y rutas de `accounts`. |
| `ftp` | Instalar la configuración de `vsftpd.conf.j2`, habilitar el servicio y reiniciarlo cuando cambie la configuración. | Usa el paquete de `packages` y permite autenticar usuarios locales; la gestión específica de usuarios y directorios FTP sigue pendiente. |
| `postgresql` | Configurar PostgreSQL, usuarios, bases de datos y TLS. | Usará los paquetes centralizados; no contempla pgAdmin ni phpPgAdmin. |
| `firewall` | Definir reglas de acceso para los servicios desplegados. | Deberá reflejar los servicios y puertos que se habiliten en los otros roles. |
| `verify` | Comprobar el estado y la disponibilidad de los servicios. | Deberá ejecutarse al final, tras la configuración del firewall. |

## Despliegue FTP

1. Confirma que la IP de `inventory/hosts.ini` corresponde al servidor openSUSE Leap 16.0 y que el usuario SSH tiene privilegios de administrador. Se recomienda usar clave SSH; si usas contraseña, solicita la entrada con `--ask-pass` y, si corresponde, `--ask-become-pass`.
2. Instala la colección en el controlador: `ansible-galaxy collection install -r requirements.yml`.
3. Ejecuta `ansible-playbook site.yml` desde este directorio.

El rol `accounts` crea `publica` y `/home/publica`, pero **no establece una contraseña**. La plantilla de vsftpd permite autenticar usuarios locales mediante PAM; asigna credenciales válidas al usuario que deba entrar por FTP. Los archivos `roles/ftp/tasks/users.yml` y `directories.yml` siguen reservados para una gestión específica de FTP.

La plantilla fija el rango pasivo TCP `30000–30100`. Deben permitirse también el puerto `21/TCP` y ese rango en el firewall o los controles de red; el rol `firewall` aún no aplica reglas. Las opciones `listen=NO` y `listen_ipv6=YES` proceden de la configuración solicitada; comprueba la conectividad IPv4 del servidor si los clientes usan IPv4. Si existe NAT, configura la dirección pasiva anunciada en la plantilla antes de desplegar.

La configuración usa FTP **sin cifrado**: las credenciales y los datos viajan en texto claro. Úsala solo en una red de confianza y configura TLS antes de exponerla a otras redes. `allow_writeable_chroot=YES` permite un directorio personal escribible dentro del entorno restringido y conviene revisar los permisos de las cuentas.

El orden de los demás roles es el **diseño previsto**; todavía no se ejecutan desde `site.yml`.
