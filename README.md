# manual_kernel_update by andmisha

### Было сделано для выполнения домашнего задания:
#### 1) Выполнил установку ПО (делал установку ПО на ПК с Windows 10 Pro x64 1909 - все версии ПО для Windows):
- VirtualBox
- Vagrant
- Packer
- Git
#### 2) Зарегистрировал аккаунты:
- GitHub - https://github.com/andmisha
- VagrantCloud - https://app.vagrantup.com/andmisha
#### 3) Зарегистрировал аккаунты:
- GitHub - https://github.com/andmisha
- VagrantCloud - https://app.vagrantup.com/andmisha
#### 4) Сделал fork репозитория https://github.com/dmitry-lyutenko/manual_kernel_update:
Ссылка на форк - https://github.com/andmisha/manual_kernel_update
#### 5) Сделал git clone репозитория своего репозитория https://github.com/andmisha/manual_kernel_update на свой ПК
#### 6) Запустил виртуальную машину
```
vagrant up
```
Так как я делал задание на Windows, vagrant нормально заработал только через консоль PowerShell. В частности vagrant ssh заработал у меня только так, так как получился конфликт
c OpenSSH, который идет по умолчанию в Windows 10 Pro x64 1909 (C:\Windows\System32\OpenSSH)
#### 7) Проверил текущую версию ядра
```
[vagrant@kernel-update ~]$ uname -r
3.10.0-1127.el7.x86_64
```
#### 8) Добавил репозиторий ElRepo
```
[vagrant@kernel-update ~]$ sudo yum install -y http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```
#### 9) Запустил установку ядра kernel-ml (ml-свежая стабильная версия, lt-свежая стабильная версия c long term)
```
[vagrant@kernel-update ~]$ sudo yum --enablerepo elrepo-kernel install kernel-ml -y
```
#### 10) Обновил загрузчик Grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
Получил вывод команды об успешном обновлении и список текущих найденных ядер в системе
```
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.10.12-1.el7.elrepo.x86_64
Found initrd image: /boot/initramfs-5.10.12-1.el7.elrepo.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-1127.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1127.el7.x86_64.img
done
```
#### 11) Сделал загрузку системы с новым ядром по умолчанию и перезагрузку виртуальной машины
```
sudo grub2-set-default 0
sudo reboot
```
#### 12) После загрузки системы проверил версию ядра
```
[vagrant@kernel-update ~]$ uname -r
5.10.12-1.el7.elrepo.x86_64
```
---
Далее необходимо создать свой box для Vagrant с подготовленной ВМ с новым ядром и загрузить его на VagrantCloud
#### 13) С помощью Packer запустил создание box (в моем случае на Windows команда запуска выглядела так)
```
C:\HashiCorp\Packer\packer.exe build -force C:\OTUS\manual_kernel_update\packer\centos.json
```
Параметр -force использовал для того, чтобы Packer затерал все, что было в предыдущий запуск, так как заработало все не с первого раза
Проблемы:
- Packer выполнялся с ошибкой
```
Error: Failed to prepare build: "centos-7.7"

1 error occurred:
        * Deprecated configuration key: 'iso_checksum_type'. Please call `packer fix`
against your template to update your template to be compatible with the current
version of Packer. Visit https://www.packer.io/docs/commands/fix/ for more
detail.
```
Документация Packer показала ключ fix для проверки json-файла, в результате чего директива iso_checksum теперь выглядит так
```
"iso_checksum": "sha256:07b94e6b1a0b0260b94c83d6bb76b26bf7a310dc78d7a9c7432809fb9bc6194a",
```
Было так:
```
"iso_checksum": "07b94e6b1a0b0260b94c83d6bb76b26bf7a310dc78d7a9c7432809fb9bc6194a",
"iso_checksum_type": "sha256",
```
- Packer почти в самом конце сборки box выдавал ошибку
```
==> centos-7.7: Provisioning with shell script: scripts/stage-1-kernel-update.sh
    centos-7.7: >>> /etc/sudoers.d/vagrant: syntax error near line 1 <<<
    centos-7.7: sudo: parse error in /etc/sudoers.d/vagrant near line 1
    centos-7.7: sudo: no valid sudoers sources found, quitting
    centos-7.7: sudo: unable to initialize policy plugin
==> centos-7.7: Provisioning step had errors: Running the cleanup provisioner, if present...
```
Проверил на работающей ВМ (на той, что создается в момент выполнения Packer) - файл /etc/sudoers.d/vagrant на месте и содержимое визуально корректное.
В итоге заменил в файле packer/http/vagrant.ks команды создания файла в sudoers.d и запись содержимого в него
Было так:
```
# Add vagrant to sudoers
#cat > /etc/sudoers.d/vagrant << EOF_sudoers_vagrant
#vagrant        ALL=(ALL)       NOPASSWD: ALL
#EOF_sudoers_vagrant
#/bin/chmod 0440 /etc/sudoers.d/vagrant
#/bin/sed -i "s/^.*requiretty/#Defaults requiretty/" /etc/sudoers
```
Стало так:
```
# Add vagrant to sudoers
/bin/echo "vagrant  ALL=(ALL) NOPASSWD:ALL" | /bin/tee /etc/sudoers.d/vagrant
/bin/chmod 0440 /etc/sudoers.d/vagrant
```
И сборка box с помощью Packer завершилась успехом :-)
```
Build 'centos-7.7' finished after 12 minutes 23 seconds.

==> Wait completed after 12 minutes 23 seconds

==> Builds finished. The artifacts of successful builds are:
--> centos-7.7: 'virtualbox' provider box: centos-7.7.1908-kernel-5-x86_64-Minimal.box
```
---
В завершении необходимо выполнить тестирование готового box и публикацию на VagrantCloud.
#### 13) Выполнил импорт box
```
PS C:\OTUS\manual_kernel_update> vagrant box add --name centos-7-5 .\packer\centos-7.7.1908-kernel-5-x86_64-Minimal.box
```
#### 14) Проверил, что импорт проше успешно
```
PS C:\OTUS\manual_kernel_update> vagrant box list
centos-7-5 (virtualbox, 0)
centos/7   (virtualbox, 2004.01)
```
#### 15) Сделал отдельную папку centos-7-5 с Vagrantfile на основе manual_kernel_update, переименовав box в centos-7-5
#### 16) Сделал vagrant up
#### 17) И проверил версию ядра
```
[vagrant@kernel-update ~]$ uname -r
5.10.12-1.el7.elrepo.x86_64
```
#### 18) При загрузке в VagrantCloud постоянно получал ошибку
```
PS C:\OTUS\manual_kernel_update\centos-7-5> vagrant cloud publish andmisha/centos-7-5 1.0 virtualbox centos-7.7.1908-kernel-5-x86_64-Minimal.box --force --release
You are about to publish a box on Vagrant Cloud with the following options:
andmisha/centos-7-5:   (v1.0) for provider 'virtualbox'
Automatic Release:     true
Saving box information...
Failed to create box andmisha/centos-7-5
Vagrant Cloud request failed - Resource not found!
```
Поэтому выполнил загрузку в ручную, ссылка - https://app.vagrantup.com/andmisha/boxes/centos-7-5/versions/1.0
