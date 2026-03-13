# Назначение
Локальный [nextcloud](https://nextcloud.com/)-сервер с подключенным текстовым редактором и инстегрированным в него LLM-обработчиком. 
## Особенности:
* Требования к аппаратному обеспечению -- пониженные. Тестирование выполнялось на ПК с
  * Ubuntu 24.10
  * 32 Gb RAM
  * Intel(R) Core(TM) i7-2600
  * GTX 1070 8Gb / Driver Version: 570.211.01

## Проверено при nextcloud:33

# Порядок настройки
## LLM Server
Используем проект llama.cpp. 
Плюсы:
* В процессе компиляции возможна наиболее точная подстройка под особенности аппаратного обеспечения, что обеспечивает максимальную скорость инференса
* Возможность запуска моделей с функционалом vision без GUI интерфейса

Минус:
* Постоянно занимает VRAM под модель + контест
  
### Установка llama.cpp
```bash
cd ~
sudo apt update
sudo apt install build-essential git cmake ccache
git clone https://github.com/danil1online/nextcloud_onlyoffice_llm.git nextcloud
cd nextcloud
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-11-8 # Стабильнее всего работает с GTX | Pascal
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
# Включаем возможность использования видеокарты Nvidia Pascal
cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON -DCMAKE_CUDA_ARCHITECTURES=61 -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON
cmake --build build --config Release -j$(nproc)
```

### Выбор модели
Анализ по состоянию на март 2026 г. показал, что с русским языком качественно работают модели Qwen3.5. Кроме того, они имеют возможность обработки изображений в текстовых документах и распознавания текста.

Учитывая 
* особенности аппаратной конфигурации,
* возможный объем обрабатываемого документа (т.е. нужен запас в VRAM под контекст)

выбрана модель из [Qwen3.5-4B-GGUF](https://huggingface.co/unsloth/Qwen3.5-4B-GGUF)

Для обработки
* текста -- Qwen3.5-4B-Q4_K_M.gguf;
* изображений -- mmproj-F16.gguf.

Модели размещены в том же каталоге ~/nextcloud/llama_cpp_models

### Проверка запуска сервера модели:
```bash
/home/user/nextcloud/llama.cpp/build/bin/llama-server -m /home/user/nextcloud/llama_cpp_models/Qwen3.5-4B-Q4_K_M.gguf --mmproj /home/user/nextcloud/llama_cpp_models/mmproj-F16.gguf -ngl 99 -c 16384 --host 0.0.0.0 --port 8080
```
здесь:
* /home/user/nextcloud/llama.cpp/build/bin/llama-server -- путь к основному исполняемому файлу
* /home/user/nextcloud/llama_cpp_models/Qwen3.5-4B-Q4_K_M.gguf -- путь к текстовой части модели
* /home/user/nextcloud/llama_cpp_models/mmproj-F16.gguf -- путь к visual части модели
* -ngl 99 -- выгрузка модели полностью на GPU
* -c 16384 -- размер контекста
* --host 0.0.0.0 -- запуск сервере для всей локальной сети
* --port 8080 -- порт для доступа

После запуска:
```bash
watch nvidia-smi
```
показывает, что llama.cpp-server занял 7371MiB из 8192MiB доступных; остается место под доп. контекст для изображений. 

### Автоматизация запуска сервера модели:
Через скрипт
```bash
sudo nano /etc/systemd/system/llama-cpp.service
```
Вносим:
```sh
[Unit]
Description=Llama.cpp Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/user/llama.cpp
ExecStart=/home/user/nextcloud/llama.cpp/build/bin/llama-server -m /home/user/nextcloud/llama_cpp_models/Qwen3.5-4B-Q4_K_M.gguf --mmproj /home/user/nextcloud/llama_cpp_models/mmproj-F16.gguf -ngl 99 -c 16384 --host 0.0.0.0 --port 8080
Restart=always

[Install]
WantedBy=default.target
```
Выставляем автозапуск и запускаем. 
```bash
sudo systemctl daemon-reload
sudo systemctl enable llama-cpp.service
sudo systemctl start llama-cpp.service
```

## Запуск nextcloud [docker-compose.yml](docker-compose.yml)
```bash
cd /home/user/nextcloud
docker compose up -d
```

## Настройка
1. Вход в браузере http://192.168.1.38:8181/, он сразу будет в trusted_domains
2. Задание логина / пароля admin. В качестве СУБД -- postgres:16
3. Если нужно добавить адреса для входа, их нужно добавить в trusted_domains. Выполнить в командной строке:
```bash
docker exec -it nextcloud bash
apt update
apt install nano
nano config/config.php
```
см., например:
```sh
  'trusted_domains' => 
  array (
    0 => '192.168.1.38:8181',
    1 => '192.168.2.52:8181',
  ),
...
  'overwrite.cli.url' => 'http://192.168.1.38:8181',
```
4. В брауезере перейти: значек пользователя в правом верхнем углу - "Приложения"
4.1. Слева внизу в списке выбрать "Офис и текст". Nextcloud сразу предложит установить переходник на OnlyOffice, соглашаемся.
4.2. Слева вверху "Активные приложения" -- отключаем PDF Viewer, тогда вместо него pdf-файлы будут открываться в OnlyOffice.
5. В брауезере перейти: значек пользователя в правом верхнем углу - "Параметры сервера". Слева внизу -- "ONLYOFFICE"
5.1. Адрес ONLYOFFICE Docs: http://192.168.1.38:8383/
5.2. Секретный ключ (оставьте пустым для отключения): my_super_secret_key_123
5.3. Дополнительные настройки сервера (раскрыть):
   5.3.1. Заголовок авторизации (оставьте пустым, чтобы использовать заголовок по умолчанию): Authorization
   5.3.2. Адрес ONLYOFFICE Docs для внутренних запросов сервера: http://documentserver/
   5.3.3. Адрес сервера для внутренних запросов ONLYOFFICE Docs: http://195.133.13.56:8181/
5.4. "Сохранить"

## Проверка
1. В браузере вверху слева "Файлы" - "Documents"
2. Нажимаем на "Welcome to Nextcloud Hub.docx", документ должен открыться в OnlyOffice
3. В списке панели инструментов выбираем AI - Настройки - "Редактировать ИИ-модели"
4. Выбираем среди возможных значений поля "Имя" - "LM Studio".
5. В поле URL указываем: http://192.168.1.38:8080/v1
6. Нажимаем "Обновить список моделей", должна автоматически определиться "LM Studio [Qwen3.5-4B-Q4_K_M.gguf]"
7. В поле "Использовать модель для" убираем "Изображения", "Обработка аудиоданных".
8. Сохраняем, пробуем, например, выделить текст в документе и нажать кнопку "Резюмировать".

## Дополнительные возможности

### Сервер AppAPI
1. В брауезере перейти: значек пользователя в правом верхнем углу - "Параметры сервера". Слева внизу -- "AppAPI"
2. "+ Register daemon"
Register a new deploy daemon
Имя: docker_socket_proxy
Протокол: http
Имя или адрес сервера: docker-socket-proxy:2375
Варианты развертывания
Сеть докеров: nextcloud_ai-net
Адрес сервера Nextcloud: http://nextcloud
Поддержка GPU: false
Вычислительное устройство: CPU
Лимит памяти: Неограничено
CPU limit: Неограничено

### Приложения для использования LLM из магазина приложений Nextcloud Hub
Возможно установить после конфигурации Сервер AppAPI
1. В меню: 
-> Приложения
-> AI
-> в поисковой строке "context" - должно найтись "Результаты из других категорий" - "Context Chat Backend"
 -> Развернуть и включить. 
Займет много времени, потому что будет качаться ghcr.io/nextcloud/context_chat_backend:5.3.0. 
Чтобы отслеживать реальный статус после запуска из gui, параллельно из командной строки можно запустить 
```bash
docker pull ghcr.io/nextcloud/context_chat_backend:5.3.0
```
-- будет отображать скачивание. 
После скачивания будет показано "0 % Инициализация" -- пора качать (именно в таком порядке, см https://apps.nextcloud.com/apps/context_chat_backend):
2. Ассистенты для поиска по всему Nextcloud:
-> Приложения 
-> AI:
Nextcloud Assistant Context Chat
Nextcloud Context Agent
Nextcloud Assistant
После установки может потребоваться
```bash
docker restart nc_app_context_chat_backend
```
Ручной запуск индексирования
```bash
docker exec -it nextcloud bash
root@ab4015541fdc:/var/www/html# php occ context_chat:scan admin
```
Пример вывода:
```sh
[admin] Scanned files/090302-ИСТб-о23.xlsx
[admin] Scanned files/Nextcloud Manual.pdf
[admin] Scanned files/Readme.md
[admin] Scanned files/Reasons to use Nextcloud.pdf
[admin] Scanned files/Templates credits.md
[admin] Scanned files/Documents/Example.md
[admin] Scanned files/Documents/Nextcloud flyer.pdf
[admin] Scanned files/Documents/Readme.md
[admin] Scanned files/Documents/Welcome to Nextcloud Hub.docx
[admin] Scanned files/Photos/Readme.md
[admin] Scanned files/Шаблоны/Business model canvas.ods
[admin] Scanned files/Шаблоны/Calendar.odt
[admin] Scanned files/Шаблоны/Certificate.odt
[admin] Scanned files/Шаблоны/Checklist.ods
[admin] Scanned files/Шаблоны/Diagram & table.ods
[admin] Scanned files/Шаблоны/Expense report.ods
[admin] Scanned files/Шаблоны/Invoice.ods
[admin] Scanned files/Шаблоны/Invoice.odt
[admin] Scanned files/Шаблоны/Letter.odt
[admin] Scanned files/Шаблоны/Meeting notes.md
[admin] Scanned files/Шаблоны/Menu.odt
[admin] Scanned files/Шаблоны/Mother's day.odt
[admin] Scanned files/Шаблоны/Party invitation.odt
[admin] Scanned files/Шаблоны/Photo book.odt
[admin] Scanned files/Шаблоны/Product plan.md
[admin] Scanned files/Шаблоны/Readme.md
[admin] Scanned files/Шаблоны/Report.odt
[admin] Scanned files/Шаблоны/Resume.odt
[admin] Scanned files/Шаблоны/Syllabus.odt
[admin] Scanned files/Шаблоны/Timesheet.ods
```
Следующая команда:
```bash
root@ab4015541fdc:/var/www/html# php occ context_chat:stats
```
Пример вывода:
```sh
ContextChat statistics:
The indexing is not complete yet.
Total eligible files: 30
Files in indexing queue: 1
New files in indexing queue (without updates): 1
Queued documents (without files):array (
)
Files successfully sent to backend: 0
Indexed documents: array (
  'files__default' => 29,
)
Actions in queue: 0
File system events in queue: 0
```
Мониторинг (параллельно):
```bash
docker logs -f nc_app_context_chat_backend
```
Если выдает ошибку про время (176:00) -- удалить файл "files/Шаблоны/Timesheet.ods"

3. Приложения - Поиск - По данным из:

https://apps.nextcloud.com/apps/llm2
https://apps.nextcloud.com/apps/integration_openai

# License

This project is licensed under the [CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/) license. See the [LICENSE](./LICENSE) file for details.
