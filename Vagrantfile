# -*- mode: ruby -*-
# vi: set ft=ruby :
# Описание параметров ВМ
MACHINES = {
  # Имя DV "pam"
  :"pam" => {
              # VM box
              :box_name => "bento/centos-8.4",
              # Количество ядер CPU
              :cpus => 2,
              # Указываем количество ОЗУ (В Мегабайтах)
              :memory => 1024,
              # Указываем IP-адрес для ВМ
              :ip => "192.168.56.10",
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    # Отключаем сетевую папку
    config.vm.synced_folder ".", "/vagrant", disabled: true
    # Добавляем сетевой интерфейс
    config.vm.network "private_network", ip: boxconfig[:ip]
    # Применяем параметры, указанные выше
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      #box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
      # переносим с хостовой машины файл скрипта
      box.vm.provision "file", source: "/home/roman/project/login.sh", destination: "login.sh"
      # переносим с хостовой машины отредактированный файл sshd
      box.vm.provision "file", source: "/home/roman/project/sshd", destination: "sshd"
      box.vm.provision "shell", inline: <<-SHELL
          #Разрешаем подключение пользователей по SSH с использованием пароля
          sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
          #Перезапуск службы SSHD
          systemctl restart sshd.service
          #Переходим в root-пользователя
          sudo -i
          #Создаём пользователя otusadm и otus
          sudo useradd otusadm && sudo useradd otus
          #Создаём пользователям пароли
          echo "Otus2023!" | sudo passwd --stdin otusadm && echo "Otus2023!" | sudo passwd --stdin otus
          #Создаём группу admin
          sudo groupadd -f admin
          #Добавляем пользователей vagrant,root и otusadm в группу admin
          usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
          #копируем файл скрипта контроля подключения пользователей
          cp /home/vagrant/login.sh /usr/local/bin/
          #Добавим права на исполнение файла
          chmod +x /usr/local/bin/login.sh
          #меняем файл sshd на отредактированный из каталога обмена 
          cp /home/vagrant/sshd /etc/pam.d/sshd
  	  SHELL

    end
  end
end
