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

4. FLUJO DE EJECUCIÓN PASO A PASO
---------------------------------
Para replicar y ejecutar exitosamente este despliegue en cualquier máquina virtual objetivo (como el entorno de prueba de Kate en `192.168.101.95`), se sigue este flujo operativo:

Paso 1: Clonar y actualizar el repositorio
Se asegura la obtención de la última versión del código desde la rama correspondiente en GitHub:
   git fetch origin
   git checkout tarea5-postgresql
   git pull origin tarea5-postgresql

Paso 2: Configurar el inventario
En el archivo `inventory/hosts.ini`, se especifican las credenciales de conexión SSH y de escalación de privilegios (`become`) para evitar bloqueos de autenticación:
   [destinos]
   192.168.101.95 ansible_user=kate ansible_password=mundolibre ansible_become_password=mundolibre

Paso 3: Lanzar el Playbook
Desde la raíz del proyecto, se ejecuta la automatización completa:
   ansible-playbook -i inventory/hosts.ini site.yml

El motor de Ansible procesará el `site.yml`, interpretará el rol `postgresql` y ejecutará de forma secuencial las 11 tareas modulares garantizando un estado final limpio, sin errores y con una ejecución totalmente idempotente (`failed=0`).

5. VERIFICACIÓN Y COMPROBACIÓN DEL ENTORNE
------------------------------------------
Una vez finalizada la ejecución, se valida el estado del servicio y los objetos creados conectándose de manera local a la máquina de destino:
- Estado del servicio:
  sudo systemctl status postgresql
- Listado de bases de datos (`mibase`):
  sudo su - postgres -c "psql -c '\l'"
- Listado de superusuarios (`alejandro`):
  sudo su - postgres -c "psql -c '\du'"
