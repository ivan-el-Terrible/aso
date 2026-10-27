PROYECTO DE AUTOMATIZACIÓN CON ANSIBLE

Repositorio: https://github.com/ivan-el-Terrible/aso

DESCRIPCIÓN
Ansible comprueba la conexión remota y mediante el rol usuarios crea el grupo
aso_laboratorio y las cuentas aso_alumno1 y aso_alumno2. Las cuentas no reciben
contraseñas propias ni permisos de administrador.

REQUISITOS
- Controlador con Ansible, cliente SSH y sshpass para acceso por contraseña.
- Servidor openSUSE con SSH activo, Python 3 y acceso de red.
- Usuario remoto con permisos para administrar usuarios y grupos.
- Contraseña de Vault, proporcionada por separado.

PASOS DE EJECUCIÓN
1. Entrar a la carpeta ansible del repositorio. Ejemplo:
   cd /aso/archivosAutomatizacion/ansible

2. Revisar y ajustar la IP del servidor:
   nano hosts

   Ejemplo:
   [servidores]
   servidor_lab ansible_host=192.168.69.133

   La IP es la utilizada en la prueba y puede cambiar. Verificar la identidad
   SSH del servidor durante la primera conexión.

3. Comprobar sintaxis:
   ansible-playbook site.yml --syntax-check

4. Ejecutar:
   ansible-playbook site.yml --ask-vault-pass

   Introducir la contraseña de Vault cuando se solicite. La contraseña de
   Vault permite descifrar el archivo; la contraseña SSH está dentro de él.

5. Comprobar el resultado. En una ejecución sin cambios pendientes:
   servidor_lab : ok=3 changed=0 unreachable=0 failed=0

   La primera ejecución puede indicar changed=2 porque crea el grupo y las
   cuentas; una segunda debería indicar changed=0 si no hay cambios pendientes.

6. Comprobar los usuarios:
   ansible servidores -m ansible.builtin.command -a "id aso_alumno1" --ask-vault-pass
   ansible servidores -m ansible.builtin.command -a "id aso_alumno2" --ask-vault-pass

   El módulo command puede indicar CHANGED aunque id solo consulta datos.

ARCHIVO CIFRADO CON ANSIBLE VAULT
Ruta: group_vars/servidores/vault.yml

El siguiente contenido fue proporcionado para la práctica. El archivo real
vault.yml debe existir en la ruta indicada: incluir este texto en el README
no hace que Ansible lo cargue. La contraseña de Vault se entrega por separado.
Para editar el archivo:
   ansible-vault edit group_vars/servidores/vault.yml

$ANSIBLE_VAULT;1.1;AES256
35383837633163643838346331663832653830323366303134326435323432303562363966653462
3633666133326134613465666261386266366661303037360a376132626364353536336133346133
30663266333132623063346633636537663832633332633264623262616363663133613434353732
3762343039323965320a616439373731373438663365633537646630633932363239383561623432
35643930326138323431613166373166346336633931633661613436313733316665626631623939
64343662656436393033396330643635623038346335373032356132396163356230316338333431
396136666237396131626338623961323963

NOTAS
- Las credenciales del grupo se aplican a todos sus destinos. Configurar
  variables por servidor si cada equipo utiliza credenciales diferentes.
- SSH puede usar una clave previamente cargada. Para verificar la contraseña,
  comprobar qué método de autenticación se usa en una conexión nueva.
- No subir claves privadas, contraseñas en texto claro ni la clave de Vault.

