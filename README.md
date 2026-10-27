# Ansible para openSUSE Leap 16.0

Este proyecto despliega FTP (vsftpd), un recurso Samba, LAMP (Apache, PHP, MariaDB y phpMyAdmin) y PostgreSQL en nodos openSUSE Leap 16.0. PostgreSQL no instala pgAdmin ni phpPgAdmin.

## Preparación

En el equipo controlador instala Ansible y OpenSSL. El nodo necesita SSH, Python, acceso a los repositorios de openSUSE y una cuenta SSH con privilegios de administrador.

1. Ajusta `inventory/hosts.ini`: dirección del nodo, `ansible_user` y `ssh_host_key_fingerprint`. Verifica la huella ED25519 directamente en el nodo con `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub -E sha256`. No guardes la contraseña SSH en el inventario.
2. Define el nombre, grupo y directorio personal de la cuenta en `group_vars/all.yml`. Usa los mismos valores en `roles/ftp/defaults/main.yml` (`ftp_user`, `ftp_home`, `ftp_upload_dir`) y `roles/samba/defaults/main.yml` (`samba_user`, `samba_group`, `samba_archives_path`). Ajusta `ftp_firewall_zone` en `roles/firewall/defaults/main.yml` según la interfaz del nodo.
3. Configura una sola contraseña cifrada para esa cuenta en `group_vars/destinos.yml`, bajo el nombre interno `publica_password`. FTP y Samba usan ese mismo valor; Ansible calcula un hash estable para Linux con `accounts_publica_password_salt`.
4. Ajusta `mariadb_database` y `mariadb_app_user` en `roles/mariadb/defaults/main.yml`. La contraseña SQL independiente es `mariadb_app_password` en `group_vars/destinos.yml`; también debe quedar cifrada con Vault. En este controlador puedes consultar su valor en `.mariadb-app-password`, excluido de Git. Si creas otra clave Vault, vuelve a cifrar ambas variables.
5. Ajusta `postgresql_db_name` y `postgresql_app_user` en `roles/postgresql/defaults/main.yml`. La configuración aportada por el compañero crea `mibase` y a `alejandro` como superusuario. PostgreSQL reutiliza la contraseña Vault de MariaDB mediante `postgresql_app_password`; el valor fijo del playbook anterior no se usa. Puedes definir una variable Vault independiente si quieres claves distintas.
6. PostgreSQL escucha en todas las interfaces y, por defecto, `postgresql_allowed_cidrs` contiene `0.0.0.0/0`: el firewall y `pg_hba.conf` permiten intentos desde cualquier IPv4. Las conexiones por socket local usan `trust`; las conexiones TCP exigen contraseña SCRAM, pero pueden usar TLS o ir sin cifrar. Cambia `postgresql_listen_addresses` y `postgresql_allowed_cidrs` si quieres limitar el acceso. El certificado inicial es autofirmado; puedes sustituir `server.crt` y `server.key` en `postgresql_data_dir` y volver a ejecutar el playbook.

Ansible lee la clave Vault desde `.ansible-vault-key`. Si recibiste el proyecto sin esa clave, crea una nueva y vuelve a cifrar la contraseña:

```bash
(umask 077; set -C; openssl rand -base64 48 > .ansible-vault-key)
ansible-vault encrypt_string --prompt --name publica_password
```

Sustituye el bloque `publica_password: !vault |` de `group_vars/destinos.yml` por el resultado. Cifra por separado la contraseña SQL con `ansible-vault encrypt_string --prompt --name mariadb_app_password` y sustituye su bloque. La clave local está excluida de Git: consérvala fuera del repositorio como respaldo y transfiérela de forma segura si usas otro controlador. Nunca la subas a GitHub.

## Ejecutar

Desde la raíz del proyecto, en el controlador:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook --syntax-check site.yml
ansible-playbook site.yml --ask-pass
```

`--ask-pass` solicita la contraseña SSH; omítelo si usas una clave SSH. Añade `--ask-become-pass` cuando el usuario remoto necesite contraseña para `sudo`.

El playbook registra la clave SSH del nodo tras comparar su huella con el inventario. Si quieres probar Ansible antes del despliegue, ejecuta:

```bash
ansible-playbook site.yml --tags ssh_host_key
ansible destinos -m ansible.builtin.ping --ask-pass
```

Al cambiar la contraseña cifrada de una cuenta Samba existente, ejecuta una vez `ansible-playbook site.yml --ask-pass -e samba_password_reset=true` para actualizarla también en Samba.

## Comprobar

El despliegue debe terminar con `failed=0` y `unreachable=0`. El rol `verify` revisa servicios, puertos locales, permisos, PHP, phpMyAdmin y la conexión SQL desde Python y PHP; comprueba además el acceso desde otro equipo. Sustituye `IP_DEL_NODO` y `USUARIO` por tus valores:

```bash
ftp -p IP_DEL_NODO
smbclient //IP_DEL_NODO/archives -U USUARIO
curl http://IP_DEL_NODO/aso-health.php
```

Para comprobar PostgreSQL en el nodo, ejecuta `sudo -u postgres psql -d postgres -c '\l'` y `sudo -u postgres psql -d postgres -c '\du'`. El rol `verify` comprueba el servicio, la base, la autenticación de su cuenta y el uso de TLS desde el propio nodo.

phpMyAdmin está en `http://IP_DEL_NODO/phpMyAdmin/` y acepta la cuenta SQL definida por `mariadb_app_user`; MariaDB permanece accesible solo desde el nodo. Apache sirve una página mínima de prueba PHP sin publicar detalles del entorno ni credenciales. En FTP, entra en `archivos` antes de subir archivos: la raíz del usuario no permite escritura. En Samba, el recurso `archives` apunta al directorio indicado por `samba_archives_path` y solo acepta la cuenta configurada. FTP y HTTP transmiten credenciales sin cifrar; úsalos únicamente en una red de confianza o configura TLS antes de exponerlos.

## Roles

| Rol | Estado y función |
| --- | --- |
| `preflight` | Activo: valida clave SSH, sistema y Zypper. |
| `packages` | Activo: instala paquetes FTP, Samba, firewall, LAMP y PostgreSQL. |
| `accounts` | Activo: crea la cuenta Linux y establece su contraseña. |
| `ftp` | Activo: configura vsftpd y directorios. |
| `samba` | Activo: configura el recurso, cuenta SMB y servicios. |
| `apache_php` | Activo: inicia Apache con PHP y publica una comprobación mínima. |
| `mariadb` | Activo: inicia MariaDB, crea la base y una cuenta SQL con contraseña Vault. |
| `phpmyadmin` | Activo: habilita el alias Apache del paquete phpMyAdmin. |
| `postgresql` | Activo: prepara TLS, configura accesos y crea una cuenta y una base de datos. |
| `firewall` | Activo: permite SSH, FTP, Samba, HTTP y las redes PostgreSQL autorizadas. |
| `verify` | Activo: comprueba servicios, permisos, LAMP y PostgreSQL con TLS. |
