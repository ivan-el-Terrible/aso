# Ansible para openSUSE Leap 16.0

Proyecto para organizar la configuración de servidores openSUSE Leap 16.0 con Ansible. Actualmente contiene **solo el esqueleto**: los playbooks, roles, variables y plantillas tienen comentarios descriptivos, pero todavía no definen tareas ejecutables ni servicios configurados.

## Estructura

```text
.
├── ansible.cfg                 # Opciones del controlador Ansible
├── site.yml                    # Playbook principal que coordinará los roles
├── requirements.yml            # Dependencias externas que se necesiten
├── .gitignore                  # Exclusiones de Git que se definirán
├── README.md                   # Descripción y mapa del proyecto
├── inventory/
│   └── hosts.ini               # Inventario operativo; aún sin hosts reales
├── group_vars/
│   ├── all.yml                 # Variables comunes a todos los hosts
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

`inventory/hosts.ini` será la fuente de hosts y grupos para `site.yml`. Las variables de `group_vars/all.yml` aplicarán a todos los hosts; `group_vars/destinos.yml` corresponderá al grupo `destinos` cuando se defina en el inventario. `site.yml` deberá seleccionar los hosts y coordinar los roles. `requirements.yml` reservará las colecciones o roles externos que hagan falta durante la implementación.

Cada rol tiene `tasks/main.yml` como punto de entrada, `defaults/main.yml` para sus valores predeterminados y `meta/main.yml` para sus metadatos. Los roles que configurarán servicios incluyen `handlers/main.yml` para futuras recargas o reinicios. Donde corresponde, `tasks/` contiene archivos separados para instalación, configuración, usuarios, directorios, bases de datos, TLS o reglas; `templates/` contiene archivos Jinja2 `.j2` destinados a generar configuraciones. Por ahora, todos estos archivos son marcadores con comentarios.

## Roles

| Rol | Responsabilidad prevista | Relación con los demás |
| --- | --- | --- |
| `preflight` | Validar la versión del sistema operativo, la conectividad SSH y la disponibilidad de Zypper. | Debe ejecutarse antes de instalar o configurar servicios. |
| `packages` | Centralizar las listas y la instalación de paquetes. | Proveerá los paquetes requeridos por los roles de servicios. |
| `accounts` | Gestionar usuarios, grupos y directorios comunes. | Preparará recursos que podrán utilizar Samba y FTP. |
| `apache_php` | Instalar y configurar Apache y PHP mediante sus plantillas. | Proporcionará el entorno web que necesita `phpmyadmin`. |
| `mariadb` | Configurar el servidor MariaDB, sus usuarios y bases de datos. | Proporcionará la base de datos administrada desde `phpmyadmin`. |
| `phpmyadmin` | Instalar y configurar la interfaz web de administración de MariaDB. | Dependerá de Apache/PHP y MariaDB. |
| `samba` | Configurar Samba, sus usuarios, directorios y la plantilla `smb.conf.j2`. | Podrá reutilizar cuentas y rutas de `accounts`. |
| `ftp` | Configurar vsftpd, sus usuarios, directorios y la plantilla `vsftpd.conf.j2`. | Podrá reutilizar cuentas y rutas de `accounts`. |
| `postgresql` | Configurar PostgreSQL, usuarios, bases de datos y TLS. | Usará los paquetes centralizados; no contempla pgAdmin ni phpPgAdmin. |
| `firewall` | Definir reglas de acceso para los servicios desplegados. | Deberá reflejar los servicios y puertos que se habiliten en los otros roles. |
| `verify` | Comprobar el estado y la disponibilidad de los servicios. | Deberá ejecutarse al final, tras la configuración del firewall. |

El orden descrito es el **diseño previsto**, aún no está implementado en `site.yml`. Antes de ejecutar Ansible será necesario definir los hosts, las variables y las tareas de cada etapa.
