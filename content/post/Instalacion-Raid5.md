Title: Instalacion de un Raid 5
Date: 2020-10-14 17:00
Category: Servicios
Tags: Debian10, Raid, 
Authors: Sergio Ibáñez
Summary: En este apartado instalaremos un raid5 y veremos un poco su funcionamiento.

# **Gestión de almacenamiento de la información (Raid 5)**

Vamos a crear una máquina virtual con un sistema operativo Linux. En esta máquina, queremos crear un raid 5 de 2 GB, para ello vamos a utilizar discos virtuales de 1 GB. Crea un fichero Vagrantfile para crear la máquina.

## **Crea una raid llamado md5 con los discos que hemos conectado a la máquina. ¿Cuantos discos tienes que conectar? ¿Qué diferencia existe entre el RAID 5 y el RAID1?**

Lo primero que se necesitara para crear el raid sera instalar el paquete mdadm, una vez descargado se creara el Raid ejecutando mdadm –create /dev/md5 –level=5 raid-device=3 /dev/sdb /dev/sdc /dev/sdd como root. Con la opcion -level indicamos que tipo de raid se va ha crear y con la opcion raid-device indicamos los discos que formaran parte del raid.

<pre>
vagrant@Raid5:~$ sudo mdadm --create /dev/md5 --level=5 --raid-device=3 /dev/sdb /dev/sdc /dev/sdd
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md5 started.
</pre>

En este caso si lo que queremos en un Raid que ocupe 2Gb necesitaremos conectar al raid 3 discos de 1Gb.

La diferencia principal que hay entre Raid 1 y Raid 5 es la forma de almacenar la información que tiene cada uno. En Raid 1 son necesarios 2 discos como mínimo y se guarda la misma información en ambos discos, es decir, discos en espejo, de forma que si uno de los discos falla, la información no se pierde.

El Raid 5, al igual que el 1 ofrece tolerancia a fallos, pero sin utilizar los discos en espejo. El Raid 5 usa la paridad y con un mínimo de 3 discos lo que hace es segmentar y almacenar la misma cantidad de información en cada disco junto a su paridad, de forma que en caso de fallo se puede reconstruir la información gracias a los otros discos.

## **Comprueba las características del RAID. Comprueba el estado del RAID. ¿Qué capacidad tiene el RAID que hemos creado?**

Para comprobar las características del raid creado anteriormente se ejecuta mdadm –detail /dev/md5

<pre>
vagrant@Raid5:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Wed Sep 30 10:56:44 2020
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Wed Sep 30 10:56:55 2020
             State : clean 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Raid5:5  (local to host Raid5)
              UUID : 6d5bd423:423f95f9:d9f668a6:5032a7f6
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
</pre>

De esta forma, comprobamos que el raid creado esta en estado clean, es decir, vacío y comprobamos que el raid tiene capacidad de 2 GB.

## **Crea un volumen lógico (LVM) de 500Mb en el raid 5.**

Para crear este volumen logico, es necesario instalar primero el paquete lvm2 y una vez instalado usar los comandos, pvcreate, vgcreate y lvcreate.

Pvcreate /dev/md5 → Para crear el volumen físico donde crearemos los volúmenes lógicos posteriormente.
Vgcreate raid5 /dev/md5 → Para crear el grupo de volúmenes dentro del volumen físico creado.
Lvcreate raid5 -L 500M vlog → Para crear el volumen lógico de 500M dentro del grupo de volúmenes creado con el nombre vlog

<pre>
vagrant@Raid5:~$ sudo pvcreate /dev/md5 
  Physical volume "/dev/md5" successfully created.
vagrant@Raid5:~$ sudo vgcreate raid5 /dev/md5 
  Volume group "raid5" successfully created
vagrant@Raid5:~$ sudo lvcreate raid5 -L 500M -n vlog 
  Logical volume "storage" created.
</pre>

Una vez hecho esto se ejecuta un lsblk para comprobar que se han creado bien los volúmenes.

<pre>
NAME           MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda              8:0    0 19.8G  0 disk  
├─sda1           8:1    0 18.8G  0 part  /
├─sda2           8:2    0    1K  0 part  
└─sda5           8:5    0 1021M  0 part  [SWAP]
sdb              8:16   0    1G  0 disk  
└─md5            9:5    0    2G  0 raid5 
  └─raid5-vlog 253:0    0  500M  0 lvm   
sdc              8:32   0    1G  0 disk  
└─md5            9:5    0    2G  0 raid5 
  └─raid5-vlog 253:0    0  500M  0 lvm   
sdd              8:48   0    1G  0 disk  
└─md5            9:5    0    2G  0 raid5 
  └─raid5-vlog 253:0    0  500M  0 lvm   
</pre>

## **Formatea ese volumen con un sistema de archivo xfs.**

Antes de poder dar formato xfs a los volúmenes del sistema sera necesario instalar el paquete xfsprogs. Una vez descargado, se le da formato ejecutando el comando mkfs.xfs -f /dev/mapper/raid5-vlog como root.

<pre>
vagrant@Raid5:~$ sudo mkfs.xfs -f /dev/mapper/raid5-vlog
meta-data=/dev/mapper/raid5-vlog isize=512    agcount=8, agsize=16000 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=128000, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=896, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
</pre>

Una vez que hecho, se comprueba el formato ejecutando un lsblk -f.

<pre>
NAME           FSTYPE            LABEL   UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda                                                                                            
├─sda1         ext4                      983742b1-65a8-49d1-a148-a3865ea09e24     16.1G     7% /
├─sda2                                                                                         
└─sda5         swap                      04559374-06db-46f1-aa31-e7a4e6ec3286                  [SWAP]
sdb            linux_raid_member Raid5:5 6d5bd423-423f-95f9-d9f6-68a65032a7f6                  
└─md5          LVM2_member               gwEZ1e-HOjx-vkBk-R4Sw-ikfe-V3E0-86w1bM                
  └─raid5-vlog xfs                       055c4f5f-9d00-44b4-a3a5-ba5599b6edd4                  
sdc            linux_raid_member Raid5:5 6d5bd423-423f-95f9-d9f6-68a65032a7f6                  
└─md5          LVM2_member               gwEZ1e-HOjx-vkBk-R4Sw-ikfe-V3E0-86w1bM                
  └─raid5-vlog xfs                       055c4f5f-9d00-44b4-a3a5-ba5599b6edd4                  
sdd            linux_raid_member Raid5:5 6d5bd423-423f-95f9-d9f6-68a65032a7f6                  
└─md5          LVM2_member               gwEZ1e-HOjx-vkBk-R4Sw-ikfe-V3E0-86w1bM                
  └─raid5-vlog xfs                       055c4f5f-9d00-44b4-a3a5-ba5599b6edd4
</pre>

## **Monta el volumen en el directorio /mnt/raid5 y crea un fichero. ¿Qué tendríamos que hacer para que este punto de montaje sea permanente?**

Para montar el volumen lo primero que se hará sera crear un directorio en /mnt llamado raid5. Una vez creado se ejecutara un mount /dev/mapper/raid5-vlog /mnt/raid5 para que el volumen se monte en el directorio creado y después se creara un fichero de prueba ejecutando touch.

<pre>
vagrant@Raid5:~$ sudo mkdir /mnt/raid5
vagrant@Raid5:~$ sudo mount /dev/mapper/raid5-vlog /mnt/raid5
vagrant@Raid5:~$ sudo touch /mnt/raid5/prueba.md
</pre>

En el caso de querer que el punto de montaje fuese permanente, tendría que configurarse /etc/fstab de la siguiente forma:

<pre>
UUID=gwEZ1e-HOjx-vkBk-R4Sw-ikfe-V3E0-86w1bM /mnt/raid5  xfs     defautls        0       0
</pre>

De esta forma se indicara al sistema que al iniciarse el disco, o en este caso los volumenes con esa UUID se montaran de forma automatica en /mnt/raid5.

## **Marca un disco como estropeado. Muestra el estado del raid para comprobar que un disco falla. ¿Podemos acceder al fichero?**

Para marcarlo como estropeado se ejecuta mdadm con la opcion -f indicando el disco que se quiere estropear, en mi caso el sdb.

<pre>
vagrant@Raid5:~$ sudo mdadm /dev/md5 -f /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md5
vagrant@Raid5:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Wed Sep 30 10:56:44 2020
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Wed Sep 30 11:41:07 2020
             State : clean, degraded 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Raid5:5  (local to host Raid5)
              UUID : 6d5bd423:423f95f9:d9f668a6:5032a7f6
            Events : 20

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd

       0       8       16        -      faulty   /dev/sdb
</pre>

Cuando se compruebe que el volumen este marcado como estropeado, se hace un ls /mnt/md5 para comprobar que el fichero de prueba creado anteriormente sigue siendo accesible. En ese caso si es accesible.

<pre>
vagrant@Raid5:~$ ls /mnt/raid5/
prueba.md
</pre>

## **Una vez marcado el disco como estropeado, lo tenemos que retirar del raid.**

Ejecutamos mdadm /dev/md5 --remove sobre el disco anteriormente marcado como estropeado.

<pre>
vagrant@Raid5:~$ sudo mdadm /dev/md5 --remove /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md5
</pre>

Para comprobar que realmente ha sido retirado se vuelve a ejecutar sudo mdadm --detail.

<pre>
vagrant@Raid5:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Wed Sep 30 10:56:44 2020
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Wed Sep 30 11:44:42 2020
             State : clean, degraded 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Raid5:5  (local to host Raid5)
              UUID : 6d5bd423:423f95f9:d9f668a6:5032a7f6
            Events : 29

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
</pre>

## **Añádimos un disco nuevo al array y comprueba como se sincroniza con el anterior.**

Al contrario que lo hecho anteriormente, esta vez se utilizaría el comando –add sobre el disco retirado.

<pre>
vagrant@Raid5:~$ sudo mdadm /dev/md5 --add /dev/sdb
mdadm: added /dev/sdb
</pre>

Y se vuelve a comprobar que este todo bien con --detail.

<pre>
vagrant@Raid5:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Wed Sep 30 10:56:44 2020
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Wed Sep 30 11:45:48 2020
             State : clean, degraded, recovering 
    Active Devices : 2
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

    Rebuild Status : 49% complete

              Name : Raid5:5  (local to host Raid5)
              UUID : 6d5bd423:423f95f9:d9f668a6:5032a7f6
            Events : 38

    Number   Major   Minor   RaidDevice State
       4       8       16        0      spare rebuilding   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
</pre>

## **Añadiendo otro disco como reserva. Vuelve a simular el fallo de un disco y comprueba como automática se realiza la sincronización con el disco de reserva.**

Para ello se configura el fichero vagrantfile para que la maquina tenga un disco extra y una vez configurado en el raid ejecutamos mdadm --add sobre el disco nuevo, en este caso sde.

<pre>
vagrant@Raid5:~$ sudo mdadm /dev/md5 --add /dev/sde
mdadm: added /dev/sde
</pre>

<pre>
vagrant@Raid5:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Wed Sep 30 10:56:44 2020
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Sep 30 11:46:58 2020
             State : clean 
    Active Devices : 3
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Raid5:5  (local to host Raid5)
              UUID : 6d5bd423:423f95f9:d9f668a6:5032a7f6
            Events : 49

    Number   Major   Minor   RaidDevice State
       4       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd

       5       8       64        -      spare   /dev/sde
</pre>

Simulamos de nuevo el fallo y comprobamos como sde se sincroniza con los otros discos y empieza a funcionar.

<pre>
vagrant@Raid5:~$ sudo mdadm /dev/md5 -f /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md5
</pre>

<pre>
vagrant@Raid5:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Wed Sep 30 10:56:44 2020
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Sep 30 11:48:35 2020
             State : clean, degraded, recovering 
    Active Devices : 2
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

    Rebuild Status : 46% complete

              Name : Raid5:5  (local to host Raid5)
              UUID : 6d5bd423:423f95f9:d9f668a6:5032a7f6
            Events : 58

    Number   Major   Minor   RaidDevice State
       5       8       64        0      spare rebuilding   /dev/sde
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd

       4       8       16        -      faulty   /dev/sdb
</pre>

## **Redimensionando el volumen y el sistema de archivo de 500Mb al tamaño del raid.**

Para redimensionar el volumen se ejecuta el comando lvextend.

<pre>
vagrant@Raid5:~$ sudo lvextend /dev/mapper/raid5-vlog -L +1.5G -r
  Size of logical volume raid5/vlog changed from 500.00 MiB (125 extents) to <1.99 GiB (509 extents).
  Logical volume raid5/vlog successfully resized.
meta-data=/dev/mapper/raid5-vlog isize=512    agcount=8, agsize=16000 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=128000, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=896, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 128000 to 521216
You have new mail in /var/mail/vagrant
</pre>

Con las opciones -L y -r lo que se hace es indicar el tamaño que se quiere aumentar del volumen (-L) y que se realizara una redimension (-r). Se vuelve a comprobar que el volumen a sido redimensionado.

<pre>
vagrant@Raid5:~$ lsblk
NAME           MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda              8:0    0 19.8G  0 disk  
├─sda1           8:1    0 18.8G  0 part  /
├─sda2           8:2    0    1K  0 part  
└─sda5           8:5    0 1021M  0 part  [SWAP]
sdb              8:16   0    1G  0 disk  
└─md5            9:5    0    2G  0 raid5 
  └─raid5-vlog 253:0    0    2G  0 lvm   /mnt/raid5
sdc              8:32   0    1G  0 disk  
└─md5            9:5    0    2G  0 raid5 
  └─raid5-vlog 253:0    0    2G  0 lvm   /mnt/raid5
sdd              8:48   0    1G  0 disk  
└─md5            9:5    0    2G  0 raid5 
  └─raid5-vlog 253:0    0    2G  0 lvm   /mnt/raid5
sde              8:64   0    1G  0 disk  
└─md5            9:5    0    2G  0 raid5 
  └─raid5-vlog 253:0    0    2G  0 lvm   /mnt/raid5
</pre>
