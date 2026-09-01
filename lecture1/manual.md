# Занятие 1. Анализ безопасности Android-приложений

- [Prerequisites](#Prerequisites)
- [Теория](#-Теория)
- [Установка инструментов](#-Установка-инструментов)
- [Android Workflow](#-Android-Workflow)

## 🚀 Prerequisites

- JDK 8+ (для работы с apktool и jadx)
- [jadx/jadx-gui](https://github.com/skylot/jadx) - декомпилятор APK
- [apktool](https://ibotpeaches.github.io/Apktool/) - разбор APK (ресурсы, манифест, smali)
- [adb](https://developer.android.com/tools/adb) - Android Debug Bridge
- Android Studio + эмулятор Android с root-правами
- APK-файл для анализа (DIVA или другой)

---

## 📖 Теория

### Устройство системы Android

Android основан на ядре Linux. Из механизмов защиты присутствуют SELinux и seccomp. Каждое приложение запускается от своего пользователя в песочнице (sandbox), к которой не имеет доступа другое приложение на устройстве. Ранее использовалось Full Disk Encryption (FDE), затем перешли на File Based Encryption (FBE).

Файловая система Android:

```
data      - каталог для монтирования одноимённого раздела;
dev       - файлы устройств;
proc      - сюда монтируется procfs;
sys       - сюда монтируется sysfs;
system    - каталог для монтирования системного раздела;
bin, sbin - набор подсобных утилит и демонов (например, adbd);
res       - набор изображений для charger;
build.prop - системные настройки;
init      - система инициализации;
init.rc   - настройки системы инициализации;
ueventd.rc - настройки демона ueventd, входящего в состав init.
```

### Android-приложения (Java, Kotlin)

Почему Java:
- **Портабельность** - в отличие от iOS, существует множество разных устройств на базе Android (ТВ, смартфоны, разные процессоры). На Java достаточно написать код один раз, и он запустится на любой машине с JVM, если не используются специфичные низкоуровневые API ОС.
- **Надёжность** - отсутствует как класс уязвимости вида buffer overflow и use after free. Мобильное приложение просто останавливается, если возникает проблема.

Kotlin теперь главный язык для разработки приложений на Android.

#### Виртуальная машина Dalvik и ART

```
Java Source -> Java Byte Code -> Dalvik Byte Code -> Виртуальная машина Dalvik (DVM)
```

Почему не JVM? В угоду скорости работы программы. Далее перешли на ART, а после - на комбинированный режим JIT/AOT:
- **ART** - Android Runtime;
- **AOT** - Ahead-Of-Time compiler.

### Эмуляция Android OS

**Эмуляция** - процесс имитации поведения железа, инструкций процессора, операционной системы на другой архитектуре при помощи программного обеспечения.
**Симуляция** - имитация интерфейса оригинального ПО и его поведения.

В качестве эмулятора используется эмулятор из Android Studio. Альтернативы: Genymotion, LDPlayer.

Рекомендации по созданию эмулятора:
- Выбирать экран с большим размером дисплея для удобства работы (окно можно растягивать, но элементы интерфейса будут масштабироваться).
- В качестве **ABI** (Application Binary Interface) выбирать версию, совпадающую с архитектурой процессора хоста, иначе работа с эмулятором будет осложнена.
- Запуск возможен через интерфейс Android Studio или через CLI:

```bash
~/Library/Android/sdk/emulator/emulator -list-avds
~/Library/Android/sdk/emulator/emulator -avd <название эмулятора>

C:\Users\%USERNAME%\AppData\Local\Android\Sdk\emulator\emulator -list-avds
C:\Users\%USERNAME%\AppData\Local\Android\Sdk\emulator\emulator -avd <название эмулятора>
```

Документация: https://developer.android.com/studio/run/emulator-commandline

> В новой версии Android Studio эмулятор открывается во встроенном окне IDE. Чтобы отображать его в отдельном окне: `File > Settings > Tools > Emulator` (на macOS - `Android Studio > Preferences > Tools > Emulator`) и снять галочку **Launch in a tool window**.

### Root-привилегии и режим разработчика

#### Режим разработчика

Включается тапом 7 раз по пункту **Версия сборки** в меню настроек (обычно `Настройки -> О телефоне -> Информация о программном обеспечении`). После нескольких нажатий появится всплывающее окно, что мы почти у цели.

Режим разработчика позволит включить **Android Debug Bridge** - CLI для взаимодействия с устройством, который предоставляет следующие функции:
- Установка приложений
- Проброс/переадресация портов
- Загрузка/скачивание файлов
- Снятие скриншотов
- Запуск shell на устройстве

#### Root-привилегии

С привилегиями root мы получаем полный доступ к файловой системе: можем удалять системные приложения, кастомизировать оболочку. Root требуется для удобного исследования безопасности мобильных приложений - в частности, чтобы заглянуть в файлы, которые приложение генерирует во время работы, или использовать динамическую инструментацию (об этом - в следующих занятиях).

Получение root на реальном устройстве: **Magisk**, **SuperSU** (инструкции на 4pda/xdaforums.com).

---

## 📦 Установка инструментов

### jadx / jadx-gui

```bash
# Ubuntu/Debian
sudo apt install jadx jadx-gui

# Или через JAR
java -jar jadx-gui-1.5.6-all.jar   # GUI
jadx -d output_dir app.apk          # CLI - декомпиляция в Java
```

Установка на других ОС:
- **macOS:** `brew install jadx`
- **Windows/Linux:** скачать с https://github.com/skylot/jadx/releases
  - `with-jre` версия - если не установлено окружение для Java;
  - `no-jre` версия - если установлено окружение для Java.

### apktool

```bash
# apktool обычно в виде JAR
java -jar apktool.jar d app.apk -o app_decoded/   # декомпиляция
java -jar apktool.jar b app_decoded/ -o rebuilt.apk  # перекомпиляция
```

> **Важно:** после пересборки `apktool b` полученный APK **не подписан** и не установится. Его нужно переподписать.

### apksigner / uber-apk-signer (переподпись пересобранного APK)

При редактировании smali/ресурсов и обратной сборке APK теряет подпись. Переподписать можно следующим образом:

```bash
# uber-apk-signer (одна команда - zipalign + debug-подпись)
java -jar uber-apk-signer.jar -a rebuilt.apk
# выдаст rebuilt-aligned-debugSigned.apk, который сразу ставится
```

- Есть и другие варианты для подписи приложений, но для учебных целей удобнее **uber-apk-signer** - он сам генерирует debug-keystore, делает zipalign и подписывает.
- Без подписи `adb install` упадёт с `INSTALL_PARSE_FAILED_NO_CERTIFICATES`.

### ADB

```bash
sudo apt install adb
adb version          # проверить установку
adb start-server     # запустить ADB-сервер
```

---

## 🚀 Android Workflow

### 1. Создание эмулятора с root-правами

Открываем Android Studio -> Tools -> Device Manager -> Create Device.

При создании эмулятора выбираем систему с **Google APIs** (не Google Play) - на них проще получить root.

После создания эмулятора:

```bash
# Запуск эмулятора (бинарник emulator из $ANDROID_HOME/emulator/, не adb)
emulator -avd <name>

# Проверка подключения
adb devices

# Получение root-доступа (работает только на userdebug/AOSP-образах)
adb root
adb shell su          # если работает - root-права есть
```

> **Примечание:** Эмуляторы с Google Play получают root не всегда. Лучше использовать AOSP-образы (без Google-сервисов) - там `adb root` срабатывает «из коробки».

### 2. Работа с ADB

#### Основные команды

```bash
adb devices                        # список подключённых устройств
adb -s <название девайса> shell    # подключиться к конкретному устройству
adb shell                          # войти в shell эмулятора
adb shell ls /data/local/tmp       # проверить рабочую директорию
adb pull /data/local/tmp/file .    # скачать файл с устройства
adb push file /data/local/tmp/     # загрузить файл на устройство
adb logcat                         # просмотр логов в реальном времени
adb logcat -c                        # очистить логи
adb logcat -d > logs.txt             # сохранить логи в файл
adb install app.apk                  # установить APK
adb uninstall jakhar.aseem.diva      # удалить приложение
```

Попав в терминал, получаем права суперпользователя:

```bash
adb root
adb shell su
```

Также возможно подключаться к устройству по Wi-Fi при помощи adb, что полезно при работе с фермой устройств. Требование - находиться в одной Wi-Fi сети с устройством.

#### Песочница устройства

```bash
adb shell
cd /data/data/jakhar.aseem.diva         # директория приложения
ls                                      # посмотреть файлы
cat shared_prefs/diva_preferences.xml   # прочитать SharedPreferences
cat databases/sqli                      # прочитать SQLite-базу (или через sqlite3)
```

### 3. Получение APK

#### С устройства

```bash
adb shell pm list packages | grep <искомый пакет>   # найти пакет
adb shell pm path <имя пакета>                      # узнать путь к APK
adb pull <путь> <куда сохранить>                    # скачать APK
```

#### Из сторов

- https://github.com/Dynamic-Mobile-Security/mdast-cli
- Установить эмулятор с Google Play и скачать из официального стора.

### 4. Структура APK

APK - это обычный ZIP-архив:

```bash
unzip app.apk -d app_unzipped/
```

Внутри находится:

```
app.apk/
├── AndroidManifest.xml     # основная информация о приложении
├── classes.dex             # dalvik executable (байткод Dalvik)
├── resources.arsc          # таблица ресурсов: xml-описания всех ресурсов
├── res/                    # значки в нескольких разрешениях, layout в xml
├── lib/                    # нативные библиотеки (.so) на C/C++ для разных платформ
├── assets/                 # произвольные файлы, bundl'ящиеся в APK
└── META-INF/               # подписи APK
```

**META-INF** содержит:
- `CERT.RSA` - сертификат приложения;
- `CERT.SF` - контрольные суммы файлов ресурсов (картинок, звуков и т.д.);
- `MANIFEST.MF` - служебная информация, описывающая сам APK-файл.

Во время работы приложение генерирует:
- `shared_prefs/` - xml-файлы с форматом key-value;
- `databases/` - базы данных SQLite.

#### AndroidManifest.xml

Содержит:
- Имя пакета (`package="jakhar.aseem.diva"`)
- Все **активности** (Activities) - экраны приложения
- **ContentProvider** - экспортированные провайдеры данных
- **Разрешения** (permissions) - что может приложение

#### smali

Smali - ассемблероподобный язык Dalvik Bytecode. Декомпилируется через apktool:

```bash
java -jar apktool.jar d app.apk -o decoded/
# smali-файлы будут в decoded/smali/jakhar/aseem/diva/
```

### 5. Анализ приложения через jadx

jadx - декомпилятор smali обратно в читаемый Java-код:

```bash
# GUI-версия
java -jar jadx-gui-1.5.6-all.jar

# CLI - полная декомпиляция в директорию
jadx -d diva_source DivaApplication.apk

# Только исходный код (без ресурсов)
jadx -d diva_source --deobf DivaApplication.apk
```
