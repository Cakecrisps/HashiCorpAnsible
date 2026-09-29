# HashiCorp Vault Cluster с помощью Ansible

Ansible-проект для автоматизированного развёртывания **кластера HashiCorp Vault из 3 узлов** с использованием **Raft Storage**, TLS, автоматической инициализации и unseal, а также **Nginx в качестве TCP Load Balancer**.

Проект предназначен для воспроизводимого развёртывания небольшого отказоустойчивого кластера Vault с использованием Ansible Roles.

## Архитектура

```text
                         ┌─────────────────────┐
                         │      Nginx LB        │
                         │     nginx_lb         │
                         │       :8200          │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
             ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
             │   Vault 1   │ │   Vault 2   │ │   Vault 3   │
             │    Raft     │ │    Raft     │ │    Raft     │
             │   :8200     │ │   :8200     │ │   :8200     │
             │   :8201     │ │   :8201     │ │   :8201     │
             └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                    │               │               │
                    └───────────────┼───────────────┘
                                    │
                              Raft Consensus
```

### Компоненты

| Компонент           | Назначение                                       |
| ------------------- | ------------------------------------------------ |
| **HashiCorp Vault** | Хранилище и управление секретами                 |
| **Raft**            | Встроенное хранилище Vault и механизм консенсуса |
| **Ansible**         | Автоматизация развёртывания и конфигурации       |
| **Nginx**           | TCP Load Balancer перед Vault                    |
| **TLS**             | Шифрование соединений                            |
| `vault_node`        | Установка и настройка Vault                      |
| `vault_init`        | Инициализация и unseal кластера                  |
| `nginx_lb`          | Настройка Nginx в качестве Load Balancer         |

## Структура репозитория

```text
.
├── inventory.ini
├── playbook.yml
├── requriments.yml
└── roles/
    ├── vault_node/
    │   ├── tasks/
    │   ├── handlers/
    │   ├── templates/
    │   └── ...
    │
    ├── vault_init/
    │   ├── tasks/
    │   ├── handlers/
    │   └── ...
    │
    └── nginx_lb/
        ├── tasks/
        ├── handlers/
        ├── templates/
        └── ...
```

> **Примечание:** файл `requriments.yml` назван с опечаткой. Стандартное имя для файла зависимостей Ansible — `requirements.yml`.

## Требования

### Ansible Control Node

На машине, с которой запускается Ansible, должны быть установлены:

* Ansible
* Python
* SSH-клиент
* SSH-доступ ко всем целевым серверам
* Ansible Collections, указанные в `requriments.yml`

Установить Collections:

```bash
ansible-galaxy collection install -r requriments.yml
```

Используемые Collections:

```yaml
collections:
  - name: ansible.mysql
    version: ">=1.0.0"

  - name: community.zabbix
    version: ">=2.0.0"

  - name: community.crypto
    version: ">=2.0.0"

  - name: community.general
    version: ">=8.0.0"
```

### Целевая инфраструктура

Для текущей конфигурации предполагается:

* 3 Vault-ноды;
* 1 Nginx Load Balancer;
* SSH-доступ к серверам;
* возможность выполнения команд через `sudo`;
* сетевой доступ между Vault-нодами.

Используемые Vault-порты:

```text
8200/tcp  Vault API
8201/tcp  Vault Cluster Communication
```

## Inventory

Пример inventory:

```ini
[vaults]
Vault1 ansible_host=192.168.122.14
Vault2 ansible_host=192.168.122.176
Vault3 ansible_host=192.168.122.181

[nginx_lb]
NginxLB ansible_host=192.168.122.72
```

SSH-пользователь, приватный ключ и настройки privilege escalation задаются в `inventory.ini`.

### Безопасность

**Не следует хранить в Git:**

* приватные SSH-ключи;
* Vault unseal keys;
* Vault root token;
* TLS private keys;
* другие секреты.

Для production рекомендуется использовать Ansible Vault, внешний Secrets Manager или другой механизм безопасной передачи секретов.

## Как работает развёртывание

Основной playbook состоит из трёх этапов.

### 1. Развёртывание Vault

```yaml
- name: Create HashiCorp vault
  hosts: vaults
  become: true
  roles:
    - vault_node
```

Role `vault_node` выполняет:

1. Установку необходимых пакетов.
2. Добавление GPG-ключа HashiCorp.
3. Настройку официального APT-репозитория HashiCorp.
4. Установку Vault.
5. Создание директорий для данных и TLS.
6. Развёртывание TLS-сертификатов.
7. Генерацию `vault.hcl`.
8. Настройку capability `cap_ipc_lock`.
9. Включение и запуск systemd-сервиса Vault.

### 2. Инициализация и unseal

```yaml
- name: Vault cluster — init and unseal
  hosts: vaults
  become: true
  roles:
    - vault_init
```

Процесс инициализации:

```text
Проверка состояния Vault
        │
        ▼
Инициализация первой Vault-ноды
        │
        ▼
Сохранение результата инициализации
        │
        ▼
Unseal первой ноды
        │
        ▼
Проверка состояния Leader
        │
        ▼
Подключение остальных нод через Raft
        │
        ▼
Unseal остальных нод
```

Результат инициализации сохраняется на Ansible Control Node:

```text
vault-init.json
```

с правами:

```text
0600
```

> **Важно:** этот файл содержит чувствительную информацию, включая материалы, необходимые для unseal Vault. Его нельзя добавлять в Git.

Для операций с unseal keys используется `no_log: true`, чтобы ключи не попадали в вывод Ansible.

### 3. Настройка Nginx

```yaml
- name: NginxLb
  hosts: nginx_lb
  become: true
  roles:
    - nginx_lb
```

Role `nginx_lb`:

* устанавливает `nginx-full`;
* проверяет наличие модуля `stream`;
* разворачивает конфигурацию Nginx;
* проверяет конфигурацию через `nginx -t`;
* включает и запускает Nginx.

Nginx работает как **TCP proxy** перед Vault.

```text
Client
  │
  │ TCP :8200
  ▼
Nginx
  │
  ├──────────► Vault 1
  │
  ├──────────► Vault 2
  │
  └──────────► Vault 3
```

## Установка

Клонируем репозиторий:

```bash
git clone https://github.com/Cakecrisps/HashiCorpAnsible.git
cd HashiCorpAnsible
```

Устанавливаем Ansible Collections:

```bash
ansible-galaxy collection install -r requriments.yml
```

Проверяем доступность серверов:

```bash
ansible -i inventory.ini all -m ping
```

Запускаем развёртывание:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Для получения дополнительной информации:

```bash
ansible-playbook -i inventory.ini playbook.yml -v
```

Для детальной диагностики:

```bash
ansible-playbook -i inventory.ini playbook.yml -vvv
```

## Bootstrap Vault-кластера

Первая Vault-нода используется для создания первоначального кластера.

Перед выполнением `vault operator init` role проверяет, был ли Vault уже инициализирован. Это позволяет повторно запускать playbook без повторной инициализации существующего кластера.

Количество ключей и threshold задаются через переменные:

```yaml
vault_unseal_key_shares:
vault_unseal_key_threshold:
```

После инициализации необходимые unseal keys используются для снятия seal с Vault-нод.

### Порядок запуска

```text
                 ┌────────────────────┐
                 │ Vault не инициализирован │
                 └──────────┬─────────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │ vault operator   │
                  │       init       │
                  └────────┬─────────┘
                           │
                           ▼
                    Первый Leader
                           │
                           ▼
                         Unseal
                           │
                           ▼
                  Leader подтверждён
                           │
                           ▼
                  Followers Join Raft
                           │
                           ▼
                  Unseal Followers
                           │
                           ▼
                   Vault Cluster
```

Такой порядок предотвращает ситуацию, когда follower пытается подключиться к Leader до завершения его инициализации и unseal.

## TLS

Vault-ноды используют TLS для защищённого взаимодействия.

Основной конфигурационный файл Vault:

```text
/etc/vault.d/vault.hcl
```
На управляющей машине:

```bash
export VAULT_ADDR=https://192.168.122.72:8200
export VAULT_CACERT="$HOME/vault-lab-secrets/pki/ca.crt"
```

На другой клиентской машине укажите путь к копии этого `ca.crt`.
Для локальной проверки от root на Vault-ноде используйте
`VAULT_ADDR=https://127.0.0.1:8200` и `VAULT_CACERT=/opt/vault/tls/ca.crt`.
Сертификат Vault содержит IP-адреса нод и балансировщика в SAN; Nginx
передаёт TLS-соединение на Vault без его завершения.

TLS-файлы хранятся в отдельной директории с ограниченными правами доступа.

При создании сертификатов необходимо учитывать:

* DNS-имена Vault-нод;
* IP-адреса Vault-нод;
* имя Load Balancer;
* IP-адрес Load Balancer;
* корректный CA;
* SAN в сертификатах.

## Конфигурация

Основные параметры должны задаваться через Ansible variables.

В зависимости от используемой конфигурации могут использоваться:

```yaml
vault_data_dir:
vault_tls_dir:
vault_secrets_local_dir:
vault_unseal_key_shares:
vault_unseal_key_threshold:
vault_env:
```

Перед использованием проекта в новой инфраструктуре рекомендуется проверить `defaults`, `vars` и templates соответствующих ролей.

## Проверка после развёртывания

### Vault

Проверить systemd:

```bash
systemctl status vault
```

Проверить состояние Vault:

```bash
vault status
```

Или в JSON:

```bash
vault status -format=json
```

### Raft

Проверить участников кластера:

```bash
vault operator raft list-peers
```

Ожидаемый результат — три Vault-ноды, объединённые в один Raft-кластер.

### Nginx

Проверить сервис:

```bash
systemctl status nginx
```

Проверить конфигурацию:

```bash
nginx -t
```

## Диагностика

### Vault-нода не подключается к кластеру

Проверить сетевую доступность:

```bash
nc -vz <leader-ip> 8200
nc -vz <leader-ip> 8201
```

Проверить:

* Leader действительно unsealed;
* порт `8200` доступен;
* порт `8201` доступен;
* TLS-сертификаты корректны;
* CA соответствует сертификатам;
* `retry_join` указывает на правильную ноду.

### Vault находится в состоянии Sealed

Проверить:

```bash
vault status
```

Также проверить наличие файла:

```text
vault-init.json
```

> Не передавайте содержимое этого файла в issue, chat, логи или другие публичные системы.

### Nginx не запускается

Проверить конфигурацию:

```bash
nginx -t
```

Посмотреть логи:

```bash
journalctl -u nginx
```

Проверить наличие `stream` module.

### Vault не запускается

Проверить:

```bash
systemctl status vault
```

и:

```bash
journalctl -u vault
```

Дополнительно проверить:

```text
/etc/vault.d/vault.hcl
```

а также:

* пути к TLS-сертификатам;
* права на TLS-файлы;
* права на Vault data directory;
* `cap_ipc_lock`.

## Безопасность

Проект работает с критически важными секретами, поэтому необходимо уделить особое внимание безопасности Ansible Control Node.

### Никогда не коммитить

```text
vault-init.json
*.key
*.pem
*.crt
id_rsa
id_ed25519
Vault root token
Vault unseal keys
```

Если TLS-сертификаты используются только для тестовой инфраструктуры, они всё равно должны быть явно отделены от production credentials.

### `vault-init.json`

Файл:

```text
vault-init.json
```

содержит данные, полученные при инициализации Vault.

Рекомендуется:

* хранить его только на защищённом Control Node;
* ограничить права до `0600`;
* исключить его из Git;
* не передавать его через незащищённые каналы;
* не включать его содержимое в CI/CD logs.

### Production

Для production deployment рекомендуется рассмотреть:

* Auto Unseal через KMS/HSM;
* централизованное управление секретами;
* отдельное хранение unseal keys;
* сетевую сегментацию;
* firewall rules;
* TLS certificate rotation;
* мониторинг Vault;
* аудит;
* hardened SSH;
* Ansible Vault или внешний Secrets Manager;
* отказ от хранения credentials непосредственно в `inventory.ini`.

## Idempotency

Playbook рассчитан на повторный запуск.

Role `vault_init` проверяет текущее состояние Vault перед выполнением initialization, что позволяет избежать повторного запуска:

```bash
vault operator init
```

для уже инициализированного кластера.

Поэтому после изменения конфигурации можно повторно выполнить:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

и Ansible синхронизирует инфраструктуру с описанной конфигурацией.

## Проверка Ansible

Перед запуском playbook рекомендуется выполнить syntax check:

```bash
ansible-playbook \
  -i inventory.ini \
  playbook.yml \
  --syntax-check
```

Посмотреть структуру inventory:

```bash
ansible-inventory \
  -i inventory.ini \
  --graph
```

Выполнить check mode:

```bash
ansible-playbook \
  -i inventory.ini \
  playbook.yml \
  --check
```

Для разработки рекомендуется использовать отдельные тестовые VM, а не существующий production Vault-кластер.

## Известные моменты

### `requriments.yml`

Сейчас файл называется:

```text
requriments.yml
```

Рекомендуется переименовать его в стандартный:

```text
requirements.yml
```

После этого установка Collections будет выглядеть привычнее:

```bash
ansible-galaxy collection install -r requirements.yml
```

### `inventory.ini`

В текущем inventory присутствуют конкретные IP-адреса инфраструктуры и путь к SSH-ключу.

Перед публикацией репозитория стоит убедиться, что:

* IP-адреса не относятся к production-инфраструктуре;
* SSH private key не находится в репозитории;
* credentials не находятся в Git history.

## Лицензия

Лицензия проекта пока не указана.

При необходимости добавьте, например:

```text
LICENSE
```

и укажите выбранную лицензию в этом разделе.

## Полезные ссылки

* [HashiCorp Vault](https://developer.hashicorp.com/vault)
* [Vault Raft Storage](https://developer.hashicorp.com/vault/docs/concepts/storage/raft)
* [Ansible](https://docs.ansible.com/)
* [Ansible Collections](https://docs.ansible.com/ansible/latest/collections_guide/index.html)

## Автор

[github.com/Cakecrisps](https://github.com/Cakecrisps)

---

**Проект:** HashiCorp Vault Cluster + Ansible
**Репозиторий:** https://github.com/Cakecrisps/HashiCorpAnsible
