---
title: "Proxmox, VM Linux completely into RAM À propos d'une façon d'implémenter une machine virtuelle sans disque basée sur Linux pour les clusters qui s'exécute uniquement en RAM."
categories:
  - Blog
tags:
  - VM completely into RAM
  - RAM VM
  - Diskless VM
  - HA
  - Live Migration 
  - Proxmox
---
# 3. Proxmox, VM Linux completely into RAM À propos d'une façon d'implémenter une machine virtuelle sans disque basée sur Linux pour les clusters qui s'exécute uniquement en RAM.

**La solution suivante, à mon avis, est bonne pour implémenter une machine virtuelle qui n'utilise pas de stockage, peut fonctionner en mode HA, migration en direct et effectuer diverses tâches et fournir une tolérance aux pannes, réalisée sur Linux Debian dans un cluster Proxmox.**

## 1. Nous créons une machine virtuelle qui fonctionne uniquement en RAM sans disque. Nous l'avons mis en HA.
### 1.1. Nous créons une machine virtuelle Linux de manière standard et la configurons en fonction des tâches qui lui seront assignées. Nous plaçons le disque sur un stockage partagé ou local. Ce disque sera utilisé pour le premier démarrage. Et ensuite, la machine sera déconnectée de ce disque et pourra « se déplacer » à travers les nœuds.
### 1.2. Dans le fichier /usr/share/initramfs-tools/scripts/local, recherchez les lignes 179-185 (après avoir fait une sauvegarde du fichier) :
```
   checkfs "${ROOT}" root "${FSTYPE}"

	# Mount root
	# shellcheck disable=SC2086
	if ! mount ${roflag} ${FSTYPE:+-t "${FSTYPE}"} ${ROOTFLAGS} "${ROOT}" "${rootmnt?}"; then
		panic "Failed to mount ${ROOT} as root file system."
	fi
```
І **Et nous changeons ce code en ceci :**  
```
   #checkfs "${ROOT}" root "${FSTYPE}"

	# Mount root
	# shellcheck disable=SC2086
	mkdir /ramboottmp
	mount ${roflag} -t ${FSTYPE} ${ROOTFLAGS} ${ROOT} /ramboottmp
	mount -t tmpfs -o size=100% none ${rootmnt}
	cd ${rootmnt}
	cp -rfa /ramboottmp/* ${rootmnt}
	umount /ramboottmp
```
### 1.3. Sauvegarder le fichier. Et entrez la commande dans le terminal en tant que root :  

`mkinitramfs -o /boot/initrd.img-ramboot`  

### 1.4. Nous vérifions que le fichier a été créé dans le dossier /boot et remettons l'ancien fichier local dans le dossier /usr/share/initramfs-tools/scripts/local à sa place (ou supprimons toutes nos modifications que nous avons effectuées à l'étape 1).

### 1.5. Allez dans le dossier : /etc et recherchez le fichier : fstab, enregistrez-en une copie et modifiez-le, en recherchant quelque chose comme ceci dans les premières lignes :

`UUID= 321dba83-9a22-442b-b06b-185d7afe1088 / ext4 defaults 1 1`

**et changer en :**  

`none / tmpfs defaults 0 0` 

### 1.6 Créons un menu approprié lors du chargement :
**le fichier:**
```
nano /etc/grub.d/40_custom
```
**Contenu:**
```
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'RAM-Debian GNU/Linux' --class debian --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-321dba83-9a22-442b-b06b-185d7afe1088' {
        load_video
        insmod gzio
        insmod part_msdos
        insmod ext2
        set root='hd0,msdos1'
        echo    'Loading Linux 6.1.0-28-amd64 ...'
        linux   /boot/vmlinuz-6.1.0-28-amd64 root=UUID=321dba83-9a22-442b-b06b-185d7afe1088 ro  quiet splash toram
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-ramboot
}
```
```
update-grub
```
**Nous obtenons le menu du grub.**

### 1.6. Créons maintenant ram.tar.gz, éteignons la machine virtuelle, démarrons la nouvelle machine virtuelle en mode liveCD et connectons le disque de cette machine virtuelle.

**Montez-le dans /mnt. Faisons-le:**  
```
# cd /mnt
# tar -czf /mnt/boot/ram.tar.gz .
```
### 1.7. Maintenant, lors du chargement de la machine virtuelle, nous sélectionnons le menu de démarrage RAM approprié. Après le chargement, déverrouillez le disque :

#### 1.7.1. Arrêtons l'accès au disque :

**Pour ce faire, utilisez la commande :**  
```
echo 1 > /sys/block/sda/device/delete
```
     
**Cela désactivera le périphérique /dev/sda au niveau du noyau. Le lecteur ne sera plus visible sur le système.**  

#### 1.7.2. Vérifions le statut :

**Assurons-nous que le disque n’apparaît plus dans la liste des périphériques :**  
```
lsblk
```

**Cela ressemblera à ceci :**  
```
root@debvsan:/home/vov# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                               8:0    0     8G  0 disk  
└─sda1                            8:1    0     8G  0 part  
root@debvsan:/home/vov#
```
```
echo 1 > /sys/block/sda/device/delete
```
```
lsblk
```
#### 1.7.3. Détacher un disque d'une machine virtuelle via le moniteur QEMU :

##### 1.7.3.1. Connectez-vous au moniteur QEMU pour une VM spécifique : 
```
qm monitor 105
```

**Vérifiez tous les périphériques connectés à la VM :**  
```
info block  
```
**Cela affichera tous les lecteurs connectés. Vous verrez quelque chose comme ceci :**  
```
root@pve1:~# qm monitor 105 
Entering QEMU Monitor for VM 105 - type 'help' for help
qm> info block
drive-scsi0 (#block190): /dev/pve/vm-105-disk-1 (raw)
    Attached to:      scsi0
    Cache mode:       writeback, direct
    Detect zeroes:    unmap
qm>
```

**Démonter le lecteur :**
```
device_del scsi0
```

**Vérifiez quels appareils sont connectés :**
```
info pci
```

**Vous verrez une liste de périphériques PCI, y compris le contrôleur SCSI.
Par exemple:**  
```
Bus  9, device   1, function 0:
    SCSI controller: PCI device 1af4:1004
      PCI subsystem 1af4:0008
      IRQ 10, pin A
      BAR0: I/O at 0x1000 [0x103f].
      BAR1: 32 bit memory at 0xfd800000 [0xfd800fff].
      BAR4: 64 bit prefetchable memory at 0xfc000000 [0xfc003fff].
      id "virtioscsi0"
```
**Retirer:**  
```
qm> device_del virtioscsi0
```
**Sortir:**  
```
q
```

##### 1.7.4.2. Coller dans le fichier de configuration :
```
root@pve1:~# nano /etc/pve/qemu-server/105.conf
```
**Suivant:**
```
disabled=1
```
**Exemple:**  
```
scsi0: local-lvm:vm-105-disk-1,disabled=1,aio=native,backup=0,discard=on,iothread=1,size=8G
scsihw: virtio-scsi-single,disabled=1
```
##### 1.7.4.3. Et pendant la migration nous verrons :
```
()
Task viewer: VM 105 - Migrate
OutputStatus
Stop
Download
task started by HA resource agent
2025-01-04 00:34:24 use dedicated network address for sending migration traffic (10.10.1.1)
2025-01-04 00:34:24 starting migration of VM 105 to node 'pve1' (10.10.1.1)
2025-01-04 00:34:24 starting VM 105 on remote node 'pve1'
2025-01-04 00:34:28 start remote tunnel
2025-01-04 00:34:29 ssh tunnel ver 1
2025-01-04 00:34:29 starting online/live migration on unix:/run/qemu-server/105.migrate
2025-01-04 00:34:29 set migration capabilities
2025-01-04 00:34:29 migration downtime limit: 100 ms
2025-01-04 00:34:29 migration cachesize: 512.0 MiB
2025-01-04 00:34:29 set migration parameters
2025-01-04 00:34:29 start migrate command to unix:/run/qemu-server/105.migrate
2025-01-04 00:34:30 migration active, transferred 357.9 MiB of 4.0 GiB VM-state, 586.6 MiB/s
2025-01-04 00:34:31 migration active, transferred 735.9 MiB of 4.0 GiB VM-state, 543.8 MiB/s
2025-01-04 00:34:32 migration active, transferred 1.1 GiB of 4.0 GiB VM-state, 399.1 MiB/s
2025-01-04 00:34:33 migration active, transferred 1.3 GiB of 4.0 GiB VM-state, 502.5 MiB/s
2025-01-04 00:34:34 migration active, transferred 1.6 GiB of 4.0 GiB VM-state, 243.0 MiB/s
2025-01-04 00:34:35 migration active, transferred 1.8 GiB of 4.0 GiB VM-state, 320.5 MiB/s
2025-01-04 00:34:36 migration active, transferred 2.2 GiB of 4.0 GiB VM-state, 462.0 MiB/s
2025-01-04 00:34:37 migration active, transferred 2.5 GiB of 4.0 GiB VM-state, 443.7 MiB/s
2025-01-04 00:34:38 average migration speed: 457.0 MiB/s - downtime 73 ms
2025-01-04 00:34:38 migration status: completed
2025-01-04 00:34:42 migration finished successfully (duration 00:00:18)
TASK OK
```
#HauteDisponibilité #RepriseActivité #CyberSecurité
