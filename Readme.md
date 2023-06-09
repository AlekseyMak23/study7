Домашнее задание "Работа с загрузчиком"

1. Создаем виртуальную машину

- Создаем Vagrantfile с учетом требований;
- Включаем вм - vagrant up.


2. Попасть в систему без пароля несколькими способами

Для получения доступа необходимо открыть GUI VirtualBox, запустить виртуальную машину и при выборе ядра для загрузки зайти в grub, нажав клавишу  e.
Попадаем в окно, где можно изменить параметры загрузки:

- Способ 1. 'init=/bin/sh'

В конце строки, начинающейся с linux16 и убираем из нее все значение console, в частности: 'console=tty0 console=ttyS0,115200n8', добавляем 'init=/bin/sh' и нажимаем сtrl-x для загрузки в систему в режиме Read-Only.
Перемонтировать ее в режим Read-Write, можно командой - 'mount -o remount,rw /'
Проверяем записав данные в любой файл или прочитав вывод команды - 'mount | grep root'
Далее можно поменять пароль командой passwd
Проверяем включен для SELUX cat /etc/selinux/config, если SELINUX=Enforcing, то меняем или на Permissive или Disabled, перезагружаемся.
Заходим в систему по новому паролю.


- Способ 2. 'rd.break'

В конце строки, начинающейся с linux16 и убираем из нее все значение console, в частности: 'console=tty0 console=ttyS0,115200n8', добавляем 'rd.break' и нажимаем сtrl-x для загрузки в систему в режиме Read-Only.
Произойдет загрузка системы в аварийном режиме, далее выполняем команду перемонтирования корня для чтения и записи - 'mount -o remount,rw /sysroot', далее 'chroot /sysroot'.
Далее можно поменять пароль, выполнив команду 'passwd' или 'passwd root'.
После смены пароля необходимо создать скрытый файл .autorelabel в /, выполнив touch /.autorelabel, этот файл нужен, для того чтобы выполнить relabel файлов в системе, если selinux включен и находиться в режиме Enforcing.
После чего можно перезагружаться и заходить в систему с новым паролем.

- Способ 3. 'rw init=/sysroot/bin/sh'
В строке, начинающейся с linux16 и убираем из нее все значение console, а частности: 'console=tty0 console=ttyS0,115200n8', заменяем ro на 'rw init=/sysroot/bin/sh' и нажимаем сtrl-x для загрузки в систему.
Файловая система сразу смонтирована в режим Read-Write.
Далее можно поменять пароль, выполнив команду 'passwd' или 'passwd root'.
После смены пароля необходимо создать скрытый файл .autorelabel в /, выполнив touch /.autorelabel, этот файл нужен, для того чтобы выполнить relabel файлов в системе, если selinux включен и находиться в режиме Enforcing.


3. Установить систему с LVM, после чего переименовать VG

- Посмотрим текущее состояние системы:
~~~
[root@lvm ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0
~~~

Переименуем:
~~~
[root@lvm ~]# vgrename VolGroup00 OtusRoot
  Volume group "VolGroup00" successfully renamed to "OtusRoot"
~~~

- Далее правим /etc/fstab, /etc/default/grub, /boot/grub2/grub.cfg. Везде заменяем старое название на новое.
~~~
[root@lvm ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/OtusRoot-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END

[root@lvm ~]# cat /etc/default/grub
GRUB_TIMEOUT=10
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
~~~

- Пересоздаем initrd image, чтобы он знал новое название Volume Group:
~~~
[root@lvm ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
~~~

- Перезагружаемся и если все сделано правильно, успешно грузимся с новым именем Volume Group и проверяем:
~~~
[root@lvm ~]# vgs
  VG       #PV #LV #SN Attr   VSize   VFree
  OtusRoot   1   2   0 wz--n- <38.97g    0
~~~


4. Добавить модуль в initrd

- Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того, чтобы добавить свой модуль, создаем там папку с именем 01test:
~~~
[root@lvm ~]# mkdir /usr/lib/dracut/modules.d/01test
~~~

- В нее поместим два скрипта:
~~~
[root@lvm 01test]# cat module-setup.sh
#!/bin/bash

check() {
    return 0
}

depends() {
    return 0
}

install() {
    inst_hook cleanup 00 "${moddir}/test.sh"
}
~~~

~~~
[root@lvm 01test]# cat test.sh
#!/bin/bash

exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
cat <<'msgend'
Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
msgend
sleep 10
echo " continuing...."
~~~

- Пересобираем образ initrd:
~~~
[root@lvm 01test]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
~~~

- Проверим, какие модули загружены в образ(отредактировать grub.cfg, убрав эти опции):
~~~
[root@lvm 01test]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test
~~~

- Редактируем /boot/grub2/grub.cfg, убраем опции - rghb и quiet, перезагружаемся:
При загрузке пауза на 10 секунд и сообщение -
~~~
Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/

~~~
