*******start*********

#скачиваем Vagrantfile

           git clone https://github.com/MaximMiklyaev/RPM.git

#переходим в директорию:

           cd /RPM

#запускаем образ

           vagrant up

#автоматически устанавливаются модули и зависимости 

#в Vagrantfile прописанны действия для автозагрузки и исполднения.
            
            sudo -i
            yum install -y mdadm smartmontools hdparm gdisk
            yum install -y nano
            yum install -y redhat-lsb-core
            yum install -y wget
            yum install -y rpmdevtools
            yum install -y rpm-build
            yum install -y createrepo
            yum install -y yum-utils
            yum install -y pcre-devel
            yum install -y gcc
            yum install -y mc
            wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.14.1-1.el7_4.ngx.src.rpm
            rpm -i nginx-1.14.1-1.el7_4.ngx.src.rpm
            wget https://www.openssl.org/source/latest.tar.gz
            tar -xvf latest.tar.gz
            cd /root/
            yum-builddep -y rpmbuild/SPECS/nginx.spec 


#Vagrantfile поднимает VM

#подключаемся к созданной VM

           vagrant ssh

#далее заходим под SU

     sudo -i

#начинаем настройку spec файл чтоб NGINX собирался с необходимыми нам опциями 
          	
          nano /root/rpmbuild/SPECS/nginx.spec

#нажимаем

          ctrl+w  для поиска вводим %build

#вывод 

         %build
         ./configure %{BASE_CONFIGURE_ARGS} \
             --with-cc-opt="%{WITH_CC_OPT}" \
             --with-ld-opt="%{WITH_LD_OPT}" \
             --with-debug
         make %{?_smp_mflags}
         %{__mv} %{bdir}/objs/nginx \
             %{bdir}/objs/nginx-debug
         ./configure %{BASE_CONFIGURE_ARGS} \
             --with-cc-opt="%{WITH_CC_OPT}" \
             --with-ld-opt="%{WITH_LD_OPT}"
         make %{?_smp_mflags}

#приводим к такому виду

          %build
          ./configure %{BASE_CONFIGURE_ARGS} \
              --with-cc-opt="%{WITH_CC_OPT}" \
              --with-ld-opt="%{WITH_LD_OPT}" \
              --with-openssl=/home/vagrant/openssl-1.1.1h
          make %{?_smp_mflags}
          %{__mv} %{bdir}/objs/nginx \
             %{bdir}/objs/nginx-debug
          ./configure %{BASE_CONFIGURE_ARGS} \
             --with-cc-opt="%{WITH_CC_OPT}" \
             --with-ld-opt="%{WITH_LD_OPT}"
          make %{?_smp_mflags}

#вводим

          ctrl+x подтверждаем/сохраняем  y

#сборка RPM пакета (долгая прогрузка)

          rpmbuild -bb /root/rpmbuild/SPECS/nginx.spec

#проверка собраных пакетов

          ll /root/rpmbuild/RPMS/x86_64/

#вывод

          total 4392
          -rw-r--r--. 1 root root 2003504 Dec  3 08:04 nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
          -rw-r--r--. 1 root root 2489336 Dec  3 08:04 nginx-debuginfo-1.14.1-1.el7_4.ngx.x86_64.rpm

#установим nginx

          yum localinstall -y /root/rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm

#запустим nginx

          systemctl start nginx

#проверим статус nginx

          systemctl status nginx

#создаём свою директорию 

          mkdir /usr/share/nginx/html/repo

#в созданую директорию копируем nginx

          cp /root/rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm /usr/share/nginx/html/repo/

#установка percona-release-1.0-9.noarch.rpm

          wget https://downloads.percona.com/downloads/percona-release/percona-release-1.0-9/redhat/percona-release-1.0-9.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-1.0-9.noarch.rpm

#инициализируем репозиторий

          createrepo /usr/share/nginx/html/repo/

#вывод

          Spawning worker 0 with 2 pkgs
          Workers Finished
          Saving Primary metadata
          Saving file lists metadata
          Saving other metadata
          Generating sqlite DBs
          Sqlite DBs complete


#конфигурируем файл 

         nano /etc/nginx/conf.d/default.conf 

#секцию location прописываем - autoindex on

          location / {
              root   /usr/share/nginx/html;
              index  index.html index.htm;
              autoindex on;
                      }

#проверяем синтаксис и перезапускаем nginx

          nginx -t
#вывод

          nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
          nginx: configuration file /etc/nginx/nginx.conf test is successful

#перезапуск

          nginx -s reload

#curl репозиторий

          curl -a http://127.0.0.1/repo/

#вывод curl

          <html>
          <head><title>Index of /repo/</title></head>
          <body bgcolor="white">
          <h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
          <a href="repodata/">repodata/</a>                                          03-Dec-2020 08:20                   -
          <a href="nginx-1.14.1-1.el7_4.ngx.x86_64.rpm">nginx-1.14.1-1.el7_4.ngx.x86_64.rpm</a>                03-Dec-2020 08:17             2003504
          <a href="percona-release-1.0-9.noarch.rpm">percona-release-1.0-9.noarch.rpm</a>                   11-Nov-2020 21:49               16664
          </pre><hr></body>
          </html>


#добавим в curl

          cat >> /etc/yum.repos.d/miklyaev.repo << EOF
          [miklyaev]
          name=miklyaev-linux
          baseurl=http://127.0.0.1/repo
          gpgcheck=0
          enabled=1
          EOF

#проверка созданого .repo

          ls /etc/yum.repos.d/

#убедимся что репозиторий подключен

          yum repolist enabled | grep miklyaev

#вывод информации о пакете 

          miklyaev                  miklyaev-linux 

#посмотрим что в нем есть

          yum list | grep miklyaev

#установим модуль

          yum install -y percona-release	

#после изменений необходимо применять их

       createrepo /usr/share/nginx/html/repo/

*********end*********