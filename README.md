<p align="center">
  <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/8/87/Vagrant.png/800px-Vagrant.png" alt="Vagrant Logo" width="20%">
</p>

## ![Lesson](https://img.shields.io/badge/Lesson-otus__backup-0A84FF?style=for-the-badge&logo=linux&logoColor=white&labelColor=111827)![Author](https://img.shields.io/badge/Author-Kamil%20Ibragimov-10B981?style=for-the-badge&logo=github&logoColor=white&labelColor=111827)![Date](https://img.shields.io/badge/Date-15.12.2025-F59E0B?style=for-the-badge&logo=calendar&logoColor=white&labelColor=111827)

### 📌 Задание
1. Настроить стенд Vagrant (Server + Client).
2. Настроить удаленный бэкап `/etc` через BorgBackup.
3. Восстановить удаленный файл.

### ✅ Результат
- [x] Стенд развернут, SSH-ключи настроены.
- [x] Скрипт бэкапа работает по расписанию (5 мин).
- [x] Файл `/etc/fstab` успешно восстановлен. Доказательство см. на скриншоте 🖼️ ["restore_evidence.png"](restore_evidence.png)

### 🧭 Оглавление
- [🧰 Шаг 1 - Конфигурация стенда](#one)
- [🧰 Шаг 2 - Скрипт бэкапа](#two)
- [🧰 Шаг 3 - Тест восстановления](#three)

---

<a id="one"></a>
## 🧰 Шаг 1 - Конфигурация стенда

Использован Vagrant для поднятия двух машин на AlmaLinux 9.
* **Server (192.168.56.10):** Подключен доп. диск в `/var/backup`, создан пользователь `borg`.
* **Client (192.168.56.11):** Установлен клиент, настроен доступ по SSH-ключам.

Файл `Vagrantfile`:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_SERVER_URL'] = '[https://vagrant.elab.pro](https://vagrant.elab.pro)'

Vagrant.configure("2") do |config|
  config.vm.box = "almalinux/9"

  config.vm.define "server" do |box|
    box.vm.network "private_network", ip: "192.168.56.10"
    box.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      # Создание и подключение диска для бэкапов
      unless File.exist?("disk.vdi")
        vb.customize ['createhd', '--filename', 'disk.vdi', '--size', '2048']
      end
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', '1', '--device', '0', '--type', 'hdd', '--medium', 'disk.vdi']
    end
  end

  config.vm.define "client" do |box|
    box.vm.network "private_network", ip: "192.168.56.11"
    box.vm.provider "virtualbox" do |vb| 
        vb.memory = "2048"
    end
  end
end
```

<a id="two"></a>
🧰 Шаг 2 - Скрипт бэкапа
Скрипт /usr/local/bin/backup.sh на клиенте. Реализовано:

Логирование в syslog (тег borg-backup).

Политика очистки: храним 90 дней daily, 12 месяцев monthly.
```bash
#!/bin/bash

export BORG_REPO="borg@192.168.56.10:/var/backup/client"
export BORG_PASSPHRASE="123" 
LOG_TAG="borg-backup"

# Создание бэкапа
echo "Starting backup..." | logger -t $LOG_TAG
borg create --stats --list ::"etc-{now:%Y-%m-%d_%H%M}" /etc 2>&1 | logger -t $LOG_TAG

# Очистка старых копий
echo "Pruning old backups..." | logger -t $LOG_TAG
borg prune --keep-daily=90 --keep-monthly=12 -y 2>&1 | logger -t $LOG_TAG

echo "Backup finished." | logger -t $LOG_TAG
Добавлено в Cron: */5 * * * * root /usr/local/bin/backup.sh
```

<a id="three"></a>
🧰 Шаг 3 - Тест восстановления
Моделирование аварии: удаление конфигурационного файла /etc/fstab.
```bash
# 1. Удаление файла
sudo rm /etc/fstab

# 2. Просмотр списка архивов
borg list borg@192.168.56.10:/var/backup/client

# 3. Восстановление во временную папку
mkdir restore_temp
cd restore_temp
borg extract ::"etc-2025-12-15_1904" etc/fstab

# 4. Возврат файла
cp etc/fstab /etc/fstab
```
