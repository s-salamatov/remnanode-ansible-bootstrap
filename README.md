# Shield Ansible Project

Комплект плейбуков и ролей для автоматической первичной настройки shield-хостов (пользователи, SSH, Tailscale, firewall, Alloy и т.д.). Контроллер хранит чувствительные данные и кэш только локально, чтобы репозиторий можно было публиковать.

## Требования
- Ansible 9+ (проверено на ansible-core 2.16 / ansible 12.2).
- Python 3.x на контроллере.
- Доступ по SSH к целевым хостам с параметрами, которые выдаёт провайдер (до первого запуска плейбука).

## Структура
- `inventory/hosts.yml` — основной инвентарь (формат описан в `inventory/README.md`).
- `group_vars/` — групповые переменные; `group_vars/all/vault.yml` содержит все секреты (Ansible Vault).
- `playbooks/` — точка входа `playbooks/shield-bootstrap.yml`.
- `roles/common/` — роль, выполняющая всё первичное конфигурирование.
- Локальное состояние контроллера (вне репозитория): `~/.ansible/common/`
  - `local_vars/` — кеш сгенерированных логинов/паролей по хостам.
  - `generated_keys/` — приватные/публичные SSH ключи, созданные для пользователей на хостах.
  - `.vault_password` — файл с паролем к Ansible Vault.

## Секреты и Vault
- Vault-файл: `group_vars/all/vault.yml` (зашифрован).
- Файл пароля: `~/.ansible/common/.vault_password` (используется автоматически из `ansible.cfg`).
- Просмотр/редактирование:
  - `ansible-vault view group_vars/all/vault.yml`
  - `ansible-vault edit group_vars/all/vault.yml`
- Добавляйте новые чувствительные значения как `vault_common_*` и ссылайтесь на них из `roles/common/defaults/main.yml` или соответствующих vars.

## Запуск
- Все хосты:  
  `ansible-playbook playbooks/shield-bootstrap.yml`
- Отдельный хост/группа:  
  `ansible-playbook playbooks/shield-bootstrap.yml -l <host_or_group>`
- Проверка без изменений:  
  `ansible-playbook playbooks/shield-bootstrap.yml -l <host> --check`

## Инвентарь
- Данные хостов в `all.hosts` + группировки по `project`, `environment`, `region`, `role`.
- Вынос общих переменных в `group_vars/<group>.yml`; не хранить секреты вне Vault.
- После изменений удобно проверять:  
  `ansible-inventory --list -i inventory/hosts.yml`

## SSH и пользователи
- Роль генерирует локально случайные логин/пароль и ключ (ed25519), создаёт пользователя, разворачивает ключ на хосте и:
  - сохраняет креды в `~/.ansible/common/local_vars/<host>.yml`;
  - копирует приватный ключ в `~/.ssh/<host>_<user>.key`;
  - обновляет `~/.ssh/config` для Host `<inventory_hostname>` (User, IdentityFile, Port, HostName) и добавляет комментарии.
- При повторных запусках использует уже сохранённые логин/пароль/ключ.

## Tailscale и Firewall
- Использует ключ из `vault_common_tailscale_authkey`.
- После настройки SSH-порта и Tailscale обновляет HostName/Port в локальном `~/.ssh/config`.
- UFW настраивается на кастомный SSH-порт и 443.

## Alloy
- Конфиг (`roles/common/templates/alloy/config.alloy.j2`) читает параметры из env vars, которые задаются в `env.conf.j2` через переменные `common_alloy_*` (берутся из Vault).
- Заполненность критичных переменных проверяется assert’ами в роли `alloy`.

## Добавление нового хоста
1. Добавьте запись в `inventory/hosts.yml` и включите её в нужные группы (`project_*`, `env_*`, `region_*`, `role_*`).
2. Убедитесь, что необходимые секреты есть в Vault.
3. При первом подключении используйте параметры от провайдера (добавьте вручную Host-блок в `~/.ssh/config`, если нужно).
4. Запустите: `ansible-playbook playbooks/shield-bootstrap.yml -l <host>`.
5. Проверьте обновлённый блок в `~/.ssh/config` и локальные файлы в `~/.ansible/common/`.

## Полезные команды
- Проверить SSH-блоки после плейбука: `grep -A4 "Host <host>" ~/.ssh/config`
- Проверить содержимое Vault: `ansible-vault view group_vars/all/vault.yml`
- Визуализировать инвентарь: `ansible-inventory --graph -i inventory/hosts.yml`

## Безопасность
- Не коммитить `~/.ansible/common/` и любые кеши/ключи (они вне репозитория).
- Если нужно сменить пароль Vault — обновите `~/.ansible/common/.vault_password` и перешифруйте `group_vars/all/vault.yml`.
- Регулярно чистите старые ключи/кеши, если хосты удалены.
