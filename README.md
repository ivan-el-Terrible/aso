# Ansible para openSUSE Leap 16.0

Proyecto para organizar la configuración de servidores openSUSE Leap 16.0 con Ansible. El rol `accounts` ya crea cuentas locales; los demás roles siguen como esqueleto.

## Estructura

```text
.
├── ansible.cfg                 # Opciones del controlador Ansible
├── site.yml                    # Playbook principal que coordinará los roles
├── requirements.yml            # Dependencias externas que se necesiten
├── .gitignore                  # Exclusiones de Git que se definirán
├── README.md                   # Descripción y mapa del proyecto
├── inventory/
│   └── hosts.ini               # Grupo destinos; aún sin hosts reales
├── group_vars/
│   ├── all.yml                 # Cuentas comunes a todos los hosts
│   └── destinos.yml            # Variables del futuro grupo «destinos» del inventario
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

`inventory/hosts.ini` define el grupo `destinos`; añade allí los hosts antes de ejecutar el playbook. `site.yml` ejecuta el rol `accounts` en ese grupo con privilegios de administrador. `group_vars/all.yml` define las cuentas comunes, y `group_vars/destinos.yml` queda disponible para variables específicas del grupo. `requirements.yml` reservará las colecciones o roles externos que hagan falta durante la implementación.

Cada rol tiene `tasks/main.yml` como punto de entrada, `defaults/main.yml` para sus valores predeterminados y `meta/main.yml` para sus metadatos. Los roles que configurarán servicios incluyen `handlers/main.yml` para futuras recargas o reinicios. Donde corresponde, `tasks/` contiene archivos separados para instalación, configuración, usuarios, directorios, bases de datos, TLS o reglas; `templates/` contiene archivos Jinja2 `.j2` destinados a generar configuraciones. Salvo `accounts`, estos archivos siguen como marcadores con comentarios.

## Roles

| Rol | Responsabilidad prevista | Relación con los demás |
| --- | --- | --- |
| `preflight` | Validar la versión del sistema operativo, la conectividad SSH y la disponibilidad de Zypper. | Debe ejecutarse antes de instalar o configurar servicios. |
| `packages` | Centralizar las listas y la instalación de paquetes. | Proveerá los paquetes requeridos por los roles de servicios. |
| `accounts` | Crear usuarios locales y sus directorios personales. | Crea `publica` con `/home/publica`; Samba y FTP podrán reutilizar esta cuenta. |
| `apache_php` | Instalar y configurar Apache y PHP mediante sus plantillas. | Proporcionará el entorno web que necesita `phpmyadmin`. |
| `mariadb` | Configurar el servidor MariaDB, sus usuarios y bases de datos. | Proporcionará la base de datos administrada desde `phpmyadmin`. |
| `phpmyadmin` | Instalar y configurar la interfaz web de administración de MariaDB. | Dependerá de Apache/PHP y MariaDB. |
| `samba` | Configurar Samba, sus usuarios, directorios y la plantilla `smb.conf.j2`. | Podrá reutilizar cuentas y rutas de `accounts`. |
| `ftp` | Configurar vsftpd, sus usuarios, directorios y la plantilla `vsftpd.conf.j2`. | Podrá reutilizar cuentas y rutas de `accounts`. |
| `postgresql` | Configurar PostgreSQL, usuarios, bases de datos y TLS. | Usará los paquetes centralizados; no contempla pgAdmin ni phpPgAdmin. |
| `firewall` | Definir reglas de acceso para los servicios desplegados. | Deberá reflejar los servicios y puertos que se habiliten en los otros roles. |
| `verify` | Comprobar el estado y la disponibilidad de los servicios. | Deberá ejecutarse al final, tras la configuración del firewall. |

El orden de los demás roles es el **diseño previsto**. `site.yml` ejecuta solo `accounts` por ahora. Para crear la cuenta, añade tus hosts al grupo `destinos` en `inventory/hosts.ini` y ejecuta `ansible-playbook -i inventory/hosts.ini site.yml`.
