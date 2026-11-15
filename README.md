# TP2 – Memoria swap y LVM

**Materia:** Arquitectura y Sistemas Operativos
**Carrera:** Tecnicatura Universitaria en Programación – UTN Avellaneda
**Alumno:** Facundo Ocampo

## Objetivo

Particionar discos adicionales, configurar memoria swap y administrar el almacenamiento con LVM (volúmenes físicos, grupos de volúmenes y volúmenes lógicos), formatear los volúmenes con ext4 y dejarlos montados de forma persistente.

## Entorno

- Virtualización: VirtualBox
- Sistema operativo: Kubuntu 24.04
- Usuario / hostname: `osboxes`
- Discos de la VM:

| Disco | Tamaño | Uso |
|-------|--------|-----|
| `sda` | 500 GiB | Sistema (`/`) |
| `sdb` | 5 GiB | Disco adicional (LVM) |
| `sdc` | 2 GiB | Disco adicional (swap + LVM) |

## 1. Configuración de discos adicionales

Los discos extra se definen en el `Vagrantfile` de la guía del TP:

```ruby
config.vm.disk :disk, size: "5GB", name: "extra_storage"
config.vm.disk :disk, size: "2GB", name: "extra_storage2"
```

En la VM se verifican con `lsblk`.

## 2. Particionado

### Disco `/dev/sdc` (2 GiB)

Con `sudo fdisk /dev/sdc` se crearon 3 particiones primarias, una extendida (contenedor) y 2 particiones lógicas dentro de ella:

| Partición | Tipo      | Tamaño   | Tipo de partición (Id) |
|-----------|-----------|----------|------------------------|
| `sdc1`    | primaria  | 200 MiB  | `82` Linux swap |
| `sdc2`    | primaria  | 100 MiB  | `8e` Linux LVM |
| `sdc3`    | primaria  | 200 MiB  | `8e` Linux LVM |
| `sdc4`    | extendida | contenedor | no se modifica |
| `sdc5`    | lógica    | 200 MiB  | `8e` Linux LVM |
| `sdc6`    | lógica    | 1,3 GiB  | `8e` Linux LVM |

El tipo se cambia dentro de `fdisk` con `t`, el número de partición y el código hexadecimal (`L` lista todos). La partición extendida no cambia de tipo porque es solamente un contenedor de particiones lógicas.

### Disco `/dev/sdb` (5 GiB)

Como `sdc` resulta chico, se particionó también `sdb` con dos particiones primarias, ambas de tipo `8e` (Linux LVM):

| Partición | Tamaño  |
|-----------|---------|
| `sdb1`    | 1,5 GiB |
| `sdb2`    | 3,5 GiB |

## 3. Memoria swap

Se consulta la memoria con `free -h` y las áreas de swap activas con `swapon -s` o `cat /proc/swaps`.

```bash
sudo mkswap /dev/sdc1     # prepara la partición como área de swap
sudo swapon /dev/sdc1     # la habilita
free -h
swapon -s
```

`mkswap` solo prepara la partición; hasta ejecutar `swapon` el sistema no la usa.

Estado verificado en la VM (`swapon -s`):

```
Filename      Type        Size     Used  Priority
/swapfile     file        524284   0     -2
/dev/sdc1     partition   204796   0     -3
```

La VM ya traía un archivo de swap (`/swapfile`, 512 MB). La partición `sdc1` se suma con prioridad menor. Al intentar repetir `mkswap` o `swapon` sobre `sdc1`, el sistema avisó que el dispositivo ya estaba en uso, porque ya estaba activa como swap.

## 4. LVM

### 4.1 Volúmenes físicos

Si el disco tenía firmas previas, se limpian con `wipefs -a <partición>` antes de crear el volumen físico.

```bash
sudo pvcreate /dev/sdc2 /dev/sdc3 /dev/sdc5 /dev/sdc6 /dev/sdb1 /dev/sdb2
sudo pvs
```

No se incluyen `sdc1` (quedó como swap) ni `sdc4` (extendida, contenedor).

### 4.2 Grupos de volúmenes

```bash
sudo vgcreate vg_datos /dev/sdb1 /dev/sdc2    # para almacenar datos
sudo vgcreate vg_web   /dev/sdb2              # para el servidor web
sudo vgs
```

Resultado final:

```
VG        #PV #LV #SN Attr   VSize  VFree
vg_datos    2   0   0 wz--n- <1.59g <1.59g
vg_web      1   3   0 wz--n- <3.50g <1.50g
```

### 4.3 Volúmenes lógicos de `vg_web`

`-L` indica el tamaño y `-n` el nombre:

```bash
sudo lvcreate -L 1G   vg_web -n lv_web1
sudo lvcreate -L 512M vg_web -n lv_web2
sudo lvcreate -L 512M vg_web -n lv_web3
sudo lvs
```

```
LV      VG     Attr       LSize
lv_web1 vg_web -wi-ao----   1.00g
lv_web2 vg_web -wi-ao---- 512.00m
lv_web3 vg_web -wi-ao---- 512.00m
```

### 4.4 Formateo

Los volúmenes quedan en `/dev/mapper/` (se consulta con `sudo fdisk -l`). Se formatean con ext4:

```bash
sudo mkfs.ext4 /dev/mapper/vg_web-lv_web1
sudo mkfs.ext4 /dev/mapper/vg_web-lv_web2
sudo mkfs.ext4 /dev/mapper/vg_web-lv_web3
```

## 5. Montaje

```bash
sudo mkdir -p /mnt/web1 /mnt/web2 /mnt/web3
sudo mount /dev/mapper/vg_web-lv_web1 /mnt/web1
sudo mount /dev/mapper/vg_web-lv_web2 /mnt/web2
sudo mount /dev/mapper/vg_web-lv_web3 /mnt/web3
df -h
```

```
/dev/mapper/vg_web-lv_web1  974M  24K  907M  1%  /mnt/web1
/dev/mapper/vg_web-lv_web2  488M  24K  452M  1%  /mnt/web2
/dev/mapper/vg_web-lv_web3  488M  24K  452M  1%  /mnt/web3
```

## 6. Persistencia (`/etc/fstab`)

Para que el montaje y el swap sobrevivan a un reinicio se obtienen los UUID con `sudo blkid` y se agregan al final de `/etc/fstab` (se hizo antes un respaldo con `sudo cp /etc/fstab /etc/fstab.bak`):

```
UUID=52da3440-9c0c-457e-bc60-c152fa1480b9 /mnt/web1 ext4 defaults,nofail 0 2
UUID=685e41e8-9cf9-4ec2-8dfd-6696b364da89 /mnt/web2 ext4 defaults,nofail 0 2
UUID=c3268687-897d-474e-81e5-c3b6e66ad4d8 /mnt/web3 ext4 defaults,nofail 0 2
UUID=58b132e6-021b-4619-8aa9-54eaf38f7aaf none      swap sw               0 0
```

La opción `nofail` evita que el arranque falle si algún volumen no está disponible.

Se probó con:

```bash
sudo umount /mnt/web1 /mnt/web2 /mnt/web3
sudo mount -a
sudo swapoff /dev/sdc1
sudo swapon -a
df -h
swapon -s
```

Los tres volúmenes volvieron a montarse y `swapon -s` volvió a listar `/swapfile` y `/dev/sdc1`.

## 7. Estado final

```
NAME                 SIZE TYPE MOUNTPOINTS
sdb                    5G disk
├─sdb1               1.5G part
└─sdb2               3.5G part
  ├─vg_web-lv_web1     1G lvm  /mnt/web1
  ├─vg_web-lv_web2   512M lvm  /mnt/web2
  └─vg_web-lv_web3   512M lvm  /mnt/web3
sdc                    2G disk
├─sdc1               200M part [SWAP]
├─sdc2               100M part
├─sdc3               200M part
├─sdc4                 1K part
├─sdc5               200M part
└─sdc6               1.3G part
```

```
PV         VG       Fmt  Attr PSize   PFree
/dev/sdb1  vg_datos lvm2 a--  <1.50g  <1.50g
/dev/sdb2  vg_web   lvm2 a--  <3.50g  <1.50g
/dev/sdc2  vg_datos lvm2 a--   96.00m  96.00m
/dev/sdc3           lvm2 ---  200.00m 200.00m
/dev/sdc5           lvm2 ---  200.00m 200.00m
/dev/sdc6           lvm2 ---    1.31g   1.31g
```

`vg_datos` quedó creado con 2 volúmenes físicos y sin volúmenes lógicos. Las particiones `sdc3`, `sdc5` y `sdc6` quedaron libres como volúmenes físicos sin asignar a ningún grupo.

## Conclusión

Con este trabajo se practicó el ciclo completo de administración de almacenamiento en Linux: particionar con `fdisk`, habilitar swap, crear volúmenes físicos, grupos y volúmenes lógicos con LVM, formatearlos y dejarlos montados de forma persistente con `/etc/fstab`. La ventaja principal de LVM frente al particionado tradicional es que permite agrupar varios discos o particiones en un mismo grupo y asignar o cambiar el tamaño de los volúmenes sin depender de la ubicación física de los datos.
