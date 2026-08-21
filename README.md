# Тестовое задание Intern/Junior DevOps-инженера от «RocketDev»

`Цель`:
- развертывание WordPress + MySQL
- Prometheus+node_exporter для сбора общих метрик ОС
- Grafana для настройки отображения на дашборде OS General параметров "отправленный/полученный трафик, количество свободного места, количество файловых дескрипторов"
- nginx в качестве прокси сервера на определенные доменные имена+базовая защита сайта(ограниченный список ip адресов), настройка самоподписанного сертификата SSL для site.local
- настройка fail2ban для базовой защиты от брутфорса на уровне ssh.

> Все это должно быть реализованно в докер контейнерах с использованием docker compose.

## Nginx
Проксирует запросы:
- site.local по HTTP/HTTPS → WordPress
- metrics.local по HTTP/HTTPS → Grafana

## Стэк
- Docker + docker-compose
- WordPress 6 + MySQL 8
- nginx (reverse proxy, ssl)
- Prometheus + node_exporter + grafana
- fail2ban
- GitHub Action(CI)

## Требования к системе
- Linux (проверялось на Ubuntu 24.04)
- Docker engine (https://docs.docker.com/engine/install/ubuntu/)

## Быстрый запуск
1. клонируйте репозиторий:
```bash
git clone https://github.com/vsrov161/RocketDevTask.git
cd RocketDevTask
```
2. добавьте домены в hosts(ubuntu):
```bash
echo "127.0.0.1 site.local metrics.local" | sudo tee -a /etc/hosts
```
3. добавление доменов в Hosts в ОС Windows:
- открыть файл по пути `C:\Windows\System32\drivers\etc\hosts`
- в самом конце файла, над строкой `# End of section` вставить
```
127.0.0.1 site.local
127.0.0.1 metrics.local
```
4. самоподписанные сертификаты уже в репозитории в `/certs`, но при необходимости вы можете их пересоздать:
```bash
mkdir -p certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/site.local.key -out certs/site.local.crt -subj "/CN=site.local"
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/metrics.local.key -out certs/metrics.local.crt -subj "/CN=metrics.local"
```
5. запустить проект:
```bash
docker compose up -d
```
6. установка и запуск `fail2ban`:
```bash
sudo apt update
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
# проверяем статус сервиса
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

7. попробуйте открыть в браузере:
- WordPress: https://site.local
- Grafana: https://metrics.local (логин/пароль: `admin/admin`). Пароль можно задать вручную в параметре `GF_SECURITY_ADMIN_PASSWORD: <ПАРОЛЬ>` в `docker-compose.yml`


> Браузер покажет предупреждение о том, что сертификат Not secure — это ожидаемое поведение, предупреждение можно проигнорировать.

## Мониторинг
Мониторинг осуществляется в Web UI Grafana по адресу https://metrics.local
- На главной странице, в правом верхнем углу, справа от текстового поля поиска нажмите `+`
- Выберите `import dashboard`
- На открывшейся старнице нажмите `Import dashboard` и выберите файл `json` вручную, либо перетащите его в это окно
- Имя дашборда `Os General` и `UID` можете оставить без изменений, нажмите кнопку `Import` чтобы завершить добавление
> Файл `OS General-dashboard.json` лежит в `grafana/dashboard`

## GitHub Actions
> С пайплайном вы можете ознакомиться по ссылке https://github.com/vsrov161/RocketDevTask/actions/

## Принятые решения
- конфиги хранятся на хосте, монтируются с параметром `ro` (read-only) - **требование ТЗ**
- использовать docker compose, так как он позволяет создать единый набор конфигураций для каждого используемого ПО, благодаря этому весь стек поднимается одной командой `docker compose up -d` (**+требование ТЗ**)
- использовать самоподписанный сертификат для `site.local / metrics.local` - **требование ТЗ**
- мониторинг: метрики ОС собираются node_exporter, хранятся в Prometheus, отображаются в Grafana - **требование ТЗ**
- ограничения для wp-admin/wp-login.php по IP реализованы на уровне nginx(`allow/deny`) - **требование ТЗ**
- базовая защита SSH от брутфорса реализована с использванием `fail2ban`, так как это ПО "заточенно" именно под брутфорс, он смотрит логи SSH, считает неудачные попытки входа и банит IP-адреса, это проще и безопаснее ручной настройки `iptables` (**+требование ТЗ**)
> Пример jail-конфига лежит в fail2ban/jail.local в репозитории. На деплой-хосте fail2ban читает /etc/fail2ban/jail.local, поэтому файл нужно скопировать туда.:
```bash
sudo cp fail2ban/jail.local /etc/fail2ban/jail.local
```

## Трудозатраты
На выполнение задания ушло примерно 15 часов чистого времени, распределенных на 3 рабочих дня. Это время включало настройку инфраструктуры, изучение документации по взаимодействию сервисов (особенно Nginx + SSL + WordPress), отладку конфигураций, создание дашборда в Grafana, сопутствующий траблшутинг, и подготовку README.