DESPLIEGUE MODULAR DE POSTGRESQL CON ANSIBLE
============================================================

1. INTRODUCCIÓN Y ARQUITECTURA
------------------------------
Este proyecto implementa la automatización del despliegue y configuración de un servidor de bases de datos PostgreSQL sobre openSUSE mediante Ansible. La arquitectura se basa en un diseño modular que separa las responsabilidades en tareas independientes, facilitando el mantenimiento, la reutilización y la idempotencia del código.

2. ESTRUCTURA DE DIRECTORIOS Y ROLES
-----------------------------------
El repositorio se organiza bajo el siguiente esquema estándar de Ansible:

aso/
├── inventory/
│   └── hosts.ini          # Definición de nodos de destino y credenciales de acceso
├── roles/
│   └── postgresql/        # Rol principal modular
│       └── tasks/
│           ├── main.yml       # Archivo orquestador que incluye los submódulos
│           ├── install.yml    # Gestión de paquetes y dependencias del sistema
│           ├── configure.yml  # Configuración de red (postgresql.conf) y accesos (pg_hba.conf)
│           ├── tls.yml        # Configuración de cifrado y seguridad en tránsito
│           ├── users.yml      # Creación de superusuarios y roles avanzados
│           └── databases.yml  # Creación y asignación de bases de datos
├── site.yml               # Playbook global que aplica los roles a los hosts definidos
└── README.md              # Documentación del proyecto

3. DESCRIPCIÓN DE CADA COMPONENTE MODULAR
-----------------------------------------
- main.yml: Actúa como punto de entrada central dentro del rol `postgresql`, invocando secuencialmente a cada submenú de tareas utilizando la directiva `include_tasks`.
- install.yml: Se encarga de actualizar los repositorios si es necesario, instalar el servidor PostgreSQL y las bibliotecas cliente de PHP necesarias para la interacción con la base de datos.
- configure.yml: Modifica los parámetros críticos de la instancia de PostgreSQL, asegurando que escuche en las interfaces correctas (`listen_addresses`) y aplicando las reglas de autenticación en el archivo de control de acceso basada en host (`pg_hba.conf`).
- tls.yml: Establece las directivas necesarias para habilitar el cifrado y las conexiones seguras en el servicio de base de datos.
- users.yml: Automatiza la creación de roles administrativos y superusuarios (por ejemplo, el usuario `alejandro`) con sus respectivas credenciales y privilegios.
- databases.yml: Se encarga de inicializar bases de datos personalizadas (como `mibase`) y asociarlas directamente a sus respectivos propietarios.

4. FLUJO DE EJECUCIÓN PASO A PASO Y SEGURIDAD (ANSIBLE VAULT)
-------------------------------------------------------------
Para replicar y ejecutar exitosamente este despliegue protegiendo las credenciales sensibles, se utiliza Ansible Vault:

Paso 1: Clonar y actualizar el repositorio
   git fetch origin
   git checkout main
   git pull origin main

Paso 2: Configurar y cifrar el inventario
En el archivo `inventory/hosts.ini`, se definen los nodos y credenciales de conexión SSH y sudo:
   [destinos]
   192.168.101.95 ansible_user=kate ansible_password=<TU_PASSWORD> ansible_become_password=<TU_PASSWORD>

Por políticas de seguridad, las credenciales no se almacenan en texto plano, sino cifradas con AES256 mediante Ansible Vault:
   ansible-vault encrypt inventory/hosts.ini

(Si se desea editar el inventario posteriormente, se utiliza: `ansible-vault edit inventory/hosts.ini`)

Paso 3: Lanzar el Playbook
Desde la raíz del proyecto, se ejecuta la automatización solicitando la clave de descifrado en tiempo de ejecución:
   ansible-playbook -i inventory/hosts.ini site.yml --ask-vault-pass

El motor de Ansible descifrará las credenciales en memoria, procesará el `site.yml` y ejecutará las 11 tareas modulares garantizando una ejecución limpia e idempotente (`failed=0`).

5. VERIFICACIÓN Y COMPROBACIÓN DEL ENTORNE
------------------------------------------
Una vez finalizada la ejecución, se valida el estado del servicio y los objetos creados conectándose de manera local a la máquina de destino:
- Estado del servicio:
  sudo systemctl status postgresql
- Listado de bases de datos (`mibase`):
  sudo su - postgres -c "psql -c '\l'"
- Listado de superusuarios (`alejandro`):
  sudo su - postgres -c "psql -c '\du'"
