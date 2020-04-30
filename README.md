# Oturans_infra_repository
[![Build Status](https://travis-ci.com/Otus-DevOps-2020-02/Oturans_infra.svg?branch=master)](https://travis-ci.com/Otus-DevOps-2020-02/Oturans_infra)

## Play-travis

1. Настроен шаблон PR по умолчанию для github (.github/PULL_REQUEST_TEMPLATE.md).
2. Настроены линтеры посредством pre-commit в частности:
    'trailing-whitespace'.
    'end-of-file-fixer'.
3. Настроена связь проекта с Travis CI, подключены уведомления в чат Slack.
4. Алгоритм сборки и тестирования пока не понятен.
5. Добавлен стандарный тест с ошибкой (play-travis/test.py).
6. Исправлена ошибка в тесте (play-travis/test.py)

## Cloud-bastion

1. на GCP поднято 2 машины

bastion - 10.132.0.9 / 35.210.169.182
someinternalhost - 10.13.0.8

2. Настроен доступ по SSH к указанным машинам
   Доп задание подключиться к someinternalhost напрямую.
    На локальной машине создаем файлик ~/.ssh/config
    В него заносим:
```
Host bastion
  HostName     35.210.169.182
  User         appuser
  IdentityFile ~/.ssh/appuser
  ForwardAgent yes

Host someinternalhost
  HostName     10.132.0.8
  User         appuser
  ProxyJump    bastion
```

      сохраняем, набираем ssh someinternalhost и мы попали в нужную ВМ.

2. На ВМ bastion установлен VPN сервер Pritunl

3.1. Подключено доменное имя посредством DNS(sslip.io) - 35-210-169-182.sslip.io
3.2. Настроен валидный сертификат.
3.3. В репозиторий сохранен файл конфигурации для  OpenVPN (cloud-bastion.ovpn).

bastion_IP = 35.210.169.182
someinternalhost_IP = 10.132.0.8

## Cloud-Testapp

testapp_IP = 35.187.42.51
testapp_port = 9292

1. Создана ВМ на GCP

~~~~
gcloud compute instances create reddit-app \
--boot-disk-size=10GB \
--image-family ubuntu-1604-lts \
--image-project=ubuntu-os-cloud \
--zone europe-west1-d \
--machine-type=g1-small \
--tags puma-server \
--restart-on-failure
~~~~

2. Установлен
Ruby - install_ruby.sh
Monodb - install_mongodb.sh
Приложение  - deploy.sh

   Можно проверить по порту 35.187.42.51:9292

3. Составлен доп скрипт(startup-script.sh) для создания виртуальной машины и развертывая приложения.

   Итоговая командна gcloud будет выглядеть:

~~~~
gcloud compute instances create reddit-app \
--boot-disk-size=10GB \
--image-family ubuntu-1604-lts \
--image-project=ubuntu-os-cloud \
--zone europe-west1-d \
--machine-type=g1-small \
--tags puma-server \
--restart-on-failure \
--metadata=startup-script=\#\!/bin/bash$'\n'apt\ update$'\n'apt\ install\ -y\ ruby-full\ ruby-bundler\ build-essential$'\n'apt-key\ adv\ --keyserver\ hkp://keyserver.ubuntu.com:80\ --recv-keys\ 0xd68fa50fea312927$'\n'bash\ -c\ \'echo\ \"deb\ http://repo.mongodb.org/apt/ubuntu\ xenial/mongodb-org/3.2\ multiverse\"\ \>\ /etc/apt/sources.list.d/mongodb-org-3.2.list\'$'\n'apt\ update$'\n'apt\ install\ -y\ mongodb-org\ $'\n'systemctl\ start\ mongod$'\n'systemctl\ enable\ mongod$'\n'git\ clone\ -b\ monolith\ https://github.com/express42/reddit.git$'\n'cd\ reddit\ \&\&\ bundle\ install$'\n'puma\ -d
~~~~

4. Создание правила фаервола через gcloud

~~~~
gcloud compute --project=infra-270920 firewall-rules create default-puma-server --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:9292 --source-ranges=0.0.0.0/0 --target-tags=puma-server
~~~~

## Packer

1. Настроена сборка образов в GCP

    **reddit-base** - [packer/ubuntu16.json][1]    - только неизменяемая среда
    **reddit-full** - [packer/immutable.json][2]   - Среда + само приложение
    Пример заполнения variables - [packer/variables.json.example][3]

2. Для сборки:
**reddit-base** использован базовый образ ubuntu_1604 и скрипты:

    [packer/scripts/install_mongodb.sh][4]
    [packer/scripts/install_ruby.sh][5]

    **reddit-full** использован образ reddit-base  скрипт

    [packer/files/bake.sh][6]

    файл конфигуарции службы puma: [packer/files/puma.service][7] (настроен автозапуск)



3. Для запуска виртуальной машины составлен скрипт gcloud
[config-scripts/create-reddit-vm.sh][8]

[1]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/packer/ubuntu16.json
[2]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/packer/immutable.json
[3]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/packer/variables.json.example
[4]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/packer/scripts/install_mongodb.sh
[5]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/packer/scripts/install_ruby.sh
[6]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/packer/files/bake.sh
[7]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/packer/files/puma.service
[8]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/packer-base/config-scripts/create-reddit-vm.sh


## Terraform - 1

1. Изучили работу команд
       terraform plan
       terraform apply
       terraform show
       terraform refresh
       terraform destroy
       terraform output
       terraform taint
       terraform fmt

2. Создаем виртуальную машину на базе образа "reddit-base". Устанавливаем наше приложение в созданный инстанс reddit-app посредством посредством использования provisioner - [main.tf][9]

3. Создаем правило фаервола для нашего приложения.
```
resource "google_compute_firewall" "firewall_puma" {
  name = "allow-puma-default"
  # Название сети, в которой действует правило
  network = "default"
  # Какой доступ разрешить
  allow {
    protocol = "tcp"
    ports    = ["9292"]
  }
  # Каким адресам разрешаем доступ
  source_ranges = ["0.0.0.0/0"]
  # Правило применимо для инстансов с перечисленными тэгами
  target_tags = ["reddit-app"]
}
```
4. Добавляем **SSH-KEY** в наш проект(appuser,appuser1,appuser2)
```
 metadata = {
    ssh-keys = "appuser:${file(var.public_key_path)}appuser1:${file(var.public_key_path)}appuser2:${file(var.public_key_path)}"
  }
```

5. Input переменные вынесены в файл, пример в файле [terraform.tfvars.example][10]

6. Output переменные вынесены в файл [outputs.tf][11]


7. В рамках cамостоятельной работы:
      7.1. Определили input переменную для приватного ключа
      7.2. Определили input переменную для зоны, так же она обладает значением по умолчанию
      7.3. Воспользовались командой terraform fmt.
      7.4. Пример заполнения [terraform.tfvars.example][10]
8. Задание *(52 слайд)-несколько ключей в проект - ключи добавлены как один так и несколько ключей - [main.tf][9]
9. Задание *(53 слайд) - терраформ удаляет ключи которые не описаны в нем, что в общем то логично, он привеодит проект к тому виду который мы задали и с теми ключами которые задали пунктом выше. Пока проблемы не вижу, скорее всего нет опыта
10. Задание 2*(54 слайд) - балансировщик не настроил, не разобрался с особенностью его работы/настройки в GCP
11. Задание 2*(55 слайд) - не выполнил, копипаст. копипаст сложнее сопровождать, человеческий фактор
12. Задание 2*(56 слайд) - Выполнил добавил count [main.tf][9]

[9]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/terraform-1/terraform/main.tf
[10]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/terraform-1/terraform/terraform.tfvars.example
[11]:https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/terraform-1/terraform/outputs.tf

## Terraform - 2

1. Создали правило Firewall, для решения конфликта что правило уже создано, было импортировано правило в stage
 - terraform import google_compute_firewall.firewall_ssh default-allow-ssh
2. На практических примерах рассмотрели как влияет очередность создания ресурсов, от внутренних зависимостей одного ресурсы от другого. Далее в ДЗ это будет проверено. при разворачивании приложения.
3. Создали отдельные шаблоны посредством [./packer/db.json][12] и [./packer/app.json][13] и следовательно собрали в GCP два шаблона reddit-app-**** и reddit-base-****

4. Вынесли из основного файла приложение, базу и настройки сети в отдельные модули:
      ./terraform/module/app  - приложение
      ./terraform/module/db   - база данных
      ./terraform/module/vpc  - настройки сети
5. Создали 2 отдельных набора сред stage и prod с ипользованием ранее созданных модулей(пункт 4) полностью перенеся внутрь каждой из сред ранее настроенное. Отличие prod среды от stage это ограничение доступа к prod только с локального ip.
6. В корне дирректории ./terraform/ вместо старого проекта, теперь лежит проект по созданию на GCP backet-а
7. **Задание со звездочкой** State наших сред (stage/prod) в перенесены на backet.
```
  terraform {
    backend "gcs" {
     bucket = "storage-bucket-oturans"
      prefix = "terraform/prod"
   }
  }
```
Проведен эксперимент одновременного применения изменений для среды prod. Первый кто успевает начать применение блокирует среду и второму выходит ошибка, что в данный момент внесение изменений в GCP не возможно.

8. **Задание со 2 звездочками** С помошью Provisioner проведен деплой приложений
   reddit-app:
    ```
        provisioner "file" {
          content     = templatefile("${path.module}/files/puma.service.tpl", { database_url = var.database_url })
            destination = "/tmp/puma.service"
        }
        provisioner "remote-exec" {
            script = "${path.module}/files/deploy.sh"
        }
    ```

для того что бы передать IP создаваемой DB приложению, мы добавилии строку с переменной **DATABASE_URL** в файл **puma.service**, а посредством функции **templatefile** передали значение **DATABASE_URL** после создания **reddit-db**.

9. Измененили конфигурацию mondodb для возможности работы с ней из внешней сети. Выполнялась так же средствами Provisioner на этапе создания **reddit-db** была прозведена замена **127.0.0.1 -> 0.0.0.0** и перезапуск mondodb.
```
  provisioner "remote-exec" {
    inline = [
      "sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mongod.conf",
      "sudo systemctl stop mongod.service",
      "sudo systemctl start mongod.service"
    ]
```
[12]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/terraform-2/packer/db.json
[13]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/terraform-2/packer/app.json

## Ansible 1

1. Установлен Ansible подготовлен файл статический inventory [ini][14];  [yml][15];
2. Рассмотрена работа нескольких модулей, и отличия использования модуля от использования баш скриптов.
  - В отличии от баш скриптов модули анализируют состояние обьекта над которым будет проводиться манипуляция и следовательно повторного срабатывания не будет если это не требуется.
  - Подробно это расморено на применении playbook [clone.yml][16] в примере видно что в случае когда используется модуль git при повторном использовании ansible ничего не меняется, если же те же действия провести через баш скрипт, повторное выполнение даст ошибку.
  - Так же из данного примера (а именно когда мы удалили то что было загружено ансиблом на сервер) при повторно использовании ansible повторно загружил все на сервер, из чего видно что ansible все приводит к тому состоянию которое задано, при этом последнего состояния он не хранит.

3. Динамическая inventary [(https://medium.com/@Temikus/ansible-gcp-dynamic-inventory-2-0-7f3531b28434)][17]
  - создан файл [inventory.gcp.yml][18] со следующим содержанием
    ```
    plugin: gcp_compute
    projects:
      - infra-270920
    zones:
      - europe-west1-d
    filters: []
    auth_kind: serviceaccount
    service_account_file: ../../../.gcp/infra.json
    ```

  - создан service accaunts и ключ (infra.json) - путь на ключик добавлен в файл [inventory.gcp.yml][18]
  - в файл [ansible.cfg][19] добавлен блок **[inventory]**, подключен плагин **gcp_compute**
  -
    ```
    [inventory]
    enable_plugins = gcp_compute
    ```
  - отредактировано значения переменной **inventory**, теперь там **./inventory.gcp.yml**
    ```
    [defaults]
    inventory = ./inventory.gcp.yml
    ```
  - команда вида **ansible all -m ping** отрабатывает, но для TravicCI изменения в части динамической инвентари прищлось закоментить.

    **P.S. ДЗ получилось очень мутным. Я до сих пор не уверен что сделал то что от меня просили. Если что то не так, просьба пояснить, я постараюсь переделать.**


[14]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-1/ansible/inventory
[15]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-1/ansible/inventory.yml
[16]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-1/ansible/clone.yml
[17]: https://medium.com/@Temikus/ansible-gcp-dynamic-inventory-2-0-7f3531b28434
[18]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-1/ansible/inventory.gcp.yml
[19]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-1/ansible/ansible.cfg


## Ansible 2

1. Составлены playbook в папке ./ansible/:

   **reddit_app_one_play.yml** - один сценарий на все в одном файле
   **reddit_app_multiple_plays.yml** - отдельные сценарии в одном файле
   **db.yml** - отдельный файл/сценарий на конфиг базы данных
   **app.yml **- отдельный файл/сценарий на конфиг среды для приложения
   **deploy.yml** - отдельный файл/сценарий на выкладывание приложения
   **site.xml** - отдельный сценарий включающий 3 отдельных (db/app/deploy)

   Дополнительные файлы

   **packer_app.yml** - отдельный файл/сценарий для сборки пакером образа сервера для приложения, замена shell provisioner
   **packer_db.yml** - отдельный файл/сценарий для сборки пакером образа сервера для базы данных, замена shell provisioner

   Дополнительные файлы:
  **/templates/db_config.j2** - дополнительный конфиг для приложения, указание базы данных
  **/templates/mongod.conf.j2** - изменение конфига mongod для работы с внешним миром.
  **/files/puma.service** - Unit файл для запуска pums в systemd

2. Расмотрена работа ключей:

      **--limit** - ограничение выборки ВМ при применении playbook
      **--tags**  - ограничение применения сценариев по тегу при применении playbook
      **--check** - проверка применения playbook без внесения изменений в продакшен
      для проверки  packer_app.yml/packer_db.yml потребовалась дополнительная установка **python-apt**

3. **Задание со \***

    Настроен динамический **inventory.gcp.yml**
    ```
        plugin: gcp_compute
        projects:
          - infra-270920
        zones:
          - europe-west1-d
        filters: []
        auth_kind: serviceaccount
        service_account_file: ../../../.gcp/infra.json
        groups:
          app: "'app' in name"
          db: "'db' in name"
        compose:
          ansible_host: networkInterfaces[0].accessConfigs[0].natIPssConfigs[0].natIP[0].natIPssConfigs[0].natIP
    ```
  Блок с групировкой взять кусками из двух статей:

  https://docs.ansible.com/ansible/latest/plugins/inventory/gcp_compute.html#parameter-keyed_groups
  http://matthieure.me/2018/12/31/ansible_inventory_plugin.html

**P.S. DI Работает так как нам надо, но остается недопонимание, так как сделано по примеру, а полного описания каждого блока не нашел. Подскажете буду благодарен.**

## Ansible 3
[![Build Status](https://travis-ci.org/Oturans/trytravis-work.svg?branch=ansible-3)](https://travis-ci.org/Oturans/trytravis-work)

1. Настроили роли db и app  
2. Настроили 2 окружения stage и prod  
3. Полностью реорганизовали структуру каталога ./ansible  
4. Настроили работу с ролью jdauphant.nginx  
5. Расмотрели механизм работы с **Ansible Vault**  
6. **Задание со \*** Динамическое инвентари настроено в прошлом ДЗ.  
7. **Задание со 2\*** В TravisCI добавлены проверки [.travis.yml][20]  
    **packer validate**  
    **terraform validate** и **tflint**  
    **ansible-lint**  
    Для отладки использвуется **trytravis**  
    https://travis-ci.org/github/Oturans/trytravis-work  
    в Readme добавлен бейдж.  

[20]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-3/.travis.yml

## Ansible 4

1. Настроено окружение для работы с vagrant  
2. Для работы в дирректории ansible создан [Vagrantfile][21]  
3. В playbook site.xml добавлен новый [base.xml][22] для установки python внутри наших ВМ. Удален user.xml за не надобностью видимо.  

4. Изменили роль DB теперь она включает задания в том числе для установки mongoDB. Добавили файл [install_mongo.yml][23]
5. Задания по настройке mongoDB перенесли так же в отдельный фалй [config_mongo.yml][24]
6. Скорректировали файл заданий для роли db после действий в пунке 4 и 5 теперь он выглядит так:  
```
---
# tasks file for db

- name: Show info about the env this host belongs to
  debug:
    msg: "This host is in {{ env }} environment!!!"

- include: install_mongo.yml
- include: config_mongo.yml
```

7. Аналогиным образом перенесли установку руби в роль app. Создали файл [ruby.yml][25] в который перенесли задания по установке. А нстройку в файл [puma.yml][26].  
8. Скорректировали файл заданий для роли app после действий в пунке 7 теперь он выглядит так:
```
---
# tasks file for app

- name: Show info about the env this host belongs to
  debug:
    msg: "This host is in {{ env }} environment!!!"

- include: ruby.yml
- include: puma.yml
```

9. Дополнили Vagrantfile в части ansible провижинера для роли app  
```
app.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbooks/site.yml"
      ansible.groups = {
      "app" => ["appserver"],
      "app:vars" => { "db_host" => "10.10.10.10"}
      }
```

10. Увидели что с ранее настроенным динамическим инвентари vagrant падает в ошибкой, перенастроили [ansible.cfg][27] на статический инвентари, и vagrant перестал падать, и это при том факте что он создает свой инвентари который можно посмотреть по пути **.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory**

11. Параметризовали выбор пользоваля от которого нужно все устанавливать для роли app.
    в app/defaults/main.yml добавляем переменную deploy_user: appuser
    Модифицируем файл puma.yml будем копировать шаблон а не готовый файл, в который будем подставлять того пользователя который будет задан в переменных.  
```
  - name: Add unit file for Puma  
      template:   <= меняем тип модуля с copy на template
        src: puma.service.j2 <= используем шаблон  
        dest: /etc/systemd/system/puma.service  
      notify: reload puma    
```

Файл шаблона создаем в app/files и он будет выглядеть так:
```
[Unit]
Description=Puma HTTP Server
After=network.target
[Service]
Type=simple
EnvironmentFile=/home/{{ deploy_user }}/db_config
User={{ deploy_user }}
WorkingDirectory=/home/{{ deploy_user }}/reddit
ExecStart=/bin/bash -lc 'puma'
Restart=always
[Install]
WantedBy=multi-user.target
```

Так же заменим в оставшейся части задания **puma.yml** прямое указание пользователя на наш парраметр:
```
- name: Add config for DB connection
  template:
    src: db_config.j2
    dest: "/home/{{ deploy_user }}/db_config"
    owner: "{{ deploy_user }}"
    group: "{{ deploy_user }}"
```

Так же у нас остается ansible/playbooks/**deploy.yml**, вносим в него аналогичные изменения.  
```
---
- name: Deploy application
  become: true
  hosts: app
  tasks:
    - name: Fetch the latest version of application code
      git:
        repo: 'https://github.com/express42/reddit.git'
        dest: "/home/{{ deploy_user }}/reddit"
        version: monolith 
      notify: reload puma

    - name: Bundle install
      bundler:
        state: present
        chdir: "/home/{{ deploy_user }}/reddit"

  handlers:
  - name: reload puma
    service: name=puma state=restarted

```

12. Дополняем [Vagrantfile][21] в части указание пользователя,  добавляем блок **extra_vars**  
```
ansible.extra_vars = {
"deploy_user" => "ubuntu"
}
```

13. В процессе работы использовали команды:  
    **vagrant up** - развернуть все  
    **vagrant provision dbserver** - накатить изменения из блока provision на ВМ dbserver  
    **vagrant provision appserver** - накатить изменения из блока provision на ВМ appserver  
    **vagrant destroy -f** - удалить все созданное.  

14. **Задание со \*** Дополняем блок extra_vars [Vagrantfile][21] передеаем параметры для настройки nginx
```
ansible.extra_vars = {
          "deploy_user" =>  "ubuntu",
          "nginx_sites" => {
            "default" => [
              "listen 80",
              "server_name \"reddit\"",
              "location / { proxy_pass http://127.0.0.1:9292;}"
            ]
          }
        }
```

#### Тестирование роли  
15. Настраиваем virtualenv и в ней устанавливаем с помощью команды **pip install -r requirements.txt**  

Предварительно дополняем файл ansible/requirements.txt   
ansible>=2.4  
molecule==2.22  **<= именно так, иначе установиться 3.х  и работать ничего не будет**  
testinfra>=1.10  
python-vagrant>=0.5.15  

16. В каталоге **ansible/roles/db:** выполняем команду  

```
molecule init scenario --scenario-name default -r db -d vagrant
```

Команда создает заготовку тестов. 

17. Сохраняем готовый тест в папку [db/molecule/default/tests/test_default.py][28]

18. Настроили в тесте проверку 27017 порту, добавляем в файл из пунка 17 в конце блок
```
# check port 27017 work
def test_tcp_port_27020(host):
    mongo_port = host.socket("tcp://0.0.0.0:27017")
    assert mongo_port.is_listening
```
19. Использовали роли db и app в плейбуках packer_db.yml и
packer_app.yml, все работает!



[21]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/Vagrantfile
[22]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/playbooks/base.yml
[23]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/roles/db/tasks/install_mongo.yml
[24]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/roles/db/tasks/config_mongo.yml
[25]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/roles/app/tasks/ruby.yml
[26]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/roles/app/tasks/puma.yml
[27]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/ansible.cfg
[28]: https://raw.githubusercontent.com/Otus-DevOps-2020-02/Oturans_infra/ansible-4/ansible/roles/db/molecule/default/tests/test_default.py

