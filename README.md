# Ansible para openSUSE Leap 16.0

Proyecto para organizar la configuración de servidores openSUSE Leap 16.0 con Ansible. El despliegue FTP ejecuta `preflight`, `packages`, `accounts`, `ftp`, `firewall` y `verify`; los demás servicios siguen como esqueleto.

## Estructura

```text
.
├── ansible.cfg                 # Opciones del controlador Ansible
├── site.yml                    # Valida la clave SSH y coordina el despliegue FTP
├── requirements.yml            # Colecciones para Zypper y firewalld
├── .gitignore                  # Exclusiones de Git que se definirán
├── README.md                   # Descripción y mapa del proyecto
├── inventory/
│   └── hosts.ini               # Grupo destinos, dirección y huella SSH del servidor
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

`inventory/hosts.ini` define el grupo `destinos` con la dirección y la huella SSH del servidor; ajusta ambos valores si cambió el destino. `site.yml` primero registra la clave SSH verificada en el controlador y después ejecuta `preflight`, `packages`, `accounts`, `ftp`, `firewall` y `verify` en ese orden y con privilegios de administrador. `group_vars/all.yml` define la cuenta `publica`; `group_vars/destinos.yml` proporciona el hash de su contraseña. `requirements.yml` declara `community.general` para Zypper y `ansible.posix` para firewalld.

Cada rol tiene `tasks/main.yml` como punto de entrada, `defaults/main.yml` para sus valores predeterminados y `meta/main.yml` para sus metadatos. Los roles de servicios incluyen `handlers/main.yml` para recargas o reinicios. Donde corresponde, `tasks/` separa instalación, configuración, usuarios, directorios, bases de datos, TLS o reglas; `templates/` contiene archivos Jinja2 `.j2`. Los seis roles del despliegue FTP ya tienen tareas funcionales.

## Roles

| Rol | Responsabilidad prevista | Relación con los demás |
| --- | --- | --- |
| `preflight` | Registrar en el controlador una clave SSH cuya huella coincida con el inventario; después validar openSUSE Leap 16.0 y Zypper en el nodo. | La tarea local se ejecuta antes de la primera conexión SSH; las validaciones remotas preceden a FTP. |
| `packages` | Detener PackageKit si está activo y centralizar la instalación de `vsftpd`, `firewalld` y `python3-firewall` con Zypper. | Evita que PackageKit bloquee Zypper y prepara los paquetes requeridos por `ftp` y `firewall`. |
| `accounts` | Crear usuarios locales y establecer sus contraseñas mediante hashes. | Crea `publica` con `/home/publica` y configura la contraseña solicitada antes de ejecutar `ftp`. |
| `apache_php` | Instalar y configurar Apache y PHP mediante sus plantillas. | Proporcionará el entorno web que necesita `phpmyadmin`. |
| `mariadb` | Configurar el servidor MariaDB, sus usuarios y bases de datos. | Proporcionará la base de datos administrada desde `phpmyadmin`. |
| `phpmyadmin` | Instalar y configurar la interfaz web de administración de MariaDB. | Dependerá de Apache/PHP y MariaDB. |
| `samba` | Configurar Samba, sus usuarios, directorios y la plantilla `smb.conf.j2`. | Podrá reutilizar cuentas y rutas de `accounts`. |
| `ftp` | Instalar `vsftpd.conf.j2`, preparar `/home/publica` y `/home/publica/archivos`, y habilitar el servicio. | Verifica la cuenta creada por `accounts` y prepara un directorio escribible por `publica`. |
| `postgresql` | Configurar PostgreSQL, usuarios, bases de datos y TLS. | Usará los paquetes centralizados; no contempla pgAdmin ni phpPgAdmin. |
| `firewall` | Permitir SSH, `21/TCP` y `30000–30100/TCP` en firewalld. | Aplica reglas permanentes después de configurar FTP y respeta el puerto SSH configurado. |
| `verify` | Comprobar servicios, escucha IPv4 local y permisos de los directorios FTP. | Se ejecuta al final, tras `firewall`. |

## Ejecución desde el master

Ejecuta los comandos Ansible en el **master** (`192.168.11.56`), dentro de este repositorio, con el usuario que administrará el despliegue. El nodo (`192.168.11.108`) solo necesita SSH y Python; no necesita tener Ansible instalado. Si entraste al nodo con `ssh`, vuelve al master con `exit`.

1. Comprueba que el master alcanza el puerto 22 del nodo y que `inventory/hosts.ini` contiene su IP, usuario SSH y huella ED25519. Obtén la huella en el nodo con `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub -E sha256` y compruébala por un canal de confianza antes de escribirla en el inventario. Si la IP o la clave del nodo cambian, actualiza esos datos antes de ejecutar el playbook. No desactives la comprobación de claves SSH para solucionar una discrepancia.
2. Desde el master, prepara las dependencias y comprueba la sintaxis:

   ```bash
   cd ~/ansible/aso
   ansible-galaxy collection install -r requirements.yml
   ansible-playbook --syntax-check site.yml
   ```

3. Para registrar **solo** la clave SSH verificada en `~/.ssh/known_hosts` del usuario actual, sin desplegar FTP:

   ```bash
   ansible-playbook site.yml --tags ssh_host_key
   ```

   Este paso es opcional antes del despliegue completo, que ya lo incluye. Después puedes probar la conexión de Ansible. Si entras al nodo con contraseña:

   ```bash
   ansible destinos -m ansible.builtin.ping --ask-pass
   ```

   Si usas una clave SSH para autenticarte, omite `--ask-pass`. `ping` debe responder con `pong`; verifica SSH y Python, no la instalación de FTP. La prueba con `ansible destinos -m ...` no ejecuta el registro automático: en un master nuevo, ejecuta primero el paso anterior. Usa el mismo usuario del master en ambos comandos, porque `known_hosts` es propio de cada usuario.

4. Para instalar y configurar FTP en el nodo, ejecuta el playbook **completo**:

   ```bash
   ansible-playbook site.yml --ask-pass
   ```

   `--ask-pass` solicita la contraseña del usuario SSH del inventario (`root` en este ejemplo). Si te autenticas con una clave SSH, usa `ansible-playbook site.yml`. Si configuras un usuario SSH distinto que necesita `sudo` con contraseña, añade `--ask-become-pass`. No uses solo `--tags ssh_host_key` para desplegar: esa etiqueta prepara SSH y no ejecuta los roles de FTP. El primer play del despliegue comprueba la huella y registra la clave; el segundo ejecuta los roles y termina con `verify`. Confirma en el resumen `failed=0` y `unreachable=0`. Una ejecución fallida o una comprobación de sintaxis correcta no demuestran que el despliegue haya terminado.

### Comprobar FTP desde el master

El rol `verify` confirma que `vsftpd` y `firewalld` están activos, que el puerto 21 acepta conexiones **locales** en el nodo y que los directorios tienen los permisos previstos. Para comprobar acceso y transferencia a través de la red, usa el cliente FTP del master:

```bash
printf 'Prueba de FTP\n' > /tmp/prueba-ftp.txt
ftp -p 192.168.11.108
```

Identifícate como `publica` con la contraseña configurada para esa cuenta. Dentro de `ftp`, ejecuta:

```text
ftp> pwd
ftp> ls
ftp> cd archivos
ftp> binary
ftp> put /tmp/prueba-ftp.txt
ftp> get prueba-ftp.txt /tmp/prueba-ftp-descargada.txt
ftp> bye
```

De vuelta en la terminal del master, comprueba que los bytes coincidan:

```bash
cmp /tmp/prueba-ftp.txt /tmp/prueba-ftp-descargada.txt && echo 'Subida y descarga correctas'
```

La raíz FTP no permite subir archivos: haz `cd archivos` antes de `put` o `mput`. Un `553 Could not create file` en la raíz es el comportamiento esperado. `mget` descarga archivos; `whoami` y `users` son comandos de la terminal, no del cliente FTP.

Si falla la prueba, distingue el punto de fallo: un tiempo de espera en SSH indica un problema de red o del puerto 22; un error de clave SSH exige revisar la huella; `Permission denied` después de conectar por SSH apunta a la autenticación; `530` en FTP apunta al usuario o contraseña; `553` dentro de `archivos` exige revisar los permisos del nodo; y si el inicio de sesión funciona pero `ls` o una transferencia se detienen, revisa los puertos pasivos `30000–30100/TCP` y la zona del firewall. Si el nodo cambió legítimamente su clave SSH, verifica la nueva huella antes de actualizar el inventario y la entrada anterior de `known_hosts`.

Si Zypper reintenta la instalación, el rol `packages` detiene previamente `packagekit.service` y `packagekit-background.service` cuando están activos. Si agota los reintentos, consulta el mensaje final de Zypper: otra operación de paquetes, un repositorio inaccesible o un error de descarga también pueden causar el fallo.

El rol `accounts` crea `publica` y `/home/publica`. Luego establece la contraseña solicitada mediante un hash SHA-512, equivalente al resultado de `passwd publica` y apto para reejecutar el playbook sin interacción. El texto claro no se guarda en el repositorio. El hash está en `group_vars/destinos.yml`; antes de publicar el repositorio o usarlo fuera de un laboratorio, cambia la contraseña y protege el hash con Ansible Vault.

La raíz FTP `/home/publica` tendrá propietario `root`, grupo `publica` y modo `0750`. El usuario podrá acceder y leer allí, pero deberá subir archivos dentro de `/home/publica/archivos`, con propietario y grupo `publica` y modo `0750`. En un cliente FTP, el directorio de subida se verá como `/archivos`.

La plantilla fija el rango pasivo TCP `30000–30100`. `firewall` abre ese rango, `21/TCP` y el puerto SSH en la zona `public`. Antes de ejecutar, confirma que la interfaz del nodo pertenece a esa zona; si no, cambia `ftp_firewall_zone` en `roles/firewall/defaults/main.yml` o sobrescríbela desde `group_vars/destinos.yml`. Las opciones `listen=NO` y `listen_ipv6=YES` proceden de la configuración solicitada. `verify` comprueba que el puerto 21 responde en IPv4 local, pero la conexión y transferencia desde otro equipo deben probarse tras el despliegue. Si existe NAT, configura la dirección pasiva anunciada en la plantilla y el reenvío de puertos.

La configuración usa FTP **sin cifrado**: las credenciales y los datos viajan en texto claro. Úsala solo en una red de confianza y configura TLS antes de exponerla a otras redes. El servicio acepta usuarios locales mediante PAM; `anonymous_enable=NO` desactiva el acceso anónimo.

El orden de los demás roles es el **diseño previsto**; todavía no se ejecutan desde `site.yml`.
