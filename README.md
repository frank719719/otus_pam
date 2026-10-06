Пишем конфигурационный файл для создания виртуальной машины с помощью vagrant

- На начальном этапе задаем параметры самой виртуальной машины и оставляем место для блока в котором будут выполнятся команды после создания.

```yaml
Vagrant.configure("2") do |config|
  config.vm.define "ubuntu-jammy" do |srv|
    srv.vm.box = "ubuntu/jammy64"

    srv.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus   = 2
      vb.name   = "ubuntu-jammy-vm"
      vb.customize ['modifyvm', :id, '--audio', 'none']
    end

    srv.vm.network :private_network,
                   ip: "192.168.56.11",
                   auto_config: true

    srv.vm.provision "shell", inline: <<-SHELL
      set -e

      # Внутри этого блока будем писать команды которые нало будет исполнить на виртуальной машине после ее создания

    SHELL
  end
end
```

- Следующим шагом добавим в конфигурационные файлы `ssh` строчку которая явно разрешит подключаться с использованием пароля. Делать это будем с помощью программы `sed` :
```bash
sed -i 's/^#PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
      sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
      systemctl restart ssh
```

```ad-info
Обратим внимание что в первом случае при поиске добавляем `#` чтобы поменять закоментированную строку на не закоментированную. Во втором случае это строка была не закоментирована поэтому знак `#` не ставил.
```

- Следующим шагом создадим пользователей, создадим для них пароли:
```bash
# Создаём пользователей с домашними каталогами
      useradd -m otusadm
      useradd -m otus

      # Задаём пароли (Ubuntu: chpasswd, не passwd --stdin)
      echo "otusadm:Otus2022!" | chpasswd
      echo "otus:Otus2022!"     | chpasswd
```

- Следующим этапом создаем группу `admin` и добавляем туда наших пользователей:
```bash
 # Группа admin уже есть в Ubuntu, но -f безопасен
      groupadd -f admin

      usermod -aG admin otusadm
      usermod -aG admin root
      usermod -aG admin vagrant
```

Если всё настроено правильно, на этом моменте мы сможем подключиться по SSH под пользователем otus и otusadm.

Далее настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни.
Выберем метод PAM-аутентификации, так как у нас используется только ограничение по времени, то было бы логично использовать метод `pam_time`, однако, данный метод не работает с локальными группами пользователей, и, получается, что использование данного метода добавит нам большое количество однообразных строк с разными пользователями. В текущей ситуации лучше написать небольшой скрипт контроля и использовать модуль `pam_exec`

 - Создадим файл-скрипт который в готовой виртуальной машине будет находиться по следующему пути: `/usr/local/bin/login.sh`

В скрипте подписаны все условия. Скрипт работает по принципу: 
Если сегодня суббота или воскресенье, то нужно проверить, входит ли пользователь в группу admin, если не входит — то подключение запрещено. При любых других вариантах подключение разрешено. 

Содержимое этого скрипта:
```bash
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
```

Создавать его будем по средствам добавления блока в скрипт vagrant.
```bash
srv.vm.provision "shell", inline: <<-SHELL
      cat >> /usr/local/bin/login.sh << 'EOF'
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
EOF
SHELL
```
```ad-note
Отмечу в отношении этого блока два важных момента.
	1) Создание файла выделил в отдельный блок `shell` потому что были проблемы с тем что не разобрался как изолировать эту комманду с созданием файла таким образом что бы это не вызывало ошибок при запуске скрипта целиком.
	2) Первый EOF в начале команды пришлось взять в одинарные кавычки. Без этого переменные (в конкретном случае PAM_USER) не записывались в конечный скрипт.
```



- Добавим права на исполнение файла:

```
chmod +x /usr/local/bin/login.sh
```

- Добавляем с помощью echo в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:

```bash
echo "auth required pam_exec.so debug /usr/local/bin/login.sh" >> /etc/pam.d/sshd
```
![[echo#Добавление строки в файл с помощью echo]]


## Получившийся скрипт целиком:


```bash
Vagrant.configure("2") do |config|
  config.vm.define "ubuntu-jammy" do |srv|
    srv.vm.box = "ubuntu/jammy64"

    srv.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus   = 2
      vb.name   = "ubuntu-jammy-vm"
      vb.customize ['modifyvm', :id, '--audio', 'none']
    end

    srv.vm.network :private_network,
                   ip: "192.168.56.11",
                   auto_config: true

      # Создаем заранее подготовленный скрипт
      srv.vm.provision "shell", inline: <<-SHELL
      cat >> /usr/local/bin/login.sh << 'EOF'
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
EOF
SHELL

      srv.vm.provision "shell", inline: <<-SHELL
      set -e

      # Включаем парольную аутентификацию
#      echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
#      echo "PasswordAuthentication yes" > /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
      sed -i 's/^#PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
      sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
      systemctl restart ssh

      # Создаём пользователей с домашними каталогами
      useradd -m otusadm
      useradd -m otus

      # Задаём пароли (Ubuntu: chpasswd, не passwd --stdin)
      echo "otusadm:Otus2022!" | chpasswd
      echo "otus:Otus2022!"     | chpasswd

      # Группа admin уже есть в Ubuntu, но -f безопасен
      groupadd -f admin

      usermod -aG admin otusadm
      usermod -aG admin root
      usermod -aG admin vagrant

      # Делаем файл login.sh исполняемым
      chmod +x /usr/local/bin/login.sh

      # Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:
      #sed -i 's|^auth required.*$|auth required pam_exec.so debug /usr/local/bin/login.sh|' /etc/pam.d/sshd
      echo "auth required pam_exec.so debug /usr/local/bin/login.sh" >> /etc/pam.d/sshd
    SHELL
  end
end

```
## Готово
