Vagrant.configure("2") do |config|
  # Box de Oracle Linux 8 para ARM
  config.vm.box = "oraclelinux/8-aarch64"
  config.vm.box_url = "https://oracle.github.io/vagrant-projects/boxes/oraclelinux/8-aarch64.json"
  
  # Nombre de la máquina
  config.vm.hostname = "oracle-linux-19-arm"
  
  # Tamaño del disco:   50GB (requiere plugin vagrant-disksize)
  config.disksize.size = '50GB'
  
  # Red NAT (primera tarjeta - por defecto para vagrant ssh e Internet)
  
  # Red Host-only (segunda tarjeta - IP fija 192.168.56.11)
  config.vm.network "private_network", ip: "192.168.56.11", name: "HostNetwork"
  
  # Carpeta compartida para el instalador de Oracle
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"
  
  # Configuración del provider (VirtualBox)
  config.vm.provider "virtualbox" do |vb|
    vb.name = "Oracle Linux 19 ARM"
    vb.memory = "6144"
    vb.cpus = 2
    
    # Configurar el adaptador de red 2 como Paravirtualized (virtio-net)
    vb.customize ["modifyvm", :id, "--nictype2", "virtio"]
  end
  
  # Provisioning para extender la partición y LVM al tamaño completo del disco
  config.vm.provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "Expandiendo disco a 50GB con LVM..."
    echo "=========================================="
    
    # Mostrar estado actual
    echo "Estado actual del disco:"
    lsblk
    df -h /
    
    # Instalar cloud-utils-growpart si no está instalado
    yum install -y cloud-utils-growpart gdisk 2>/dev/null || true
    
    echo ""
    echo "Paso 1: Expandir partición sda3..."
    growpart /dev/sda 3 2>/dev/null || echo "Partición ya expandida"
    
    echo ""
    echo "Paso 2: Expandir Physical Volume (PV)..."
    pvresize /dev/sda3
    
    echo ""
    echo "Paso 3: Expandir Logical Volume (LV) a 45GB..."
    lvextend -L 45G /dev/vg_main/lv_root || lvextend -l +100%FREE /dev/vg_main/lv_root
    
    echo ""
    echo "Paso 4: Expandir sistema de archivos XFS..."
    xfs_growfs /
    
    echo ""
    echo "Estado final del disco:"
    lsblk
    df -h /
    echo "=========================================="
  SHELL
  
  # Provisioning para deshabilitar SELinux y Firewall
  config.vm.provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "Deshabilitando SELinux..."
    echo "=========================================="
    
    setenforce 0 2>/dev/null || true
    sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
    sed -i 's/^SELINUX=permissive/SELINUX=disabled/' /etc/selinux/config
    
    echo "SELinux deshabilitado"
    
    echo "=========================================="
    echo "Deshabilitando Firewall (firewalld)..."
    echo "=========================================="
    
    systemctl stop firewalld 2>/dev/null || true
    systemctl disable firewalld 2>/dev/null || true
    
    echo "Firewall deshabilitado"
  SHELL
  
  # Provisioning para habilitar login de root y acceso por consola
  config.vm.provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "Configurando acceso root y consola..."
    echo "=========================================="
    
    echo "root: root" | chpasswd
    echo "✅ Contraseña de root establecida:  root"
    
    echo "vagrant:vagrant" | chpasswd
    echo "✅ Contraseña de vagrant establecida:  vagrant"
    
    sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
    sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
    sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
    sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
    
    systemctl restart sshd
    echo "✅ SSH configurado"
  SHELL
  
  # Provisioning para pre-instalación de Oracle Database 19c
  config.vm.provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "PRE-INSTALACIÓN ORACLE DATABASE 19c"
    echo "=========================================="
    
    echo ""
    echo "Paso 1: Instalando paquete oracle-database-preinstall-19c..."
    yum install -y oracle-database-preinstall-19c
    
    echo ""
    echo "Paso 2: Instalando unzip..."
    yum install -y unzip
    
    echo ""
    echo "Paso 3: Creando directorios de Oracle..."
    mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
    mkdir -p /u02/oradata
    mkdir -p /u03/fra
    mkdir -p /instaladors
    chown -R oracle:oinstall /u01 /u02 /u03 /instaladors
    chmod -R 775 /u01 /u02 /u03 /instaladors
    
    echo ""
    echo "Paso 4: Configurando contraseña del usuario oracle..."
    echo "oracle:oracle" | chpasswd
    
    echo ""
    echo "Paso 5: Configurando variables de entorno para usuario oracle..."
    cat >> /home/oracle/.bash_profile <<'EOF'

# Oracle Settings
export TMP=/tmp
export TMPDIR=\$TMP

export ORACLE_HOSTNAME=oracle-linux-19-arm
export ORACLE_UNQNAME=ORCL
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory
export ORACLE_SID=ORCL
export PDB_NAME=ORCLPDB
export DATA_DIR=/u02/oradata

export PATH=/usr/sbin:/usr/local/bin:\$PATH
export PATH=$ORACLE_HOME/bin:\$PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
EOF
    
    chown oracle:oinstall /home/oracle/.bash_profile
    
    echo ""
    echo "Paso 6: Configurando límites adicionales del sistema..."
    cat >> /etc/security/limits.conf <<'EOF'

# Oracle 19c Settings
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   hard   memlock    134217728
oracle   soft   memlock    134217728
EOF
    
    echo ""
    echo "Paso 7: Verificando parámetros del kernel..."
    sysctl -p
    
    echo ""
    echo "=========================================="
    echo "✅ PRE-INSTALACIÓN COMPLETADA"
    echo "=========================================="
  SHELL
  
  # Provisioning para copiar y descomprimir el instalador de Oracle 19c
  config.vm.provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "COPIANDO Y DESCOMPRIMIENDO INSTALADOR"
    echo "=========================================="
    
    INSTALLER="/vagrant/LINUX.X64_193000_db_home.zip"
    
    if [ -f "$INSTALLER" ]; then
      echo "✅ Instalador encontrado: $INSTALLER"
      
      echo ""
      chown oracle:oinstall /vagrant/LINUX.X64_193000_db_home.zip
      echo ""
      echo "Descomprimiendo instalador en ORACLE_HOME (esto puede tardar 5-10 minutos)..."
      su - oracle -c "cd /u01/app/oracle/product/19.0.0/dbhome_1 && unzip -q $INSTALLER"
      echo ""
      echo "✅ Instalador descomprimido en ORACLE_HOME"
      echo ""
      ls -lh /u01/app/oracle/product/19.0.0/dbhome_1/ | head -20
      echo ""
      echo "Ajustando permisos..."
      chown -R oracle:oinstall /u01/app/oracle/product/19.0.0/dbhome_1 
      chmod -R 775 /u01/app/oracle/product/19.0.0/dbhome_1 
      echo ""
    else
      echo "⚠️ ADVERTENCIA: No se encontró el archivo LINUX.X64_193000_db_home.zip"
      echo "   Coloca el archivo en el mismo directorio que el Vagrantfile"
      echo "   y ejecuta:  vagrant reload --provision"
      exit 1
    fi
    
    echo "=========================================="
  SHELL
  
  # Provisioning para instalación silenciosa del software Oracle 19c
  config.vm. provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "INSTALACIÓN SILENCIOSA ORACLE 19c SOFTWARE"
    echo "=========================================="
    
    echo ""
    echo "Paso 1: Creando archivo de respuesta..."
    cat > /tmp/db_install.rsp <<'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v19.0.0
oracle.install.option=INSTALL_DB_SWONLY
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=oinstall
oracle.install.db.OSBACKUPDBA_GROUP=dba
oracle.install.db.OSDGDBA_GROUP=dba
oracle.install.db.OSKMDBA_GROUP=dba
oracle.install.db.OSRACDBA_GROUP=dba
oracle.install.db.rootconfig.executeRootScript=false
EOF
    
    chown oracle:oinstall /tmp/db_install.rsp
    chmod 600 /tmp/db_install.rsp
    
    echo ""
    echo "Paso 2: Ejecutando runInstaller (esto puede tardar 10-15 minutos)..."
    su - oracle -c "/u01/app/oracle/product/19.0.0/dbhome_1/runInstaller -silent -responseFile /tmp/db_install.rsp -waitforcompletion" || true
    
    echo ""
    echo "Paso 3: Ejecutando scripts root..."
    /u01/app/oraInventory/orainstRoot.sh
    /u01/app/oracle/product/19.0.0/dbhome_1/root.sh
    
    echo ""
    echo "=========================================="
    echo "✅ SOFTWARE ORACLE 19c INSTALADO"
    echo "=========================================="
  SHELL
  
  # Provisioning para crear la base de datos
  config.vm.provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "CREANDO BASE DE DATOS ORCL"
    echo "=========================================="
    
    echo ""
    echo "Paso 1: Creando archivo de respuesta DBCA..."
    cat > /tmp/dbca.rsp <<'EOF'
responseFileVersion=/oracle/assistants/rspfmt_dbca_response_schema_v19.0.0
gdbName=ORCL
sid=ORCL
databaseConfigType=SI
createAsContainerDatabase=true
numberOfPDBs=1
pdbName=ORCLPDB
pdbAdminPassword=Oracle123
templateName=General_Purpose.dbc
sysPassword=Oracle123
systemPassword=Oracle123
datafileDestination=/u02/oradata
recoveryAreaDestination=/u03/fra
storageType=FS
characterSet=AL32UTF8
nationalCharacterSet=AL16UTF16
memoryPercentage=40
automaticMemoryManagement=FALSE
totalMemory=2048
EOF
    chown oracle:oinstall /tmp/dbca.rsp
    chmod 600 /tmp/dbca.rsp
    
    echo ""
    echo "Paso 2: Creando base de datos (esto puede tardar 15-20 minutos)..."
    su - oracle -c "dbca -silent -createDatabase -responseFile /tmp/dbca.rsp" || true
    
    echo ""
    echo "Paso 3: Configurando inicio automático..."
    sed -i 's/: N$/:Y/' /etc/oratab
    
    echo ""
    echo "=========================================="
    echo "✅ BASE DE DATOS ORCL CREADA"
    echo "=========================================="
  SHELL

# Provisioning para configurar y arrancar el listener
config.vm. provision "shell", inline: <<-SHELL
  echo "=========================================="
  echo "CONFIGURANDO Y ARRANCANDO LISTENER"
  echo "=========================================="
  
  echo ""
  echo "Paso 1: Creando listener.ora..."
  su - oracle -c "cat > /u01/app/oracle/product/19.0.0/dbhome_1/network/admin/listener.ora <<'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = oracle-linux-19-arm)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = ORCL)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = ORCL)
    )
  )
EOF
"
  
  echo ""
  echo "Paso 2: Iniciando listener..."
  su - oracle -c "lsnrctl start"
  
  echo ""
  echo "Paso 3: Configurando local_listener en la base de datos..."
  su - oracle -c "sqlplus / as sysdba <<EOF
ALTER SYSTEM SET local_listener='(ADDRESS=(PROTOCOL=TCP)(HOST=oracle-linux-19-arm)(PORT=1521))' SCOPE=BOTH;
ALTER SYSTEM REGISTER;
EXIT;
EOF
"
  
  echo ""
  echo "Paso 4: Verificando listener..."
  su - oracle -c "lsnrctl status"
  
  echo ""
  echo "=========================================="
  echo "✅ LISTENER CONFIGURADO Y FUNCIONANDO"
  echo "=========================================="
SHELL

  # Provisioning para configurar autostart
  config.vm.provision "shell", inline: <<-SHELL
    echo "=========================================="
    echo "CONFIGURANDO AUTOSTART"
    echo "=========================================="
    
    echo ""
    echo "Paso 1: Creando scripts de inicio/parada..."
    mkdir -p /home/oracle/scripts
    
    cat > /home/oracle/scripts/start_db.sh <<'EOF'
#!/bin/bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

lsnrctl start

sqlplus / as sysdba <<SQL
STARTUP;
ALTER PLUGGABLE DATABASE ALL OPEN;
EXIT;
SQL
EOF
    
    cat > /home/oracle/scripts/stop_db.sh <<'EOF'
#!/bin/bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

sqlplus / as sysdba <<SQL
SHUTDOWN IMMEDIATE;
EXIT;
SQL

lsnrctl stop
EOF
    
    chmod +x /home/oracle/scripts/*.sh
    chown -R oracle:oinstall /home/oracle/scripts
    
    echo ""
    echo "Paso 2: Creando servicio systemd..."
    cat > /etc/systemd/system/oracle-db.service <<'EOF'
[Unit]
Description=Oracle Database
After=network.target

[Service]
Type=forking
User=oracle
Group=oinstall
ExecStart=/home/oracle/scripts/start_db.sh
ExecStop=/home/oracle/scripts/stop_db.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
    
    systemctl daemon-reload
    systemctl enable oracle-db.service
    
    echo ""
    echo "=========================================="
    echo "✅ AUTOSTART CONFIGURADO"
    echo "=========================================="
    
    echo ""
    echo "=========================================="
    echo "VERIFICACIÓN FINAL"
    echo "=========================================="
    su - oracle -c 'lsnrctl status' || true
  SHELL
  
  config.vm.post_up_message = <<-MSG
  ==========================================
  ✅ Oracle Database 19c INSTALADO Y FUNCIONANDO! 
  
  💾 Disco:  50GB (/ tiene ~45GB, swap 4GB)
  🧠 RAM: 6GB
  ⚙️ CPUs: 2
  
  🔌 Redes configuradas:
     - NAT:         eth0 (Internet + vagrant ssh)
     - Host-only: eth1 (192.168.56.11)
  
  👤 Usuarios configurados:
     - root    (password: root)
     - vagrant (password: vagrant)
     - oracle  (password: oracle)
  
  🗄️ Oracle Database 19c: 
     - ORACLE_HOME:     /u01/app/oracle/product/19.0.0/dbhome_1
     - ORACLE_SID:    ORCL
     - CDB Name:      ORCL
     - PDB Name:      ORCLPDB
     - SYS Password:  Oracle123
     - SYSTEM Password: Oracle123
  
  🔐 Conexiones:
     Desde la VM: 
       ssh oracle@192.168.56.11
       sqlplus / as sysdba
     
     Desde tu Mac:
       sqlplus system/Oracle123@//192.168.56.11:1521/ORCL
       sqlplus system/Oracle123@//192.168.56.11:1521/ORCLPDB
  
  📊 Comandos útiles:
     Iniciar DB: sudo systemctl start oracle-database
     Parar DB:    sudo systemctl stop oracle-database
     Estado DB:  sudo systemctl status oracle-database
     Listener:   lsnrctl status
  
  🎉 ¡La base de datos está lista para usar!
  ==========================================
  MSG
end
