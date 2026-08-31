# Динамический анализ Android-приложений

- [Prerequisites](#Prerequisites)
- [Теория](#Теория)
- [Практика](#Практика)

## Prerequisites

К началу занятия должны быть готовы:

- Android Emulator: **Pixel 4, Android 14 (API 34), x86_64**;
- root-доступ в учебной среде;
- Python 3;
- **Frida 16.5.9**;
- **frida-tools 13.0.0**;
- Objection.

> Создание эмулятора и базовая работа с ADB рассматриваются до этого занятия и здесь не повторяются.

### Установка в Linux shell

```bash
sudo apt update
sudo apt install python3 python3-venv python3-pip -y

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install frida==16.5.9 frida-tools==13.0.0 objection
```

Проверка:

```bash
python3 --version
frida --version
objection --version
```

Ожидаемая версия Frida:

```text
16.5.9
```

### Установка в PowerShell

Проверяем наличие Python:

```powershell
python --version
```

Создаём виртуальное окружение:

```powershell
python -m venv .venv
```

Активируем его:

```powershell
.\.venv\Scripts\Activate.ps1
```

Обновляем `pip` и устанавливаем инструменты:

```powershell
python -m pip install --upgrade pip
pip install frida==16.5.9 frida-tools==13.0.0 objection
```

Проверка:

```powershell
python --version
frida --version
objection --version
```

Ожидаемая версия Frida:

```text
16.5.9
```

> Если PowerShell блокирует запуск `Activate.ps1`, для текущего процесса можно временно разрешить выполнение скриптов:
>
> ```powershell
> Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
> ```

### Версия Frida Server

На Android должен использоваться **frida-server 16.5.9**.

Для эмулятора из занятия:

```text
Android 14
API 34
x86_64
```

нужна сборка:

```text
frida-server-16.5.9-android-x86_64
```

Скачать:

https://github.com/frida/frida/releases/tag/16.5.9


## Теория

---

### 1. Magisk

**Magisk** — набор инструментов для модификации Android.

В рамках занятия важны три компонента:

- **MagiskSU** — получение root-привилегий;
- **Zygisk** — запуск модулей внутри процессов Android;
- **DenyList** — исключение выбранных приложений из воздействия Magisk.

Root нужен исследователю для более полного доступа к системе и запуска инструментов динамического анализа.

---

### 2. Frida

**Frida** — инструмент динамической инструментации.

Frida позволяет внедрять JavaScript-код в процесс приложения и во время его работы:

- перехватывать вызовы функций;
- смотреть и изменять аргументы;
- изменять возвращаемые значения;
- читать и изменять память;
- напрямую вызывать методы приложения.

Для Android обычно используются:

- **frida-tools** — клиент на компьютере;
- **frida-server** — сервер, запущенный на Android-устройстве.

Frida Server:  
https://github.com/frida/frida/releases/tag/16.5.9

---

### 3. Структура Frida-скрипта

Базовая логика скрипта:

1. Инициализировать работу с Java:

```javascript
Java.perform(function () {
    // ...
});
```

2. Найти нужный класс:

```javascript
const Target = Java.use("package.ClassName");
```

Для native-кода вместо этого можно искать функцию через `Module.findExportByName`.

3. Переопределить интересующий метод:

```javascript
Target.method.implementation = function (value) {
    console.log("value = " + value);

    const result = this.method(value);

    console.log("result = " + result);

    return result;
};
```

Таким образом можно наблюдать поведение приложения, не изменяя APK на диске.

---

### 4. Frida CodeShare

Готовые Frida-скрипты можно искать в:

https://codeshare.frida.re/

Запуск скрипта из CodeShare:

```bash
frida --codeshare автор/название -f YOUR_BINARY
```

Готовый скрипт следует рассматривать как сторонний код и проверять перед запуском.

---

### 5. Objection

**Objection** — инструмент динамической инструментации, построенный поверх Frida.

Установка:

```bash
pip3 install objection
```

Главное отличие:

| Frida | Objection |
|---|---|
| Пишем собственные скрипты | Можно использовать готовые возможности |
| Полный контроль над hook | Быстрее для типовых задач |
| Кастомная логика | Типовые bypass и исследование приложения |
| Удобна для глубокого анализа | Удобен для быстрого пентеста |

Frida и Objection не исключают друг друга: если возможностей Objection недостаточно, можно написать собственный Frida-скрипт.

---

### 6. Перехват трафика

Для анализа сетевого взаимодействия приложения можно использовать Burp Suite.

Общая схема:

```text
Android-приложение
        |
        v
      Proxy
        |
        v
   Burp Suite
        |
        v
      Сервер
```

Для HTTPS необходимо, чтобы тестовое устройство доверяло сертификату Burp.

Последовательность:

1. Получить сертификат из Burp.
2. Перевести его в подходящий для Android формат.
3. Добавить сертификат в trust store тестового устройства.
4. Настроить Wi-Fi на использование proxy.
5. При необходимости перенаправить трафик с устройства на хост.
6. Проверить запросы в Burp.

---

### 7. Xposed / LSPosed

**Xposed/LSPosed** позволяют изменять поведение Android-приложений во время выполнения через hook механизм.

В отличие от разового Frida-скрипта, LSPosed может использовать модули, которые автоматически применяются к выбранным приложениям.

---

### 8. Как приложение может обнаруживать анализ

#### Root / Magisk

Приложение может проверять:

- наличие `su` в `PATH`;
- наличие Magisk;
- признаки изменённого `boot.img`;
- возможность записи в системные каталоги;
- изменённые system properties.

#### LSPosed

Возможные проверки:

- наличие Xposed/LSPosed-классов в `ClassLoader`;
- анализ stack trace;
- проверки целостности поведения приложения.

#### Frida

Приложение может искать:

- процесс `frida-server`;
- открытые порты `27042` и `27043`;
- Frida-библиотеки в памяти;
- аномалии времени выполнения после hook.

#### Emulator

Можно проверять:

- свойства устройства;
- аппаратные особенности;
- сетевые отличия.

Такие проверки усложняют анализ, но сами по себе не должны быть единственным механизмом защиты критичной логики.

---

# Практика

## 1. Подготовленная среда

К началу занятия предполагается, что уже подготовлены:

- Android Emulator;
- настроенное подключение к устройству;
- возможность выполнять команды на устройстве;
- root-доступ в учебной среде.

Для практики используется заранее созданный эмулятор:

```text
Pixel 4
Android 14 (API 34)
ABI: x86_64
```

Отдельно создание эмулятора и работа с ADB в этой методичке не рассматриваются.
---

## 2. Установка Frida Tools

На компьютере:

```bash
pip install frida-tools
```

Проверяем установку:

```bash
frida --version
```

---

## 3. Запуск Frida Server

Используем **frida-server 16.5.9**, соответствующий установленной версии Frida на компьютере.

Для используемого на занятии эмулятора нужна сборка:

```text
android-x86_64
```

Скачать:

https://github.com/frida/frida/releases/tag/16.5.9

Переносим файл на устройство:

```bash
adb push frida-server /data/local/tmp/
```

Открываем shell:

```bash
adb shell
```

Переходим в каталог:

```bash
cd /data/local/tmp/
```

Выдаём права на запуск:

```bash
chmod 755 frida-server
```

Запускаем:

```bash
./frida-server
```

Терминал с `frida-server` оставляем запущенным.

---

## 4. Проверка подключения Frida

В новом терминале на компьютере:

```bash
frida-ps -U -a
```

Команда должна вывести запущенные приложения.

Находим интересующее приложение и его package name.

Пример из занятия:

```text
owasp.mstg.uncrackable1
```

---

## 5. Подключение к приложению

Если приложение уже запущено:

```bash
frida -U owasp.mstg.uncrackable1
```

Теперь Frida подключена к процессу приложения.

---

## 6. Запуск приложения со своим скриптом

Создаём файл:

```text
uncrackable1.js
```

Минимальный шаблон:

```javascript
Java.perform(function () {

    const Target = Java.use("package.ClassName");

    Target.method.implementation = function (value) {
        console.log("[+] method called");
        console.log("[+] value = " + value);

        const result = this.method(value);

        console.log("[+] result = " + result);

        return result;
    };

});
```

Перед запуском необходимо заменить:

```text
package.ClassName
method
```

на класс и метод, найденные при анализе приложения.

Запускаем:

```bash
frida -U --no-pause -l uncrackable1.js -f owasp.mstg.uncrackable1
```

После вызова нужного метода в терминале будут отображаться его аргументы и результат.

---

## 7. Изменение поведения метода

Допустим, метод возвращает `false`, из-за чего приложение не позволяет продолжить работу.

Вместо оригинального результата можно вернуть собственное значение:

```javascript
Java.perform(function () {

    const Target = Java.use("package.ClassName");

    Target.method.implementation = function () {
        console.log("[+] method intercepted");

        return true;
    };

});
```

После загрузки скрипта проверка будет получать `true`, хотя исходный APK не изменялся.

На практике таким способом можно показать принцип runtime bypass.

---

## 8. Использование готового Frida-скрипта

Открываем:

https://codeshare.frida.re/

Находим подходящий скрипт.

Запускаем:

```bash
frida --codeshare автор/название -f YOUR_BINARY
```

Сравниваем результат с собственным Frida-скриптом.

---

## 9. Objection

Устанавливаем:

```bash
pip3 install objection
```

На занятии Objection используется как пример инструмента, который позволяет выполнять типовые действия без самостоятельного написания Frida-скриптов.

Что сравнить:

```text
Frida
  -> сами определяем класс
  -> сами ставим hook
  -> сами меняем логику

Objection
  -> используем готовые возможности
  -> быстрее выполняем типовые действия
```

---

## 10. Перехват трафика

Для просмотра HTTP(S)-трафика приложения используем Burp Suite.

Общая схема:

```text
Android-приложение
        |
        v
   Burp Suite
        |
        v
      Сервер
```

### 10.1. Экспортируем сертификат Burp

В Burp открываем:

```text
Settings
-> Tools
-> Proxy
-> Import / export CA certificate
```

Выбираем:

```text
Export
-> Certificate in DER format
```

Сохраняем сертификат, например:

```text
burp.der
```

---

### 10.2. Переводим сертификат в PEM

#### Linux shell

```bash
openssl x509 -inform DER -in burp.der -out burp.pem
```

#### PowerShell

```powershell
openssl x509 -inform DER -in burp.der -out burp.pem
```

---

### 10.3. Устанавливаем сертификат на Android

Копируем сертификат на устройство:

#### Linux shell

```bash
adb push burp.pem /sdcard/Download/
```

#### PowerShell

```powershell
adb push .\burp.pem /sdcard/Download/
```

На эмуляторе открываем:

```text
Settings
-> Security
-> Encryption & credentials
-> Install a certificate
-> CA certificate
```

Выбираем:

```text
burp.pem
```

После установки сертификат Burp появится среди пользовательских доверенных сертификатов.

> Начиная с Android 7 приложения не обязаны доверять пользовательским CA. Поэтому браузер может успешно работать через Burp, а конкретное приложение — нет. В таком случае потребуется добавить сертификат в system trust store либо использовать bypass проверки через Frida/Objection.

---

### 10.4. Настраиваем Burp Proxy

В Burp проверяем Proxy Listener:

```text
Proxy
-> Proxy settings
-> Proxy listeners
```

Например:

```text
127.0.0.1:8080
```

Для эмулятора удобно сделать reverse:

```bash
adb reverse tcp:8080 tcp:8080
```

После этого задаём proxy на Android:

```bash
adb shell settings put global http_proxy 127.0.0.1:8080
```

Проверяем:

```bash
adb shell settings get global http_proxy
```

Ожидаемый результат:

```text
127.0.0.1:8080
```

---

### 10.5. Проверяем перехват

Запускаем приложение и выполняем действие, которое отправляет сетевой запрос.

В Burp открываем:

```text
Proxy
-> HTTP history
```

Если всё настроено правильно, увидим запросы приложения.

Можно посмотреть:

- URL;
- HTTP-метод;
- headers;
- cookies;
- параметры;
- request body;
- response body.

---

### 10.6. Если HTTPS не перехватывается

Если сертификат установлен, но приложение всё равно выдаёт TLS-ошибку, вероятно используется SSL Pinning.

Можно попробовать Objection:

```bash
objection -n <package> start
```

Внутри:

```text
android sslpinning disable
```

Если готовый bypass не работает, нужную проверку ищем в приложении и перехватываем через Frida.

---

### 10.7. После занятия

Отключаем proxy:

```bash
adb shell settings put global http_proxy :0
```

Удаляем reverse:

```bash
adb reverse --remove tcp:8080
```

---

## 11. Что должно получиться

После практики должна быть понятна последовательность:

```text
Android Emulator
       |
       v
 Frida Server
       |
       v
 Android process
       |
       +----> Frida hook
       |
       +----> Objection
       |
       +----> Burp Suite
```

Главная идея динамического анализа:

> Мы исследуем приложение непосредственно во время его выполнения и можем наблюдать или изменять его поведение без обязательного изменения исходного APK.

---

## Дополнительная практика

В качестве закрепления материала можно дополнительно пройти лабораторную работу **Mobile Hacking Lab — Secure Notes**:

https://academy.mobilehackinglab.com/course/lab-secure-notes

Задача лабораторной — исследовать Android-приложение и получить PIN-код, защищённый через `ContentProvider`.

Практика хорошо подходит для самостоятельного закрепления навыков анализа Android-приложений после занятия.

---

## Полезные ссылки

- Frida: https://frida.re/
- Frida Server: https://github.com/frida/frida/releases/tag/16.5.9
- Frida CodeShare: https://codeshare.frida.re/
- rootAVD: https://gitlab.com/newbit/rootAVD
- objection: https://github.com/sensepost/objection
- Настройка бурпа для перехвата трафика андроида: https://portswigger.net/burp/documentation/desktop/mobile/config-android-device

