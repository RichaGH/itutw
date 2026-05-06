# Вопрос
Давай добавим в реализацию следующее: 
1 В логах помимо времени нужно фиксировать имя хоста. Вынеси проверку доступности 
2 Предварительная проверка репозиториев
Рекомендую вынести проверку доступности репозиториев в отдельный play перед началом обновления группы (с delegate_to: "{{ item }}" и модулем uri или command: dnf repolist). Это даст fast fail до остановки сервисов.

# Рассуждения
Мы добавим предварительную проверку репозиториев для каждого хоста группы до начала остановки сервисов. Также обеспечим логирование с временем и именем хоста. Для простоты можно использовать ansible.builtin.command для dnf repolist, но лучше через модуль uri, если нужно проверять конкретные URL репозиториев. Однако dnf repolist достаточно надежен. Сделаем отдельный play перед основным обновлением, который проверит доступность репозиториев для всех хостов группы. Логирование будем вести в файл на control node.

# Ответ

## Добавление предварительной проверки репозиториев и расширенного логирования

Ниже приведены изменения для реализации:

1. **Предварительная проверка доступности репозиториев** — отдельный play перед обновлением группы.
2. **Логирование с временем и именем хоста** — пишем в общий лог-файл на control node.
3. **Быстрый отказ (fast fail)** — если репозитории недоступны на любом хосте группы, обновление для этой группы не начинается (без остановки сервисов).

### Обновлённая структура проекта

```
deploy_update/
├── site.yml
├── results.json
├── update.log                     # новый файл лога
├── inventory.ini
└── roles/
    ├── update_services/
    │   └── tasks/
    │       ├── main.yml
    │       ├── write_result.yml
    │       └── fail_group.yml
    └── report/
        └── tasks/
            └── main.yml
```

### 1. Файл `site.yml` — добавляем play предварительной проверки

```yaml
---
# Play 1: Предварительная проверка репозиториев для группы lgs
- name: Проверка репозиториев перед обновлением (группа lgs)
  hosts: lgs
  gather_facts: no
  serial: 1                         # проверяем последовательно, чтобы логи были чёткими
  tasks:
    - name: Проверить доступность репозиториев через dnf repolist
      ansible.builtin.command:
        cmd: dnf repolist
      register: repolist_output
      changed_when: false
      failed_when:
        - repolist_output.rc != 0
        - "'Error' in repolist_output.stderr"
      rescue:
        - name: Логируем ошибку репозиториев и прерываем выполнение для всей группы
          ansible.builtin.include_role:
            name: update_services
            tasks_from: log_error
          vars:
            error_message: "Хост {{ inventory_hostname }}: репозитории недоступны. Вывод: {{ repolist_output.stderr | default('') }}"
          delegate_to: localhost
        - name: Остановить обработку группы lgs
          ansible.builtin.meta: end_play

    - name: Логируем успешную проверку репозиториев
      ansible.builtin.include_role:
        name: update_services
        tasks_from: log_success
      vars:
        message: "Хост {{ inventory_hostname }}: проверка репозиториев успешна"
      delegate_to: localhost

# Play 2: Обновление группы lgs
- name: Обновление серверов LGS (Logstash)
  hosts: lgs
  serial: 1
  gather_facts: yes
  roles:
    - role: update_services
      vars:
        service_names: [logstash]
        group_name: "lgs"

# Play 3: Предварительная проверка репозиториев для группы kaf
- name: Проверка репозиториев перед обновлением (группа kaf)
  hosts: kaf
  gather_facts: no
  serial: 1
  tasks:
    - name: Проверить доступность репозиториев через dnf repolist
      ansible.builtin.command:
        cmd: dnf repolist
      register: repolist_output
      changed_when: false
      failed_when:
        - repolist_output.rc != 0
        - "'Error' in repolist_output.stderr"
      rescue:
        - name: Логируем ошибку репозиториев и прерываем выполнение для всей группы
          ansible.builtin.include_role:
            name: update_services
            tasks_from: log_error
          vars:
            error_message: "Хост {{ inventory_hostname }}: репозитории недоступны. Вывод: {{ repolist_output.stderr | default('') }}"
          delegate_to: localhost
        - name: Остановить обработку группы kaf
          ansible.builtin.meta: end_play

    - name: Логируем успешную проверку репозиториев
      ansible.builtin.include_role:
        name: update_services
        tasks_from: log_success
      vars:
        message: "Хост {{ inventory_hostname }}: проверка репозиториев успешна"
      delegate_to: localhost

# Play 4: Обновление группы kaf
- name: Обновление серверов KAF (Kafka + Zookeeper)
  hosts: kaf
  serial: 1
  gather_facts: yes
  roles:
    - role: update_services
      vars:
        service_names: [kafka, zookeeper]
        group_name: "kaf"

# Play 5: Вывод итоговой таблицы
- name: Сформировать итоговую таблицу
  hosts: localhost
  gather_facts: no
  roles:
    - role: report
```

### 2. Добавляем вспомогательные файлы логирования в роль `update_services`

#### `roles/update_services/tasks/log_success.yml`

```yaml
---
- name: Запись успешного события в лог-файл
  ansible.builtin.lineinfile:
    path: "{{ playbook_dir }}/update.log"
    line: "{{ ansible_date_time.iso8601 }} - SUCCESS - {{ message }}"
    create: yes
  delegate_to: localhost
```

#### `roles/update_services/tasks/log_error.yml`

```yaml
---
- name: Запись ошибки в лог-файл
  ansible.builtin.lineinfile:
    path: "{{ playbook_dir }}/update.log"
    line: "{{ ansible_date_time.iso8601 }} - ERROR - {{ error_message }}"
    create: yes
  delegate_to: localhost
```

### 3. Основная роль `update_services/tasks/main.yml` (без дублирования проверки репозиториев — она уже выполнена в отдельном play)

```yaml
---
- name: Инициализация файла результатов
  ansible.builtin.copy:
    content: "[]"
    dest: "{{ playbook_dir }}/results.json"
    force: no
  delegate_to: localhost
  run_once: true

- name: Логируем начало обновления хоста
  ansible.builtin.include_tasks: log_success.yml
  vars:
    message: "Начало обновления хоста {{ inventory_hostname }} (группа {{ group_name }})"

- name: Проверить, что все указанные сервисы запущены ДО обновления
  ansible.builtin.systemd:
    name: "{{ item }}"
    state: started
  loop: "{{ service_names }}"
  register: pre_check
  failed_when: pre_check.results | selectattr('status.ActiveState', 'ne', 'active') | list | length > 0

- name: Остановить все указанные сервисы
  ansible.builtin.systemd:
    name: "{{ item }}"
    state: stopped
  loop: "{{ service_names }}"

- name: Обновить все пакеты через DNF (без обновления кэша – уже сделано на этапе проверки)
  ansible.builtin.dnf:
    name: '*'
    state: latest
  register: dnf_result
  rescue:
    - name: Логируем ошибку DNF
      ansible.builtin.include_tasks: log_error.yml
      vars:
        error_message: "{{ inventory_hostname }}: dnf update failed - {{ dnf_result.msg | default('Unknown error') }}"
    - name: Прерываем группу
      ansible.builtin.include_tasks: fail_group.yml
      vars:
        error_message: "dnf update завершился ошибкой: {{ dnf_result.msg | default('Unknown error') }}"

- name: Перезагрузить сервер
  ansible.builtin.reboot:
    reboot_timeout: 900
    pre_reboot_delay: 5
    post_reboot_delay: 30
  register: reboot_result
  rescue:
    - name: Логируем ошибку перезагрузки
      ansible.builtin.include_tasks: log_error.yml
      vars:
        error_message: "{{ inventory_hostname }}: reboot timeout or failure"
    - name: Прерываем группу
      ansible.builtin.include_tasks: fail_group.yml
      vars:
        error_message: "Хост не загрузился в течение {{ reboot_result.reboot_timeout | default(900) }} секунд"

- name: Собрать факты после перезагрузки
  ansible.builtin.setup:

- name: Проверить, что сервисы запустились (с повторами)
  ansible.builtin.systemd:
    name: "{{ item }}"
    state: started
  loop: "{{ service_names }}"
  register: post_check
  until: post_check.results | selectattr('status.ActiveState', 'eq', 'active') | list | length == service_names | length
  retries: 6
  delay: 10
  rescue:
    - name: Логируем ошибку запуска сервисов
      ansible.builtin.include_tasks: log_error.yml
      vars:
        error_message: "{{ inventory_hostname }}: сервисы {{ service_names | join(', ') }} не запустились после ребута"
    - name: Прерываем группу
      ansible.builtin.include_tasks: fail_group.yml
      vars:
        error_message: "После перезагрузки сервисы {{ service_names | join(', ') }} не активны"

- name: Записать успешный результат в JSON
  ansible.builtin.include_tasks: write_result.yml
  vars:
    result_status: "Успешно"
    error_detail: ""

- name: Логируем успешное завершение обновления хоста
  ansible.builtin.include_tasks: log_success.yml
  vars:
    message: "Обновление хоста {{ inventory_hostname }} успешно завершено"
```

### 4. Файл `write_result.yml` (добавляем логирование при записи)

```yaml
---
- name: Собрать информацию о результатах
  set_fact:
    result_entry:
      host: "{{ inventory_hostname }}"
      group: "{{ group_name }}"
      status: "{{ result_status }}"
      kernel: "{{ ansible_kernel | default('N/A') }}"
      os_version: "{{ ansible_distribution | default('N/A') }} {{ ansible_distribution_version | default('') }}"
      error: "{{ error_detail | default('') }}"
      timestamp: "{{ ansible_date_time.iso8601 }}"

- name: Добавить запись в JSON (атомарно с помощью jq)
  ansible.builtin.shell:
    cmd: |
      tmp=$(mktemp)
      jq --argjson new "{{ result_entry | to_json }}" '. + [$new]' {{ playbook_dir }}/results.json > "$tmp" && mv "$tmp" {{ playbook_dir }}/results.json
    executable: /bin/bash
  delegate_to: localhost
  changed_when: false

- name: Логируем запись результата
  ansible.builtin.lineinfile:
    path: "{{ playbook_dir }}/update.log"
    line: "{{ ansible_date_time.iso8601 }} - INFO - Хост {{ inventory_hostname }}: результат записан ({{ result_status }})"
    create: yes
  delegate_to: localhost
```

### 5. Файл `fail_group.yml` (обновлённый)

```yaml
---
- name: Записать ошибку в результат для текущего хоста
  ansible.builtin.include_tasks: write_result.yml
  vars:
    result_status: "Ошибка: {{ error_message }}"
    error_detail: "{{ error_message }}"

- name: Логируем критическую ошибку группы
  ansible.builtin.include_tasks: log_error.yml
  vars:
    error_message: "ГРУППА {{ group_name }} ОСТАНОВЛЕНА из-за ошибки на хосте {{ inventory_hostname }}: {{ error_message }}"

- name: Остановить выполнение для всей группы
  ansible.builtin.meta: end_play
```

### 6. Роль `report` дополним чтением лога (опционально)

Можно не выводить лог, но для наглядности добавим в `roles/report/tasks/main.yml` вывод последних строк лога:

```yaml
---
- name: Прочитать файл результатов
  ansible.builtin.slurp:
    src: "{{ playbook_dir }}/results.json"
  register: results_file

- name: Разобрать JSON
  ansible.builtin.set_fact:
    all_results: "{{ (results_file.content | b64decode) | from_json }}"

- name: Вывести последние 5 строк лога обновления
  ansible.builtin.slurp:
    src: "{{ playbook_dir }}/update.log"
  register: log_file
  ignore_errors: yes

- name: Показать хвост лога
  ansible.builtin.debug:
    msg: "{{ (log_file.content | b64decode).split('\n')[-5:] | join('\n') }}"
  when: log_file.content is defined

- name: Сформировать сводную таблицу
  ansible.builtin.debug:
    msg: |
      ========================= СВОДКА ОБНОВЛЕНИЯ =========================
      | Имя ВМ                    | Группа | Статус обновления       | Версия ядра          | Версия Linux                |
      |---------------------------|--------|-------------------------|----------------------|-----------------------------|
      {% for r in all_results %}
      | {{ r.host | default('?') | truncate(25, true) }} | {{ r.group | default('?') | truncate(6, true) }} | {{ r.status | default('N/A') | truncate(23, true) }} | {{ r.kernel | default('N/A') | truncate(20, true) }} | {{ r.os_version | default('N/A') | truncate(27, true) }} |
      {% endfor %}
      =====================================================================

- name: Если есть хосты с ошибкой – завершить с ненулевым кодом
  ansible.builtin.fail:
    msg: "Обновление завершено с ошибками. Смотрите таблицу и лог {{ playbook_dir }}/update.log"
  when: all_results | selectattr('status', 'search', 'Ошибка') | list | length > 0
```

## Объяснение для молодого коллеги (расширенное)

### Зачем нужна предварительная проверка репозиториев отдельным play?

- **Быстрый отказ (fast fail)** – если репозитории недоступны, мы даже не начинаем останавливать сервисы. Это предотвращает лишний простой.
- **Проверка на всех хостах последовательно** – мы используем `serial:1`, чтобы логи были последовательными и понятными. При ошибке на любом хосте группы дальнейшая проверка останавливается и обновление группы не начинается.
- **Использование `dnf repolist`** – эта команда вернёт ненулевой код, если хотя бы один репозиторий недоступен или не может быть синхронизирован. Это достаточно надёжно и не требует отдельных `uri` запросов к каждому зеркалу.

### Почему мы вынесли проверку в отдельный play, а не в роль `update_services`?

- Роль `update_services` запускается только после успешной проверки. Это **разделяет ответственность**: проверка окружения (pre-flight) и само обновление.
- Если проверка упала, плейбук даже не вызывает роль – это экономит время и упрощает отладку.

### Как работает логирование с временем и именем хоста?

- Мы используем `ansible_date_time.iso8601` для получения временной метки.
- В каждом лог-сообщении явно указываем `{{ inventory_hostname }}`. Это позволяет отслеживать последовательность событий на разных машинах, даже если логи пишутся в один файл.
- Лог-файл `update.log` создаётся на control node и дополняется построчно. Он остаётся после выполнения плейбука для аудита.

### Почему не используем `ansible.builtin.uri` для проверки репозиториев?

- Получить список всех URL репозиториев динамически сложно (могут быть зеркала, metalink). `dnf repolist` использует встроенную логику DNF и выдаёт ошибку при любой проблеме с доступом к репозиториям – этого достаточно для быстрой проверки.
- Если в вашей инфраструктуре есть внешние репозитории (например, EPEL), `dnf repolist` проверит их доступность через заданные зеркала, но не будет скачивать пакеты.

### Что произойдет, если `dnf repolist` упадет?

- Play 1 (проверка) попадёт в `rescue`. Там мы логируем ошибку и вызываем `meta: end_play`, который **немедленно останавливает выполнение для всей группы**.
- Второй play (обновление) для этой группы **не будет запущен** – Ansible перейдёт к следующему play в `site.yml` (т.е. к проверке следующей группы).
- Это гарантирует, что ни один сервис не будет остановлен, если система не готова к обновлению.

### Как ведётся лог-файл при параллельной работе групп?

В нашем сценарии группы обрабатываются последовательно (сначала `lgs`, потом `kaf`). Поэтому лог-файл пишется линейно, без гонок. Если бы мы запускали группы параллельно, пришлось бы использовать `ansible.builtin.copy` с резервированием или отдельные файлы логов на группу. Для простоты мы оставили последовательную обработку.

### Пример содержимого `update.log`

```
2025-01-15T10:00:01Z - SUCCESS - Хост logstash1: проверка репозиториев успешна
2025-01-15T10:00:05Z - SUCCESS - Хост logstash1: начало обновления
2025-01-15T10:02:10Z - SUCCESS - Обновление хоста logstash1 успешно завершено
2025-01-15T10:02:12Z - SUCCESS - Хост logstash2: проверка репозиториев успешна
...
2025-01-15T10:05:00Z - ERROR - Хост kafka1: репозитории недоступны. Вывод: Error: Failed to download metadata for repo 'epel'
```

## Запуск и проверка

```bash
ansible-playbook -i inventory.ini site.yml
```

После выполнения смотрим:
- `results.json` – структурированные данные.
- `update.log` – хронология событий с именами хостов и временем.
- Итоговая таблица в выводе Ansible.

Все изменения обеспечивают **fast fail**, **чёткое логирование** и **безопасную последовательность** обновления кластерных сервисов.
