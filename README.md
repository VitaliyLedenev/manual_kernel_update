# Обновить ядро в базовой системе
## Подготовка окружения для выполнения домашней работы
- В качестве ОС как-бы хост машины выбираем Centos8
- Провайдером виртуализации будет служить boxes
- Ставим в boxes [CentOS-8.1.1911-x86_64-boot.iso](https://mirror.yandex.ru/centos/8.1.1911/isos/x86_64/CentOS-8.1.1911-x86_64-boot.iso "CentOS-8.1.1911-x86_64-boot.iso"). При установке, в  источниках откуда брать пакеты ставим repolist:
http://mirror.corbina.net/pub/Linux/centos/8/BaseOS/x86_64/os/
- и далее ставим minimal
### Ставим необходимый набор ПО
* прежде отключаем selinux в /etc/sysconfig/selinux  и 
```shell
setenforce 0
reboot
```
* Устанавливаем необходимо ПО
```shell
dnf install epel-release -y && \
dnf install htop mc net-tools psmisc git rsync glibc-langpack-ru  -y && \
dnf groupinstall "Development Tools" -y && \
dnf install elfutils-libelf-devel && cd /usr/src && \
curl -O https://releases.hashicorp.com/vagrant/2.2.7/vagrant_2.2.7_x86_64.rpm && \ 
dnf install vagrant_2.2.7_x86_64.rpm - y &&  \
curl -O https://download.virtualbox.org/virtualbox/rpm/rhel/virtualbox.repo && \
mv virtualbox.repo /etc/yum.repos.d/ && \
dnf update -y && dnf install VirtualBox-6.1.x86_64 -y && \
/usr/lib/virtualbox/vboxdrv.sh setup && \
usermod -a -G vboxusers root && \
curl -O https://releases.hashicorp.com/packer/1.5.1/packer_1.5.1_linux_amd64.zip && \
unzip packer_1.5.1_linux_amd64.zip && \
mv ./packer /usr/local/bin/ && \
chmod +x /usr/local/bin/packer && type packer
```
### Скачиваем склонированный репозиторий 
```shell
git clone  git@github.com:VitaliyLedenev/manual_kernel_update.git
```
### Входим в vm с vargant и обновляем ядро до последнего
```shell
vagrant up && vargant ssh
```
- отключаем selinux, ставим ядро из репо, обновляем конфиг груб, перезагружаемся
```shell
sudo echo "SELINUX=disabled" > /etc/sysconfig/selinux && sudo setenforce 0 && \
sudo yum install -y http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm && \
sudo yum --enablerepo elrepo-kernel install kernel-ml -y && \
 sudo grub2-mkconfig -o /boot/grub2/grub.cfg && \
 sudo grub2-set-default 0 && \
 sudo reboot 
```
- входим после перезагрузки и проверяем с каким ядром загрузилась ос
```shell
uname -r
```
## Собираем собственный образ в packer
- в каталоге packer, в файле centos.json имеем описание того что мы хотим собрать, в том числе описывается размещение shell скриптов, в которых описываем установку и конфигурацию ПО, упаковку всего этого добра в образ.

- содержимое centos.json
```json
{
  "variables": {
    "artifact_description": "CentOS 7.7 with kernel 5.x",
    "artifact_version": "7.7.1908",
    "image_name": "centos-7.7",
    "headless": "true"
    
  },

  "builders": [
    {
      "name": "{{user `image_name`}}",
      "type": "virtualbox-iso",
      "vm_name": "packer-centos-vm",
      "headless": "{{user `headless`}}",
   
      "boot_wait": "10s",
      "disk_size": "10240",
      "guest_os_type": "RedHat_64",
      "http_directory": "http",

      "iso_url": "http://mirror.yandex.ru/centos/7.7.1908/isos/x86_64/CentOS-7-x86_64-Minimal-1908.iso",
      "iso_checksum": "9a2c47d97b9975452f7d582264e9fc16d108ed8252ac6816239a3b58cef5c53d",
      "iso_checksum_type": "sha256",

      "boot_command": [
        "<tab> text ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/vagrant.ks<enter><wait>"
      ],

      "shutdown_command": "sudo -S /sbin/halt -h -p",
      "shutdown_timeout" : "5m",

      "ssh_wait_timeout": "20m",
      "ssh_username": "vagrant",
      "ssh_password": "vagrant",
      "ssh_port": 22,
      "ssh_pty": true,

      "output_directory": "builds",

      "vboxmanage": [
        [  "modifyvm",  "{{.Name}}",  "--memory",  "1024" ],
        [  "modifyvm",  "{{.Name}}",  "--cpus",  "2" ]
      ],

      "export_opts":
      [
        "--manifest",
        "--vsys", "0",
        "--description", "{{user `artifact_description`}}",
        "--version", "{{user `artifact_version`}}"
      ]

    }
  ],

  "post-processors": [
    {
      "output": "centos-{{user `artifact_version`}}-kernel-5-x86_64-Minimal.box",
      "compression_level": "7",
      "type": "vagrant"
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "execute_command": "{{.Vars}} sudo -S -E bash '{{.Path}}'",
      "start_retry_timeout": "1m",
      "expect_disconnect": true,
      "pause_before": "20s",
      "override": {
        "{{user `image_name`}}" : {
          "scripts" :
            [
              "scripts/stage-1-kernel-update.sh",
              "scripts/stage-2-clean.sh"
            ]
        }
      }
    }
  ]
}
```

- обращаем внимание на строки 6 и 15, без этого vbox будет пытаться запустить gui, которому взяться неоткуда на хост-машине.
### Запускаем сборку
```json
packer build centos.json
```
- если всё будет хорошо, то будет скачан исходный iso-образ CentOS, установлен на виртуальную машину в автоматическом режиме, обновлено ядро и осуществлен экспорт в указанный нами файл. Если не вносилось изменений в предложенные файлы, то в текущей директории мы увидим файл centos-7.7.1908-kernel-5-x86_64-Minimal.box. Он и является результатом работы packer.

## Тестируем полученный образ
- выполняем импорт образа в vagrant
```shell
vagrant box add --name centos-7-5 centos-7.7.1908-kernel-5-x86_64-Minimal.box
```
- проверяем, что он импортировался:
```shell
$ vagrant box list
centos-7-5            (virtualbox, 0)
```
### Теперь запустим образ
- но прежде нам потребуется конфиг Vagrant. создадим каталог test и можно скоипровать уже имеющийся или инициировать новый и далее его настроить:
```shell
vagrant init centos-7-5
```
- но мы скопируем:
```shell
mkdir test; cd test; cp ../Vagrant ./; ls -l
```
- и отредактируем файл, заменив имя на наше:
```shell
:box_name => "centos-7-5",
```

```
- запустим образ, войдём и проверим ядро:
```shell
vagrant up && vagrant ssh
uname -r
```
Если все в порядке, то машина будет запущена и загрузится с новым ядром.
- Удалим тестовый образ из локального хранилища:
```shell
vagrant box remove centos-7-5
```

## Vagrant cloud
- можно поделиться своим образом и далее использовать его по необходимости, Vagrant предоставляет такую возможность.
- чтобы загрузить в облако выполнем предварительную [регистрацию](https://app.vagrantup.com/ "регистрацию")
- получим токен
- залогинемся  отправим образ:
```shell
vagrant login --token 1dWYw0kZw2WmeA.atlasv1.BwaYRSqbXLaZMfBNjIw26yzt9XaNCAxY1uOU34NZh84wEt9BR8XaL7sI9ZSNZeK1nu0
```
- отправляем в облако:
```shell
vagrant cloud publish --release VitaliyLedenyov/centos-7-5 1.0 virtualbox  centos-7.7.1908-kernel-5-x86_64-Minimal.box
```
- здесь:
  - ** cloud publish** - загрузить образ в облако;
  - **release** - указывает на необходимость публикации образа после загрузки;
  - **<username>/centos-7-5 **- username, указаный при публикации и имя образа;
 - **1.0** - версия образа;
 - **virtualbox** - провайдер;
 - **centos-7.7.1908-kernel-5-x86_64-Minimal.box** - имя файла загружаемого образа;

В результате создан и [загружен в vagrant cloud образ виртуальной машины](https://app.vagrantup.com/VitaliyLedenyov/boxes/centos-7-5 "загружен в vagrant cloud образ виртуальной машины"). Данный подход позволяет создать базовый образ виртульной машины с необходимыми обновлениями или набором предустановленного ПО. К примеру при создании MySQL-кластера можно создать образ с предустановленным MySQL, а при развертывании нужно будет добавить или изменить только настройки (то есть отличающуюся часть). Таким образом существенно экономя затрачиваемое время.
