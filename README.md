# Домашнее задание: Работа с загрузчиком

1. Попасть в систему без пароля несколькими способами.
2. Установить систему с LVM, после чего переименовать VG.
3. Добавить модуль в initrd.

# Выполение:

## 1. Попасть в систему без пароля несколькими способами.

### Метод №1.

Включаем ВМ. При выборе ядра для загрузки нажимаем `-е` для возможности редактирования параметров загрузки.  
В строке, начинающейся с `linuz16` дописываеи `init=/bin/sh`, жмем `Сtrl+X` для запуска загрузки. Мы в системе.   
Для возможности редактирования необходимо перемонтировать рутовую ФС в режим Read-Write командой `mount -o remount,rw /`.

![image](https://user-images.githubusercontent.com/108300153/203931028-beeb8df4-908c-4901-bf4f-600d2c55e4f6.png)

### Метод №2.

Как и в предыдущем примере, во время выбора загрузки ядра в параметры загрузки в строку  с `linuz16` дописываем `rd.break`. После загрузки попадаем в `emergency mode`.  Файловая система также в режиме `Read-Only`.  
Перемонтируем корневой раздел в режиме Read-Write, чтобы выполнять команды.  
Теперь с помощью команды `chroot /sysroot/` перемещаемся в каталог в качестве root.  
Меняем пароль рута с помощью `passwd`.   
Сообщаем SELinux c помощью `touch /.autorelabel`, что в ФС изменился пароль. Это приведет к тому, что вся файловая система будет «перемаркирована».  
Перезагружаемся и заходим с новым паролем.

![image](https://user-images.githubusercontent.com/108300153/203927005-4036b5f7-0b18-4ac2-a3b0-493997bf38a8.png)

### Метод №3.

В этом методе в той же строке в параметрах загрузки вместо `ro` пишем `rw init=/sysroot/bin/sh` для загрузки ФС сразу  в режиме Read-Write.

![image](https://user-images.githubusercontent.com/108300153/203932365-c0f8f412-239e-421f-9ff9-3d0ea9609512.png)

## 2. Установить систему с LVM, после чего переименовать VG.

Меняем название VG:

```bash
[root@tw4 ~]# vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  centos   1   2   0 wz--n- 48.80g    0 
[root@tw4 ~]# vgrename centos OtusRoot
  Volume group "centos" successfully renamed to "CentosRoot"
```

Правим /etc/fstab:

```bash
[root@tw4 ~]# sed -i 's/centos/OtusRoot/g' /etc/fstab 
[root@tw4 ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Thu Nov 24 23:01:02 2022
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/OtusRoot-root /                       xfs     defaults        0 0
UUID=7c35d157-68e6-4de6-9c10-8946fc084c95 /boot                   xfs     defaults        0 0
UUID=D7DC-8134          /boot/efi               vfat    umask=0077,shortname=winnt 0 0
/dev/mapper/OtusRoot-swap swap                    swap    defaults        0 0
```

Вносим изменения в /etc/default/grub:

```bash
[root@tw4 ~]# sed -i 's/centos/OtusRoot/g' /etc/default/grub 
[root@tw4 ~]# cat /etc/default/grub 
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=OtusRoot/root rd.lvm.lv=OtusRoot/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```
Меняем все вхождения и в/boot/efi/EFI/centos/grub.cfg. 

Пересоздаем initrd image:

```bash
[root@tw4 ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-1160.el7.x86_64.img 3.10.0-1160.el7.x86_64
...
*** Creating initramfs image file '/boot/initramfs-3.10.0-1160.el7.x86_64.img' done ***
```

Перезагружаемся и проверяем название VG:

```bash
[root@tw4 ~]# vgs
  VG       #PV #LV #SN Attr   VSize  VFree
  OtusRoot   1   2   0 wz--n- 48.80g    0 
```

## 3. Добавить модуль в initrd.

Создаем директорию для нашего модуля и создаем там необходимые скрипты:
```bash
[root@tw4 ~]# mkdir /usr/lib/dracut/modules.d/01test
[root@tw4 ~]# cd /usr/lib/dracut/modules.d/01test
[root@tw4 01test]# nano module-setup.sh 
[root@tw4 01test]# nano test.sh
[root@tw4 01test]# chmod +x module-setup.sh test.sh 
```

Пересоздаем образ initrd:

```bash
[root@tw4 01test]# dracut -f -v
Executing: /sbin/dracut -f -v
...
*** Creating initramfs image file '/boot/initramfs-3.10.0-1160.el7.x86_64.img' done ***
[root@tw4 01test]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test
```
Перезагружаем систему, не забыв выключить `quiet` и `rghb` в параметрах загрузки.
