# mailcow-migrate-2new
Миграция существующего сервера на новую установку

## Требования для переноса:
  1. Активировать доступ к API на серверах для чтения и записи (это два разных ключа!)
  2. Внести рабочий IP в список разрешенных
  3. Записать данные для подключения к серверу источнику
```
XHOSTNAME=mx.mailcow.com
XAPIKEYR=0d4c744b-1276-49ac-982d-a08789dac1b6
XAPIKEYW=bac72807-f4b9-408f-838d-71d81c59e8d2
```
  4. Записать данные для подключения к серверу получателю
```
XHOSTNAME=mz.mailcow.com
XAPIKEYR=79855ede-0ea5-415f-bf40-37febb835de1
XAPIKEYW=316d975c-4ac8-43a4-8737-9ec3bd5a7a7d
```
  5. На серверах смонтирован по NFS каталог резервных копий
  6. Установлен imapsync на одном из серверов
  7. Установлен jq для работы с json

## Готовим данные на сервере источнике

1. Получаем список имеющихся ящиков от исходного сервера
```bash
curl -X 'GET' 'https://mx.mailcow.com/api/v1/get/mailbox/all' -H 'accept: application/json' -H 'X-API-Key: 0d4c744b-1276-49ac-982d-a08789dac1b6' | jq '.[] | .username' > usernames.lst
```

2. Удаляем символы кавычек из файла
```bash
sed -i 's/"//g' usernames.lst
```

3. Получаем количество почтовых ящиков
```bash
curl -X 'GET' 'https://mx.mailcow.com/api/v1/get/mailbox/all' -H 'accept: application/json' -H 'X-API-Key: 0d4c744b-1276-49ac-982d-a08789dac1b6' | jq '.[] | .username' | wc -l
```

4. Создаём файл паролей passwords.lst по количеству полученных ящиков с помощью https://passwordsgenerator.net/

5. Создаём список логинов/паролей для рассылки пользователям с помощью этого скрипта
```bash
#!/bin/bash

# Assign file descriptors to users and passwords files
exec 3< usernames.lst
exec 4< passwords.lst

# Read user and password
while read iusername <&3 && read ipasswd <&4 ; do
  # Just print this for debugging
  #printf "\tCreating user: %s and password: %s\n" $iusername $ipasswd
  echo $iusername,$ipasswd >> userpasswd.lst
done
```

6. Меняем пароли на сервере источнике с помощью скрипта
```bash
#!/bin/bash

# set X-API-Key w write permissions
XAPIKEYW=bac72807-f4b9-408f-838d-71d81c59e8d2
# set hostname
XHOSTNAME=mx.mailcow.com

# set IFS
IFS=$','

# Read user and password
while read -r iusername ipasswd; do

# debug only
# printf "\tCreating user: %s and password: %s\n" $iusername $ipasswd

curl --header "Content-Type: application/json" \
  --header "X-API-Key: $XAPIKEYW" \
  --request POST \
  --data '{"attr": {"password":"'"$ipasswd"'","password2":"'"$ipasswd"'"}, "items": "'"$iusername"'"}' \
  http://$XHOSTNAME/api/v1/edit/mailbox

done < userpasswd.lst
```

7. Снимаем резервную копию исходного сервера, исключаем из неё vmail
```bash
MAILCOW_BACKUP_LOCATION=/mnt/nasx/mailcow THREADS=8 ./helper-scripts/backup_and_restore.sh backup crypt redis rspamd postfix mysql
```

## Перенос данных на сервер получатель

1. Восстанавливаем резервную копию на сервере получателе
```bash
MAILCOW_BACKUP_LOCATION=/mnt/nasx/mailcow ./helper-scripts/backup_and_restore.sh restore
```

2. Пересчитываем параметры квоты для пользователей на сервере получателе
```bash
docker compose exec dovecot-mailcow doveadm quota recalc -A
```

3. Запускаем imapsync для переноса данных почты скриптом
```bash
#!/bin/sh
while IFS=',' read iusername ipasswd 
  do 
    ./imapsync --host1 "mx.mailcow.com" --user1 "$iusername" --password1 "$ipasswd" \
               --host2 "mz.mailcow.com" --user2 "$iusername" --password2 "$ipasswd" \
               --ssl1 --ssl2 --automap --delete2duplicates --skipcrossduplicates
done < userpasswd.lst
```
Если сервер получатель не имеет корректного SSL сертификата, необходимо заменить --ssl2 на --nossl2

### Установка imapsync

* ставим зависимости
```bash
apt install \
 libauthen-ntlm-perl \
 libcgi-pm-perl \
 libcrypt-openssl-rsa-perl \
 libdata-uniqid-perl \
 libencode-imaputf7-perl \
 libfile-copy-recursive-perl \
 libfile-tail-perl \
 libio-socket-inet6-perl \
 libio-socket-ssl-perl \
 libio-tee-perl \
 libhtml-parser-perl \
 libjson-webtoken-perl \
 libmail-imapclient-perl \
 libparse-recdescent-perl \
 libproc-processtable-perl \
 libmodule-scandeps-perl \
 libreadonly-perl \
 libregexp-common-perl \
 libsys-meminfo-perl \
 libterm-readkey-perl \
 libtest-mockobject-perl \
 libtest-pod-perl \
 libunicode-string-perl \
 liburi-perl \
 libwww-perl \
 libtest-nowarnings-perl \
 libtest-deep-perl \
 libtest-warn-perl \
 make \
 time \
 cpanminus
```

* получаем скрипт
```
wget -N https://raw.githubusercontent.com/imapsync/imapsync/master/imapsync
```

* делаем его исполнимым
```
chmod +x imapsync
```
