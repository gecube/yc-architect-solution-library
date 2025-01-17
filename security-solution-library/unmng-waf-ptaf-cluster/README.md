# Отказоустойчивая эксплуатация PT Application Firewall на базе Yandex.Cloud
Цель демо: Установка PT Web Application Firewall (далее PT WAF) в Yandex.Cloud в отказоустойчивой конфигурации.

# Содержание:
- Описание
- Развертывание
- Описание шагов работы с PT WAF
- Проверка прохождения траффика и отказоустойчивости
- Дполнительные материалы: настройка кластеризации PT WAF и настройка Yandex Application LoadBalancer 

## Описание:
В рамках workshop будут выполнены:
- установка инфраструктуры с помощью terraform (infrastructure as a code)
- инсталяция и базовая конфигурация PT WAF cluster в 2-х зонах доступности Yandex.Cloud

Отказоучстойчивость обеспечивается за счет:
- кластеризации самих PT WAF в режиме active-active
- балансировки траффика с помощью External-LB Yandex.Cloud
- cloud-finction Yandex.Cloud, которая отслеживает состояние PT WAF и в случаи их падения направляет траффик на приложения напрямую - "BYPASS"

#### Сценарий окружения:
Предполагается, что в Yandex.Cloud у Клиента уже развернут небезопасный сценарий публикации ВМ наружу: ВМ с веб приложениями в 2-х зонах доступности. Также внешний сетевой балансировщик нагрузки. 

*для установки целой схемы снуля необходимо использовать playbook из папки "from-scratch"

#### Схема до:
![ha-proxy-До](https://user-images.githubusercontent.com/85429798/127031869-5be84785-0566-4cc5-a1f6-d041d051feb4.png)




#### Схема после:
![ha-proxy-после](https://user-images.githubusercontent.com/85429798/127031876-bed5eed2-4218-4dea-986d-4bebc1b2f5c5.png)




## Подготовка/Пререквизиты:
- установить и настроить [yc client](https://cloud.yandex.ru/docs/cli/quickstart)
- установить [terraform](https://www.terraform.io/downloads.html)
- установить [jq](https://macappstore.org/jq/)

## Развертывание

#### Развертывание terraform:

- скачать архив с файлами [pt_archive.zip](https://github.com/yandex-cloud/yc-architect-solution-library/blob/main/security-solution-library/unmng-waf-ptaf-cluster/main/pt_archive.zip)
- перейти в папку с файлами
- вставить необходимые параметры в файле variables.tf (в комментариях указаны необходимые команды yc для получения значений)
- выполнить команду инициализации terraform

```
terraform init

```
- выполнить команду импорта load-balancer

```
terraform import yandex_lb_network_load_balancer.ext-lb $(yc load-balancer network-load-balancer list --format=json | jq '.[].id' | sed 's/"//g') 
```

- выполнить команду запуска terraform
```
terraform apply
```
- включить NAT на subnet: ext-subnet-a, ext-subnet-b (для того, чтобы PT WAF мог выходить в интернет за обновлениями и активировать лицензию)
- назначить Security Group "app-sg" на ВМ: app-a, app-b

[<img width="1135" alt="image" src="https://user-images.githubusercontent.com/85429798/126979165-eb4c9e6b-806d-401c-bec1-53f54cbecef1.png">](https://www.youtube.com/watch?v=IOYw4fdn69A)

##

## Описание шагов работы с PT WAF:
Видеоинструкция этапа:

- пробрасываем порты по SSH для подключения к серверам PT AF (ВЫПОЛНЯЕМ В 2 РАЗНЫХ ОКНАХ ТЕРМИНАЛА):
```
ssh -L 22001:192.168.2.10:22013 -L 22002:172.18.0.10:22013 -L 8443:192.168.2.10:8443 -L 8444:172.18.0.10:8443 -i ./pt_key.pem yc-user@$(yc compute instance list --format=json | jq '.[] | select( .name == "ssh-a")| .network_interfaces[0].primary_v4_address.one_to_one_nat.address '| sed 's/"//g') 
```
после этого вы окажитесь в терминале ssh-a (брокер машина)



#### Настройка обработки траффика

- подключаемся на локальной машине по 127.0.0.1:8443 

- логин пароль: admin/positive (меняем)

- configuration - upstreams - create

- ip (смотрим в yc) и порт

- мапим интерфейсы
configuration - network - gateways 

в каждом редактировать: activate - галочка, network-aliases: mgmt, wan

- заводим сервис
configuration - network - services

name - application
net interface alias - wan
listen port - 80
upstream - internal

- Configuration -> Security -> Web Applications:
- указать имя сервиса
- указать созданный ранее service
- указать protection mode "detection"
- locations "/"

[![image](https://user-images.githubusercontent.com/85429798/127023351-f0731361-5ba5-429a-82e9-5cc3c14a6355.png)](https://www.youtube.com/watch?v=lCFnHanCSSE)


## Проверка прохождения траффика и отказоустойчивости
- посмотрите внешний ip адреса внешнего балансировщика нагрузки
- отклюим ptaf-a и убедимся, что траффик проходит
- отключим app-a и убедимся, что траффик проходит
- отклюим ptaf-b и убедимся, что BYPASS сработает и траффик переключится напрямую на внутренний балансировщик
- включите ptaf-a, ptaf-b обратно и убедитесь то, что траффик снова идет через ptaf

[![image](https://user-images.githubusercontent.com/85429798/127031813-f9460c50-2765-40d4-aa16-f66fc7fd70b7.png)](https://www.youtube.com/watch?v=DQYzXVKVVjg)



## Дополнительные материалы

## Настройка кластеризации PT WAF ------------------

#### Общая настройка двух серверов

- настройте на каждом из серверов пароль пользователя pt. пароль по умолчанию - positive.
для этого подключаемся к серверам PT AF (В 2 РАЗНЫХ ОКНАХ ТЕРМИНАЛА):
```
ssh pt@192.168.2.10 -p 22013 
ssh pt@172.18.0.10 -p 22013
```
- выберите вариант 0 - Exit to WSC и нажмите Enter.
- скопируйтие и вставьте команды:
```
host add 192.168.2.10 ptaf-a
host add 172.18.0.10 ptaf-b
timezone Europe/Moscow
ntp add ru.pool.ntp.org

config commit
```
- дождитесь окончания записи новой конфигурации:

██████████████████████████████████████████████████ 100 %                                                                            


#### Настройка slave-сервера
- подключитесь к ptaf-b: 
```
ssh pt@172.18.0.10 -p 22013
```

- введите команду
```
password set <мастер-пароль> (должен совпадать с тем, который задан на узле master). 
```

-  выйдет сообщение: "It doesn't look like this password is autogenerated. Use it anyway? (y/n)". нажмите Y.

- скопируйте и вставьте команды:
```
cluster set mongo replset waf
cluster set elastic replset waf
cluster set elastic nodes ptaf-b ptaf-a
```

#### Настройка master-сервера
- подключитесь к ptaf-a: 
```
ssh pt@192.168.2.10 -p 22013 
```
- выполните команду:
```
password set <мастер-пароль> (должен совпадать с тем, который задан на узле slave).  
```

- выйдет сообщение: "It doesn't look like this password is autogenerated. Use it anyway? (y/n)". нажмите Y.

- скопируйте и вставьте команды: 
```
cluster set mongo replset waf
cluster set mongo nodes ptaf-b
cluster set elastic replset waf
cluster set elastic nodes  ptaf-a ptaf-b
```

#### Создание кластера

- сначала запустим синхронизацию на SLAVE-сервере использовав команду:
```
config commit
```
- дождитесь когда на SLAVE-сервере появится сообщение: "TASK: [mongo | please configure all other nodes of your cluster]". после этого  переключитесь на MASTER-сервер и начните синхронизацию той же командой:
```
config commit
```
*в случае, если на MASTER сервере загрузка вылетит то применить config commit

- далее конфигурация на узле master остановилась на сообщении TASK: [mongo | please configure all other nodes of your cluster], просто вручную выполните команду config sync на узле SLAVE.

- на SLAVE выполнить:
```
config sync 
```
- на Master выполнить:
```
config sync
```

- на Master выполнить:
```
mongo --authenticationDatabase admin -u root -p $(cat /opt/waf/conf/master_password) waf --eval 'c = db.sentinel; l = c.findOne({_id: "license"}); Object.keys(l).forEach(function(k) { if (l[k].ip) { delete l[k].ip; l[k].hostname = "yclicense.ptsecurity.ru" }}); c.update({_id: l._id}, l)'
```

[<img width="1041" alt="image" src="https://user-images.githubusercontent.com/85429798/127007705-3a727cec-07c9-4071-80ca-1631070f83f2.png">](https://www.youtube.com/watch?v=zuTxyEeM7Vg)


## Настройка Yandex Application LoadBalancer ------------------

В данной схеме возможно использовать [Application LoadBalancer Yandex.Cloud](https://cloud.yandex.ru/docs/application-load-balancer/)

Существует подробная инструкция по [Организация виртуального хостинга](https://cloud.yandex.ru/docs/application-load-balancer/solutions/virtual-hosting)
(включая интеграцию с certificate manager для управления SSL сертификатами)

