
# Извлечение временных файлов из HFZ-iWatch с помощью статического и динамического анализа на macOS

---

## **1. Подготовка среды**
- **Используйте чистую виртуальную macOS** (Parallels, VMware, реальные Mac mini — зависит от того, что поддерживает HFZ-iWatch).
- Установите:
  - **IDA Pro** (статический анализ)
  - **lldb** (отладчик для macOS)
  - **Frida** (`pip3 install frida-tools`)
  - (опционально) **Hopper**, **Ghidra** для дополнительного RE.
  - Утилиты: `fs_usage`, `dtruss`, `vmmap`, `strings`, `xxd`.

---

## **2. Статический анализ в IDA Pro**

**Цель:** выяснить – где и как генерируются временные файлы, какие функции копируют/записывают данные.

- **Откройте HFZ-iWatch бинарник в IDA Pro.**
- **Выполните поиск строк:**
  - ■ `ibss`, `ibec`, `diag`
  - ■ Слова типа: `tmp`, `temp`, `write`, `open`, `memcpy`, `malloc`, `delete`, `unlink`
- **Проследите Xrefs (перекрёстные ссылки)** — где упоминание этих строк используется в коде (например, функции записи/отправки файлов на устройство).
- **Идентифицируйте интересные функции**, отвечающие за:
  - — Формирование содержимого файлов (логика генерации)
  - — Файловые операции (открытие, запись, удаление)
  - — Сетевые операции или передачу на USB
- **Посмотрите на антиотладочные проверки** (ищите вызовы к системам проверки отладчика, например, `ptrace`, `sysctl`, чтение /proc).

**Запишите/скриньте адреса функций** — пригодятся для постановки breakpoints и написания хук-скриптов!

---

## **3. Построение карты функций и точек для динамики**

**Зачем:** Нужны имена функций и адреса из IDA — они относятся к секциям кода в процессе HFZ-iWatch.

- **Отметьте:** Все потенциальные места генерации и передачи временных файлов.
- **Соберите сигнатуры файлов** (`magic bytes` для поиска в дампе, пример: IBSS-образ обычно начинается с определённой структуры).

---

## **4. Запуск HFZ-iWatch и динамический анализ**

### **A) LLDB — динамический отладчик**

1. **Запустите HFZ-iWatch из терминала, чтобы получить процесс:**
   ```bash
   ./HFZ-iWatch
   ```
2. **Откройте новый терминал, найдите PID процесса:**
   ```bash
   ps aux | grep HFZ-iWatch
   ```
3. **Прикрепитесь к процессу через lldb:**
   ```bash
   lldb -p <PID>
   ```
4. **Поставьте breakpoint на интересующую функцию (по адресу или имени — используйте info из IDA):**
   ```lldb
   breakpoint set --address <адрес_функции>
   ```
   Или:
   ```lldb
   breakpoint set --name <имя_функции>
   ```
5. **Дождитесь срабатывания breakpoint.**
6. **Исследуйте память:**
   - Посмотрите значения регистров (адреса буферов).
   - Используйте команду:
     ```lldb
     memory read --format x --size 1 --count <Сколько_байт_читать> <адрес>
     ```
   - Если адрес передаётся в функцию записи — снимите дамп буфера.
7. **Сохраните данные:**
   - Можно вывести в файл через перенаправление (или скриптом на Python).

#### **Если стоят антиотладочные проверки — ищите их в IDA (шаг 2) и временно патчите/обходите их, либо используйте Frida для обхода.**

---

### **B) Frida — динамическое подключение и перехват функций**

1. **Подключитесь к процессу:**
   ```bash
   frida-ps -U | grep HFZ
   frida -p <PID> -l your_script.js
   ```
2. **Напишите короткий скрипт (примерно):**
   ```javascript
   // Пример для перехвата write()
   Interceptor.attach(Module.findExportByName(null, "write"), {
      onEnter: function (args) {
        // args[1] — указатель на буфер, args[2] — длина
        var data = Memory.readByteArray(args[1], parseInt(args[2]));
        // Сохранить буфер (например, через отправку в файл/лог)
        send(data);
      }
   });
   ```
   - Также можно хукать свои интересные функции из IDA (по их именам или адресам).
3. **Данные будут отправляться из скрипта frida в вашу консоль/обработчик (файл).**

---

### **5. Альтернативный путь — дамп процесса**

- В нужный момент остановите процесс и выполните дамп памяти:
  ```bash
  sudo dtruss -p <PID> | grep write
  # или
  sudo fs_usage -w | grep HFZ
  ```
- Получите core dump с помощью того же lldb:
  ```lldb
  process save-core /tmp/hfz_watch.core
  ```
- Поиск интересных блоков данных:
  ```bash
  strings /tmp/hfz_watch.core | grep -Ei 'ibss|ibec|diag'
  ```
  Или более обще — анализируйте hex-дамп.

---

### **6. Советы по автоматизации**

- Пишите небольшие скрипты для поиска блоков данных по сигнатурам в дампах память.
- Можно автоматом выгружать нужные области/буферы по срабатыванию breakpoint/frida hook.

---

## **7. Важные замечания**

- **Возможно придётся несколько раз повторить кольцо статического + динамического анализа, чтобы найти оптимальные точки для снятия дампа.**
- Все действия выполняйте на копии системы/в тестовой среде.
- Всегда проверяйте законность своих действий в соответствии со страной.

---

**Если нужна помощь по написанию конкретных скриптов (например, Frida) или пояснения по работе с lldb — запросите!**
