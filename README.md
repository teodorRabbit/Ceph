# Установка и настройка Ubuntu
  + Установка [Ubuntu Server 14.04 LTS](http://releases.ubuntu.com/trusty/ubuntu-14.04.5-server-amd64.iso)
  + Отключение ввода пароля при обращении через sudo:  
    В файле /etc/sudoers находим строку `%sudo   ALL=(ALL:ALL) ALL` и заменяем её на `%sudo   ALL=(ALL:ALL) NOPASSWD: ALL`.     Проделываем это на всех хостах.
  + Добавление алиаса ceph='sudo ceph' (для удобства в будущем):
    Открываем файл ~/.bashrs и в конец файла добавляем строку `alias ceph='sudo ceph'`, и перезагружаем оболочку
  + Создание ssh-ключей, чтобы не вводить пароль каждый раз, при подключении к другому хосту по ssh:
    - Создаем ключ для подключения к хостам с первого без набора пароля:  
      `ssh-keygen`
    - Копируем ключ на ceph-node-01:  
      `ssh-copy-id ceph-node-01`
    - Копируем ключ на ceph-node-02:  
      `ssh-copy-id ceph-node-02`
    - Копируем ключ на ceph-node-03:  
      `ssh-copy-id ceph-node-03`

# Развертывание ceph
  + Установка ceph-deploy:
    - `sudo apt install ceph-deploy`
  + Разворачиваем Ceph:
    - Создание конфигурационного файла ceph.conf:  
      `ceph-deploy new ceph-node-01 ceph-node-02 ceph-node-03`
    - Установка Ceph 0.80.11 Jewel на ceph-node-01, ceph-node-02, ceph-node-03:  
      `ceph-deploy install --release jewel ceph-node-01 ceph-node-02 ceph-node-03`
    - Посмотреть версию (проверка того, что ceph установился):  
      `ceph -v`
  + Установка монитров для ceph:  
    `ceph-deploy mon create-initial`
  + Установка OSD:
    - Просмотреть список доступных дисковых устройств:  
      `ceph-deploy disk list ceph-node-01`
    - Уничтожение как GPT, так и MBR на устройствах sdb, sdc, sdd:  
      `ceph-deploy disk zap ceph-node-01:sdb ceph-node-01:sdc ceph-node-01:sdd`
    - Создание OSD на устройствах sdb, sdc, sdd:  
      `ceph-deploy osd create ceph-node-01:sdb ceph-node-01:sdc ceph-node-01:sdd`

    Делаем всё то-же самое
    ----------------------------------------------------------------------------------------------------------------------
    - Для ceph-node-02:  
    ```sh
    ceph-deploy disk list ceph-node-02  
    ceph-deploy disk zap ceph-node-02:sdb ceph-node-02:sdc ceph-node-02:sdd  
    ceph-deploy osd create ceph-node-02:sdb ceph-node-02:sdc ceph-node-02:sdd
    ```  

    - И для ceph-node-03:  
    ```sh
    ceph-deploy disk list ceph-node-03
    ceph-deploy disk zap ceph-node-03:sdb ceph-node-03:sdc ceph-node-03:sdd
    ceph-deploy osd create ceph-node-03:sdb ceph-node-03:sdc ceph-node-03:sdd
    ```
  + Проверяем статус ceph:  
    `ceph -s`
  + Должен быть **WARNING**: clock skew. Он появляется из-за того, что время на хостах не синхронизировано. Исправляем:  
    - Устанавливаем ntpdate и ntp-doc:  
      `sudo apt install ntp ntpdate ntp-doc`
    - В файле /etc/ntp.conf находим добавляем сервера времени
    - Перезагружаем ntp:  
      `sudo service ntp restart`
    - Проверяем, через некоторое время **WARNING** должен исчезнуть:  
      `ceph -s`

# Установка и настройка iSCSI Target
  + Создание RBD:
    - Создание образа iscsi размером 10240 МБ:  
      `sudo rbd create iscsi --size 10240`
    - Проверяем, что образ создан:  
      `sudo rbd ls`
    - Добавляем образ iscsi в карту rbd:  
      `sudo rbd map --image iscsi`
    - Проверяем, что образ iscsi добавился в карту:  
      `sudo rbd showmapped`
  + Настройка iSCSI Target:
    - Устанавливаем iSCSI Target:  
      `sudo apt install tgt`
    - В ~/ceph.conf обязательно отключаем кеширование rbd. Для этого добавляем в этот файл следующие строки:  
    ```
    [client]
    rbd_cache = false
    ```
    - Задаем экспорт по iscsi rbd тома, в файле /etc/tgt/targets.conf:  
    ```
    <target iqn.2016-11.rbdstore.iscsi.com:iscsi>
      driver iscsi
      bs-type rbd
      backing-store rbd/iscsi
      initiator-address ALL
    </target>
    ```
    - Перезапускаем iscsi target:  
      `sudo service tgt restart`
    - Просматриваем текущую настройку iSCSI Target:  
      `sudo tgt-admin -s`
    - Подключаем диск в Windows через Инициатор iSCSI, и всё!
