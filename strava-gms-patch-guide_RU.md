# Гайд: Патчинг Strava APK для удаления зависимости от Google Play Services (GMS)

> **Цель:** Заставить Strava работать без GMS — на Huawei без Google, на телефонах с microG, на кастомных ROM.  
> **Протестировано на:** Strava v448.10 (ReVanced мод), Strava v455.11 (premium мод)  
> **ОС для работы:** Linux (Arch, Ubuntu и др.)

---

## Важные выводы из практики

### Какой мод выбрать
- **Strava v448.10 ReVanced** — авторизация через код на почту **работает** ✅
- **Strava v455.11 premium мод** — авторизация через код на почту **не работает** ❌ (проблема в реализации мода, не в патчинге GMS)
- Всегда проверяй авторизацию **до** патчинга GMS — если она не работает в моде, патчинг GMS не поможет

### Почему ReVanced лучше для повторного патчинга
- Уже пересобран и подписан → повторный apktool проходит без проблем
- Подпись уже не оригинальная → нет страха "испортить"
- Некоторые GMS-зависимости уже могут быть убраны

---

## Необходимые инструменты

```bash
# Arch Linux
sudo pacman -S android-tools jdk-openjdk
yay -S android-sdk-build-tools apktool

# Ubuntu/Debian
sudo apt install apktool aapt adb openjdk-17-jdk
# apksigner и zipalign — через android-sdk-build-tools или Android Studio
```

Проверь пути:
```bash
find /opt/android-sdk -name "apksigner" 2>/dev/null
find /opt/android-sdk -name "zipalign" 2>/dev/null
# Обычно: /opt/android-sdk/build-tools/37.0.0/apksigner
```

Также нужен **jadx-gui** для разведки:
- Скачать: https://github.com/skylot/jadx/releases

---

## Шаг 1 — Разведка через jadx-gui

Открой APK в jadx-gui. Через меню **Навигация → Поиск строк** ищи:

```
isGooglePlayServicesAvailable
```

Убедись что галочка **Код** включена. Запомни какие классы содержат этот вызов.

**Ключевые классы которые нужно патчить:**
- `com.strava.SplashActivity` — главная блокировка при запуске
- Обфусцированные классы типа `f8/a`, `T6/a`, `X8/e4`, `K7/h4` — внутренние проверки

**Классы которые можно пропустить** (не блокируют запуск):
- `com.facebook.internal.*` — аналитика Facebook
- `io.branch.referral.*` — реферальная система
- `com.google.android.gms.*` — сам GMS (не трогаем)
- `com.mapbox.common.location.*` — карты (отдельная история)
- `com.google.android.recaptcha.*` — капча (отдельная история)

---

## Шаг 2 — Декомпиляция

```bash
apktool d "НазваниеФайла.apk" -o output_dir
```

Если папка уже существует:
```bash
apktool d "НазваниеФайла.apk" -o output_dir -f
```

---

## Шаг 3 — Поиск всех вхождений в smali

```bash
grep -r "isGooglePlayServicesAvailable" output_dir/smali* -l
```

Получишь список файлов. Для каждого нужного файла:

```bash
grep -n "isGooglePlayServicesAvailable" output_dir/smali_classesX/путь/к/файлу.smali
```

Запомни номера строк.

---

## Шаг 4 — Просмотр контекста

Для каждого вхождения смотри контекст (±7 строк):

```bash
sed -n 'НОМЕР_СТРОКИ_МИНУС_7,НОМЕР_СТРОКИ_ПЛЮС_7p' путь/к/файлу.smali
```

**Что ищешь:** строку `move-result vX` после вызова. Именно в регистр `vX` попадает результат проверки GMS. Между вызовом и `move-result` могут быть `.line N` директивы — это нормально, их не трогаем.

**Два варианта условия после move-result:**
- `if-eqz vX, :cond_N` — если 0 то всё ок (стандартный)
- `if-nez vX, :cond_N` — если не 0 то всё ок (обратная логика)

В обоих случаях патч одинаковый — принудительно записываем 0 в регистр.

---

## Шаг 5 — Найти точный номер строки move-result

```bash
grep -n "move-result vX" путь/к/файлу.smali
```

Где `vX` — регистр из строки вызова (v0, v1, v2, v4 и т.д.). Выбери строку которая идёт сразу после нужного вызова по номеру.

---

## Шаг 6 — Вставка патча

Вставляем `const/4 vX, 0x0` **после** строки `move-result`:

```bash
sed -i 'НОМЕР_MOVE_RESULTa\    const/4 vX, 0x0' путь/к/файлу.smali
```

> ⚠️ Обязательно 4 пробела перед `const/4` — smali использует пробелы для отступов.

**Проверь результат:**
```bash
sed -n 'НОМЕР_МИНУС_3,НОМЕР_ПЛЮС_5p' путь/к/файлу.smali
```

**Правильный результат должен выглядеть так:**
```smali
    invoke-virtual {vA, vB}, Lcom/google/android/gms/common/GoogleApiAvailability;->isGooglePlayServicesAvailable(...)I
    .line 103          ← необязательно, может не быть
    move-result v2
    const/4 v2, 0x0   ← наш патч
    if-eqz v2, :cond_4
```

**Частая ошибка** — `const/4` вставляется ДО `move-result`. Тогда удаляй:
```bash
sed -i 'НОМЕР_НЕВЕРНОЙ_СТРОКИd' путь/к/файлу.smali
```

И повторяй grep для актуального номера строки (после каждой вставки/удаления номера сдвигаются!).

---

## Шаг 7 — Повторить для всех файлов

Патчим все файлы из списка (кроме исключений из Шага 1). Для каждого:
1. `grep -n "isGooglePlayServicesAvailable"` → номер строки вызова
2. `grep -n "move-result vX"` → номер строки move-result
3. `sed -i 'NUMa\    const/4 vX, 0x0'` → вставка
4. Проверка через `sed -n`

---

## Шаг 8 — Сборка

```bash
apktool b output_dir -o patched.apk
```

Должно завершиться без ошибок:
```
I: Building resources with aapt2...
I: Building apk file...
I: Built apk into: patched.apk
```

---

## Шаг 9 — Создание ключа подписи (один раз)

```bash
keytool -genkey -v -keystore ~/my.keystore -alias mykey \
  -keyalg RSA -keysize 2048 -validity 10000 -noprompt \
  -dname "CN=My, OU=My, O=My, L=My, S=My, C=US" \
  -storepass password -keypass password
```

Сохрани `~/my.keystore` — он нужен для всех будущих патчей. Если потеряешь — нужно будет переустанавливать все патченые APK.

---

## Шаг 10 — Подпись и выравнивание

**Строго в таком порядке: сначала zipalign, потом apksigner.**

```bash
# 1. Выравнивание
/opt/android-sdk/build-tools/37.0.0/zipalign -v 4 patched.apk aligned.apk

# 2. Подпись (v2 схема)
/opt/android-sdk/build-tools/37.0.0/apksigner sign \
  --ks ~/my.keystore \
  --ks-key-alias mykey \
  --ks-pass pass:password \
  --key-pass pass:password \
  --out final.apk aligned.apk
```

> ⚠️ Не используй `jarsigner` — он создаёт только v1 подпись, Android 7+ требует v2. APK установится с ошибкой.

> ⚠️ Если раньше использовал jarsigner — удали старую подпись перед apksigner:
> ```bash
> zip -d patched.apk "META-INF/*"
> ```

---

## Шаг 11 — Установка

```bash
adb install final.apk
```

Если ошибка конфликта подписей — удали старую версию приложения полностью:
```bash
adb shell pm uninstall com.strava
adb install final.apk
```

---

## Диагностика типичных ошибок

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `INSTALL_PARSE_FAILED_NO_CERTIFICATES` | Использован jarsigner вместо apksigner | Удали META-INF, пересобери с apksigner |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | Конфликт подписей | Удали старое приложение |
| `There was a problem parsing the package` | APK повреждён при сборке | Пересобери, проверь ошибки apktool |
| `const/4` вставился до `move-result` | Неверный номер строки в sed | Удали через `sed -i 'Nd'`, повтори grep |
| Код авторизации "устарел" сразу | Проблема в самом моде, не в GMS | Используй другую версию мода |

---

## Проверка авторизации без VPN (для России)

Strava заблокирована через CloudFront геоблок (`403 ERROR - configured to block access from your country`). Нужен полноценный туннель (WireGuard, VLESS) — SNI-сплиттинг не помогает так как CF видит реальный IP.

При авторизации через WireGuard на VPS — убедись что VPS не в датацентре из чёрного списка Strava.

---

## Быстрый чеклист

- [ ] Проверить авторизацию в моде ДО патчинга
- [ ] Декомпилировать apktool
- [ ] grep по всем smali* папкам
- [ ] Исключить facebook/branch/gms/mapbox/recaptcha
- [ ] Для каждого файла: найти move-result, вставить const/4, проверить
- [ ] Собрать apktool b
- [ ] zipalign → apksigner (именно в этом порядке)
- [ ] adb install
- [ ] Удалить старую версию если конфликт подписей
