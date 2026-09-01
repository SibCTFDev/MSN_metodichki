# 🧪 Практика: Решение DIVA

DIVA - учебное приложение с 13 лабораторными работами по уязвимостям Android.

#### Список заданий

| # | Задание | Класс(ы) | Тип уязвимости |
|---|---|---|---|
| [d1](#пример-1-insecure-logging-d1) | Insecure Logging | `LogActivity` | Логирование CC в logcat |
| d2 | Hardcode Issues Part 1 | `HardcodeActivity` | Секрет захардкожен в Java |
| d3 | Insecure Data Storage Part 1 | `InsecureDataStorage1Activity` | SharedPreferences plaintext |
| d4 | Insecure Data Storage Part 2 | `InsecureDataStorage2Activity` | SQLite без шифрования + SQL-инъекция в INSERT |
| d5 | Insecure Data Storage Part 3 | `InsecureDataStorage3Activity` | Файл temp в dataDir |
| d6 | Insecure Data Storage Part 4 | `InsecureDataStorage4Activity` | Файл на external storage |
| d7 | SQL Injection | `SQLInjectionActivity` | SQL-инъекция через rawQuery |
| d8 | Input Validation Part 2 | `InputValidation2URISchemeActivity` | WebView + arbitrary URI |
| d9 | Access Control Part 1 | `AccessControl1Activity` -> `APICredsActivity` | Hardcoded API-креды через implicit intent |
| d10 | Access Control Part 2 | `APICreds2Activity` | PIN-флаг из intent extra (`check_pin`) контролируется вызывающим |
| d11 | Access Control Part 3 | `AccessControl3Activity` + `AccessControl3NotesActivity` + `NotesProvider` | PIN в plaintext + exported ContentProvider |
| d12 | Hardcode Issues Part 2 | `Hardcode2Activity` + `DivaJni` | Секрет в нативной библиотеке (.so) |
| d13 | Input Validation Part 3 | `InputValidation3Activity` + `DivaJni` | Проверка пароля в нативной библиотеке |

#### Пример 1: Insecure Logging (d1)

```java
// LogActivity
public void checkout(View view) {
    EditText cctxt = (EditText) findViewById(R.id.ccText);
    try {
        processCC(cctxt.getText().toString());
    } catch (RuntimeException e) {
        Log.e("diva-log", "Error while processing transaction with credit card: " + cctxt.getText().toString());
        Toast.makeText(this, "An error occured. Please try again later", 0).show();
    }
}
```

**Проблема:** Номер кредитной карты записывается в logcat Plaintext.

**Проверка:**

```bash
adb logcat -d | grep diva-log
# 08-10 09:16:51.642  2181  2181 E diva-log: Error while processing transaction with credit card: 1234
```

**Исправление:** Не логировать чувствительные данные, использовать маскированные значения:

```java
Log.e("diva-log", "Error while processing transaction with credit card: ****" + creditCardNumber.substring(12));
```


#### Пример 2: Hardcoding Issues (d2)

```java
// HardcodeActivity
public void access(View view) {
    EditText hckey = (EditText) findViewById(R.id.hcKey);
    if (hckey.getText().toString().equals("vendorsecretkey")) {
        Toast.makeText(this, "Access granted! See you on the other side :)", 0).show();
    } else {
        Toast.makeText(this, "Access denied! See you in hell :D", 0).show();
    }
}
```

**Проблема:** Секрет `vendorsecretkey` захардкожен в бинарнике.

Такой секрет нельзя считать защищенным. APK находится под полным контролем пользователя: его можно распаковать, декомпилировать и проанализировать. Поэтому строку можно обнаружить в коде или ресурсах приложения и использовать для обхода проверки.

**Исправление:** если секрет используется для проверки прав доступа или выполнения другой критичной бизнес-логики, такую проверку следует выполнять на доверенной стороне, например на сервере.

Приложение отправляет необходимые данные серверу, сервер самостоятельно выполняет проверку и возвращает результат.

Если приложению все же необходимо хранить чувствительные данные локально, следует использовать механизмы, предназначенные для их защиты. Например, криптографические ключи можно хранить с помощью Android Keystore. Однако локальное защищенное хранилище не превращает секрет внутри клиентского приложения в полностью недоступный для пользователя: поэтому критичные серверные секреты в APK хранить не следует.

#### Пример 3: Insecure Data Storage Part 1 (d3)

```java
// InsecureDataStorage1Activity
public void saveCredentials(View view) {
    SharedPreferences spref = PreferenceManager.getDefaultSharedPreferences(this);
    SharedPreferences.Editor spedit = spref.edit();
    EditText usr = (EditText) findViewById(R.id.ids1Usr);
    EditText pwd = (EditText) findViewById(R.id.ids1Pwd);
    spedit.putString("user", usr.getText().toString());
    spedit.putString("password", pwd.getText().toString());
    spedit.commit();
    Toast.makeText(this, "3rd party credentials saved successfully!", 0).show();
}
```

**Проблема:** Логин и пароль хранятся в plaintext в `SharedPreferences`.

**Как прочитать:**

```bash
adb shell
su
cat /data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="password">w</string>
    <string name="user">q</string>
</map>
```

**Исправление:** Использовать Android Keystore + шифрование (AES), либо EncryptedSharedPreferences из Android Jetpack:

```java
EncryptedSharedPreferences.create(
    context,
    "secret_shared_prefs",
    MasterKey.Builder(context).build(),
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
);
```


#### Пример 4: Insecure Data Storage Part 2 (d4)

```java
// InsecureDataStorage2Activity
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    try {
        this.mDB = openOrCreateDatabase("ids2", 0, null);
        this.mDB.execSQL("CREATE TABLE IF NOT EXISTS myuser(user VARCHAR, password VARCHAR);");
    } catch (Exception e) {
        Log.d("Diva", "Error occurred while creating database: " + e.getMessage());
    }
    setContentView(R.layout.activity_insecure_data_storage2);
}

public void saveCredentials(View view) {
    EditText usr = (EditText) findViewById(R.id.ids2Usr);
    EditText pwd = (EditText) findViewById(R.id.ids2Pwd);
    try {
        this.mDB.execSQL("INSERT INTO myuser VALUES ('" + usr.getText().toString() + "', '" + pwd.getText().toString() + "');");
        this.mDB.close();
    } catch (Exception e) {
        Log.d("Diva", "Error occurred while inserting into database: " + e.getMessage());
    }
    Toast.makeText(this, "3rd party credentials saved successfully!", 0).show();
}
```

**Проблема:** Пароли хранятся в plaintext в SQLite-базе

**Как прочитать:**

```bash
adb shell
sqlite3 /data/data/jakhar.aseem.diva/databases/ids2
sqlite> SELECT * FROM myuser;
qwe|asd
thisIsLogin|thisIsPassword
```

**Исправление:** Шифровать данные перед записью, использовать параметризованные запросы:

```java
mDB.execSQL("INSERT INTO myuser VALUES (?, ?)", new String[]{user, pwd});
```


#### Пример 5: Insecure Data Storage Part 3 (d5)

```java
// InsecureDataStorage3Activity
public void saveCredentials(View view) {
    EditText usr = (EditText) findViewById(R.id.ids3Usr);
    EditText pwd = (EditText) findViewById(R.id.ids3Pwd);
    File ddir = new File(getApplicationInfo().dataDir);
    File uinfo = File.createTempFile("uinfo", "tmp", ddir);
    uinfo.setReadable(true);
    uinfo.setWritable(true);
    FileWriter fw = new FileWriter(uinfo);
    fw.write(usr.getText().toString() + ":" + pwd.getText().toString() + "\n");
    fw.close();
}
```

**Проблема:** Файл с кредами создаётся как temp-файл прямо в `dataDir` приложения (`/data/data/jakhar.aseem.diva/`), а не в `cacheDir`, и **не удаляется** после использования - он переживает перезапуск приложения. Имя предсказуемо: `uinfo` + случайный суффикс + `tmp` (паттерн `uinfoXXXXXtmp`).

Файл лежит в приватной директории приложения, поэтому на **rooted** устройстве/эмуляторе его можно прочитать через `adb shell su` (или `adb root`).

**Как найти:**

```bash
adb shell
ls /data/data/jakhar.aseem.diva/uinfo*tmp*
cat /data/data/jakhar.aseem.diva/uinfoXXXXXtmp
# user:password
# либо сразу:
cat $(ls /data/data/jakhar.aseem.diva/uinfo*)
# thisIsLogin:thisIsPassword
```

**Исправление:** Использовать `openFileOutput()` с MODE_PRIVATE:

```java
FileOutputStream fos = openFileOutput("credentials", MODE_PRIVATE);
fos.write((user + ":" + pwd).getBytes());
fos.close();
```


#### Пример 6: Insecure Data Storage Part 4 (d6)

```java
// InsecureDataStorage4Activity
public void saveCredentials(View view) {
    EditText usr = (EditText) findViewById(R.id.ids4Usr);
    EditText pwd = (EditText) findViewById(R.id.ids4Pwd);
    File sdir = Environment.getExternalStorageDirectory();
    File uinfo = new File(sdir.getAbsolutePath() + "/.uinfo.txt");
    uinfo.setReadable(true);
    uinfo.setWritable(true);
    FileWriter fw = new FileWriter(uinfo);
    fw.write(usr.getText().toString() + ":" + pwd.getText().toString() + "\n");
    fw.close();
}
```

**Проблема:** Файл пишется на **external storage** (`/sdcard/.uinfo.txt`) - на Android < 10 (legacy storage) он доступен любому приложению с разрешением `READ_EXTERNAL_STORAGE`. На Android 10+ работает Scoped Storage, но DIVA таргетирует старый API, поэтому уязвимость актуальна

**Проверка:**

```bash
adb shell
ls -la /sdcard/.uinfo.txt
cat /sdcard/.uinfo.txt
# hello:mynameisslimshady
```

**Исправление:** Не хранить аутентификационные данные на external storage. Использовать Internal Storage с MODE_PRIVATE:

```java
FileOutputStream fos = openFileOutput("credentials", Context.MODE_PRIVATE);
```


#### Пример 7: SQL Injection (d7)

```java
// SQLInjectionActivity
public void search(View view) {
    EditText srchtxt = (EditText) findViewById(R.id.ivi1search);
    try {
        Cursor cr = this.mDB.rawQuery("SELECT * FROM sqliuser WHERE user = '" + srchtxt.getText().toString() + "'", null);
        StringBuilder strb = new StringBuilder("");
        if (cr != null && cr.getCount() > 0) {
            cr.moveToFirst();
            do {
                strb.append("User: (" + cr.getString(0) + ") pass: (" + cr.getString(1) + ") Credit card: (" + cr.getString(2) + ")\n");
            } while (cr.moveToNext());
        } else {
            strb.append("User: (" + srchtxt.getText().toString() + ") not found");
        }
        Toast.makeText(this, strb.toString(), 0).show();
    } catch (Exception e) {
        Log.d("Diva-sqli", "Error occurred while searching in database: " + e.getMessage());
    }
}
```

**Проблема:** SQL-инъекция через конкатенацию строки запроса.

**Проверка:**

```
# Ввести в DIVA:
' OR '1'='1
# Во всплывающем toast сообщении будут все записи из таблицы (login, password, CC)
```

**Исправление:** Использовать параметризованный запрос:

```java
db.rawQuery("SELECT * FROM sqliuser WHERE user = ?", new String[]{userInput});
```


#### Пример 8: Input Validation Part 2 (d8)

```java
// InputValidation2URISchemeActivity
public void get(View view) {
    EditText uriText = (EditText) findViewById(R.id.ivi2uri);
    WebView wview = (WebView) findViewById(R.id.ivi2wview);
    wview.loadUrl(uriText.getText().toString());
}
```

В `onCreate()` также:
```java
webSettings.setJavaScriptEnabled(true);
```

**Проблема:** WebView загружает URL без какой-либо валидации. Пользователь может ввести любой URI:

- `file:///data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml` - чтение SharedPreferences
- `file:///etc/hosts` - чтение системных файлов

**Проверка:**

```
# Ввести в поле WebView:
file:///data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml
# WebView отобразит содержимое XML с логином и паролем
```

**Исправление:** Валидировать URI, разрешить только http/https:

```java
URI uri = new URI(uriText.getText().toString());
if (!"http".equals(uri.getScheme()) && !"https".equals(uri.getScheme())) {
    Toast.makeText(this, "Invalid scheme!", Toast.LENGTH_SHORT).show();
    return;
}
wview.loadUrl(uri.toString());
```


#### Пример 9: Access Control Part 1 (d9)

```java
// AccessControl1Activity
public void viewAPICredentials(View view) {
    Intent i = new Intent();
    i.setAction("jakhar.aseem.diva.action.VIEW_CREDS");
    if (i.resolveActivity(getPackageManager()) != null) {
        startActivity(i);
    } else {
        Toast.makeText(this, "Error while getting API details", 0).show();
        Log.e("Diva-aci1", "Couldn't resolve the Intent VIEW_CREDS to our activity");
    }
}
```

```xml
<!-- AndroidManifest.xml -->
<activity
    android:label="@string/apic_label"
    android:name="jakhar.aseem.diva.APICredsActivity">
    <intent-filter>
        <action android:name="jakhar.aseem.diva.action.VIEW_CREDS"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</activity>
```

**Проблема:**
`Implicit intent` с action `"jakhar.aseem.diva.action.VIEW_CREDS"` - без `package` или `component`, любое приложение может создать intent с таким же action и перехватить (если зарегистрирует свой receiver).

**Проверка**
```bash
adb shell am start -a jakhar.aseem.diva.action.VIEW_CREDS
```

**Исправление:** Использовать explicit intent, не хардкодить креды:

```java
Intent i = new Intent("jakhar.aseem.diva.action.VIEW_CREDS");
i.setPackage("jakhar.aseem.diva");
```


#### Пример 10: Access Control Part 2 (d10)

```java
// APICreds2Activity - onCreate()
protected void onCreate(Bundle savedInstanceState) {
    boolean bcheck = getIntent().getBooleanExtra(getString(R.string.chk_pin), true);
    if (!bcheck) {
        tv.setText("TVEETER API Key: secrettveeterapikey\nAPI User name: diva2\nAPI Password: p@ssword2");
    } else {
        tv.setText("Register yourself at http://payatu.com to get your PIN and then login with that PIN!");
    }
}
```

Данная строка `getString(R.string.chk_pin)` означает нахождение названия функции в ресурсном файле:

```xml
<!-- resources.arsc/res/values/strings.xml -->
...
<string name="chk_pin">check_pin</string>
...
```

```xml
<!-- AndroidManifest.xml -->
<activity
    android:label="@string/d10"
    android:name="jakhar.aseem.diva.AccessControl2Activity"/>
<activity
    android:label="@string/apic2_label"
    android:name="jakhar.aseem.diva.APICreds2Activity">
    <intent-filter>
        <action android:name="jakhar.aseem.diva.action.VIEW_CREDS2"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</activity>
```

**Проблема:** У `APICreds2Activity` в манифесте зарегистрирован intent-filter с action `jakhar.aseem.diva.action.VIEW_CREDS2`, поэтому его можно запустить напрямую извне приложения. Решение о проверке PIN принимается по boolean extra `check_pin`, который читается из intent: `if (!bcheck)` показывает креды, когда флаг `false`. Внешний вызывающий (через `adb` или другое приложение) сам контролирует этот extra и передаёт `check_pin=false`, обходя PIN. Суть - доверие значению из intent вместо реальной проверки состояния авторизации.

**Проверка через ADB:**

```bash
adb shell am start -a jakhar.aseem.diva.action.VIEW_CREDS2 --ez check_pin false
```

**Исправление:** Не доверять флагу из intent - требовать PIN всегда, валидировать на сервере.


#### Пример 11: Access Control Part 3 (d11)

```xml
<!-- AndroidManifest.xml - NotesProvider -->
<provider
    android:authorities="jakhar.aseem.diva.provider.notesprovider"
    android:enabled="true"
    android:exported="true"
    android:name="jakhar.aseem.diva.NotesProvider"/>
```

**Проблема:** ContentProvider экспортирован (`exported="true"`) без всяких разрешений (`readPermission`/`writePermission` отсутствуют). По задумке доступ к заметкам открывается только после ввода PIN в `AccessControl3NotesActivity`, но из-за конфигурации провайдера любое приложение на устройстве (или ADB) может читать, вставлять и удалять записи напрямую, минуя UI и PIN.

Целевой URI собирается из манифеста (`authorities`) и имени таблицы (`notes`, видно в `NotesProvider.java`):

```java
// NotesProvider
static final Uri CONTENT_URI = Uri.parse("content://jakhar.aseem.diva.provider.notesprovider/notes");
static final UriMatcher urimatcher = new UriMatcher(-1);

static {
    urimatcher.addURI(AUTHORITY, TABLE, 1);
    urimatcher.addURI(AUTHORITY, "notes/#", 2);
}

```

```
content://jakhar.aseem.diva.provider.notesprovider/notes
или
content://jakhar.aseem.diva.provider.notesprovider/notes/2
```

**Проверка через ADB:**

```bash
adb shell content query --uri content://jakhar.aseem.diva.provider.notesprovider/notes
```

**Дополнительно:** PIN в `AccessControl3NotesActivity` сравнивается со значением из SharedPreferences (`notespin`), которое тоже лежит в plaintext - отдельная уязвимость хранения:

```java
// AccessControl3NotesActivity - accessNotes()
String pin = sp.getString("notespin", "");     // PIN хранится в plaintext
if (userPin.equals(pin)) { ... }               // обход - прочитать prefs через adb root
```

```bash
adb shell cat /data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml
# <string name="notespin">1234</string>
```

**Исправление:**

1. Для ContentProvider - закрыть экспорт или защитить разрешениями:

```xml
<provider
    android:exported="false"
    android:readPermission="jakhar.aseem.diva.READ_NOTES"
    android:writePermission="jakhar.aseem.diva.WRITE_NOTES"
    android:name="jakhar.aseem.diva.NotesProvider"/>
```

2. Для PIN - не хранить в plaintext, хешировать (bcrypt/Argon2) или держать в Android Keystore.


#### Пример 12: Hardcode Issues Part 2 - JNI (d12)

```java
// Hardcode2Activity
public void access(View view) {
    EditText hckey = (EditText) findViewById(R.id.hc2Key);
    if (this.djni.access(hckey.getText().toString()) != 0) {
        Toast.makeText(this, "Access granted! See you on the other side :)", 0).show();
    } else {
        Toast.makeText(this, "Access denied! See you in hell :D", 0).show();
    }
}
```

```java
// DivaJni
public native int access(String str);
static { System.loadLibrary("divajni"); }
```

**Проблема:** Секрет перенесён в нативную библиотеку (`lib/<abi>/libdivajni.so`). Однако строка лежит в `.rodata` открытым текстом и находится обычным `strings`.

**Как найти:**

```bash
# strings сразу покажет секрет - для функции access это 'olsdfgad;lh'
strings lib/x86_64/libdivajni.so

dish@BOOK0HLD1C:~$ strings DivaApplication/lib/x86_64/libdivajni.so
__cxa_finalize
...
<$!H
olsdfgad;lh
.dotdot
;*3$"
GCC: (GNU) 4.9 20140827 (prerelease)
gold 1.11
```

``` bash
# objdump: access() сравнивает 11 байт с 'olsdfgad;lh'
objdump -d DivaApplication/lib/x86_64/libdivajni.so

...
00000000000004e0 <Java_jakhar_aseem_diva_DivaJni_access>:
 4e0:   48 8d 64 24 f8          lea    -0x8(%rsp),%rsp
 4e5:   48 89 d6                mov    %rdx,%rsi
 4e8:   48 8b 07                mov    (%rdi),%rax
 4eb:   31 d2                   xor    %edx,%edx
 4ed:   ff 90 48 05 00 00       call   *0x548(%rax)
 4f3:   b9 0b 00 00 00          mov    $0xb,%ecx
 4f8:   48 8d 35 67 00 00 00    lea    0x67(%rip),%rsi        # 566 <JNI_OnLoad+0x6> - надо посмотреть, что лежит по этому адресу 
 4ff:   48 89 c7                mov    %rax,%rdi
 502:   f3 a6                   repz cmpsb %es:(%rdi),%ds:(%rsi)
 504:   0f 94 c0                sete   %al
 507:   48 8d 64 24 08          lea    0x8(%rsp),%rsp
 50c:   0f b6 c0                movzbl %al,%eax
 50f:   c3                      ret
...

objdump -s DivaApplication/lib/x86_64/libdivajni.so
...
Contents of section .rodata:
 0566 6f6c7364 66676164 3b6c6800 2e646f74  olsdfgad;lh..dot
 0576 646f7400                             dot.
...
```

Секрет для d12: **`olsdfgad;lh`**

**Исправление:** Не хардкодить секреты вообще. Использовать серверную валидацию, Android Keystore, или dynamic loading с обфускацией (обфускация - не защита).


#### Пример 13: Input Validation Part 3 (d13)

```java
// InputValidation3Activity
public void push(View view) {
    EditText cTxt = (EditText) findViewById(R.id.ivi3CodeText);
    if (this.djni.initiateLaunchSequence(cTxt.getText().toString()) != 0) {
        Toast.makeText(this, "Launching in T - 10 ...", 0).show();
    } else {
        Toast.makeText(this, "Access denied!", 0).show();
    }
}
```

```java
// DivaJni
public native int initiateLaunchSequence(String str);
```

**Проблема:** Проверка кода выполняется в нативной библиотеке `libdivajni.so`. Здесь, в отличие от d12, есть хитрость: `strings` покажет строку `.dotdot`, а ввести можно как **`.dotdot`**, так и **`!dotdot`**.

**Логика функции `initiateLaunchSequence`** (видно в дизассемблере):
1. Введённая строка копируется в локальный буфер;
2. Если первый символ равен `'!'` (0x21) - он заменяется на `'.'` (0x2e);
3. Затем 7 байт буфера сравниваются со строкой `.dotdot`.

То есть вводим `!dotdot` -> после подмены получается `.dotdot` -> совпадает -> «Launching».

**Как найти:**

```bash
# strings покажет '.dotdot' - но это ещё не ответ, нужен дизассемблер
strings lib/x86_64/libdivajni.so | grep dotdot
# дизассемблер показывает подмену 0x21 -> 0x2e и сравнение 7 байт
objdump -d lib/x86_64/libdivajni.so
# либо через Ghidra/IDA: найти Java_jakhar_aseem_diva_DivaJni_initiateLaunchSequence
```

Ответ для d13: **`!dotdot`** или **`.dotdot`**

**Исправление:** Перенести проверку на сервер, использовать подписанные токены.

Однако, кто нам мешает пропатчить приложение и скипнуть проверку? 

Декомпилируем приложение:
```bash
java -jar apktool_3.0.3.jar d DivaApplication.apk
```

Вот наши проверки в smali в файле InputValidation3Activity.smali

```smali
.line 24
.local v0, "cTxt":Landroid/widget/EditText;
iget-object v1, p0, Ljakhar/aseem/diva/InputValidation3Activity;->djni:Ljakhar/aseem/diva/DivaJni;
invoke-virtual {v0}, Landroid/widget/EditText;->getText()Landroid/text/Editable;
move-result-object v2
invoke-virtual {v2}, Ljava/lang/Object;->toString()Ljava/lang/String;
move-result-object v2
invoke-virtual {v1, v2}, Ljakhar/aseem/diva/DivaJni;->initiateLaunchSequence(Ljava/lang/String;)I
move-result v1
if-eqz v1, :cond_24

.line 25
const-string v1, "Launching in T - 10 ..."
invoke-static {p0, v1, v3}, Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;
move-result-object v1
invoke-virtual {v1}, Landroid/widget/Toast;->show()V
```
Видно строчку, в которой происходит сравнение: `if-eqz v1, :cond_24`, и мы можем поменять условие на противоположное: `if-nez v1, :cond_24`

Это равносильно тому, что мы поменяли 
```java
(this.djni.initiateLaunchSequence(cTxt.getText().toString()) != 0)
```

на

```java
(this.djni.initiateLaunchSequence(cTxt.getText().toString()) == 0)
```

Соберем приложение
```bash
java -jar apktool_3.0.3.jar b DivaApplication -o diva-patched.apk
I: Using Apktool 3.0.3 on DivaApplication.apk with 8 threads
I: AndroidManifest.xml and resources have not changed.
I: Smaling smali folder into classes.dex...
I: Building apk file...
I: Importing lib...
I: Built apk into: diva-patched.apk
```

Подпишем приложение
```bash
java -jar uber-apk-signer-1.3.0.jar -a diva-patched.apk
```

Найдем уже установленное и удалим его, после чего установим пропатченное
```bash
>adb shell pm list packages | grep diva
package:jakhar.aseem.diva

>adb uninstall jakhar.aseem.diva
Success

>adb install diva-patched-aligned-debugSigned.apk
Performing Incremental Install
Performing Streamed Install
Success
```

Осталось ввести что-либо, кроме правильного пароля, и увидеть пройденную проверку
