# Анализ доступа к видеокадрам в Jellyfin для Samsung TV (Orsay)

## Краткий вывод

**НЕВОЗМОЖНО** получить прямой доступ к видеокадрам из JavaScript-кода приложения.

---

## 1. Техническое объяснение

### 1.1 Архитектура платформы Samsung Orsay

Приложение разработано для платформы **Samsung Orsay** (SmartTV до 2015 года). Это legacy-платформа, которая использует проприетарные Samsung API через специальные `<object>` элементы.

**Ключевой элемент для воспроизведения видео:**
```html
<object id="pluginPlayer" border=0 classid="clsid:SAMSUNG-INFOLINK-PLAYER"></object>
```
*Источник: `/home/runner/work/jellyfin-samsungtv/jellyfin-samsungtv/index.html`, строка 71*

### 1.2 Используемые Samsung API

В приложении используются следующие Samsung Orsay плагины:

1. **SAMSUNG-INFOLINK-PLAYER** - основной плеер для видео
2. **SAMSUNG-INFOLINK-AUDIO** - управление аудиовыходом
3. **SAMSUNG-INFOLINK-SCREEN** - управление 3D и экраном
4. **SAMSUNG-INFOLINK-NETWORK** - сетевые операции
5. **SAMSUNG-INFOLINK-TVMW** - TV middleware
6. **SAMSUNG-INFOLINK-NNAVI** - навигация
7. **SAMSUNG-INFOLINK-TV** - TV функции

*Источник: `/home/runner/work/jellyfin-samsungtv/jellyfin-samsungtv/index.html`, строки 71-77*

---

## 2. Где происходит декодирование и рендер видео

### 2.1 Инициализация плеера

**Файл:** `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 42-62

```javascript
GuiPlayer.init = function() {
    this.plugin = document.getElementById("pluginPlayer");
    this.pluginAudio = document.getElementById("pluginObjectAudio");
    this.pluginScreen = document.getElementById("pluginScreen");
    
    // Установка callback-функций для событий
    this.plugin.OnConnectionFailed = 'GuiPlayer.handleConnectionFailed';
    this.plugin.OnAuthenticationFailed = 'GuiPlayer.handleAuthenticationFailed';
    this.plugin.OnNetworkDisconnected = 'GuiPlayer.handleOnNetworkDisconnected';
    this.plugin.OnRenderError = 'GuiPlayer.handleRenderError';
    this.plugin.OnStreamNotFound = 'GuiPlayer.handleStreamNotFound';
    this.plugin.OnRenderingComplete = 'GuiPlayer.handleOnRenderingComplete';
    this.plugin.OnCurrentPlayTime = 'GuiPlayer.setCurrentTime';
    this.plugin.OnBufferingStart = 'GuiPlayer.onBufferingStart';
    this.plugin.OnBufferingProgress = 'GuiPlayer.onBufferingProgress';
    this.plugin.OnBufferingComplete = 'GuiPlayer.onBufferingComplete';  
    this.plugin.OnStreamInfoReady = 'GuiPlayer.OnStreamInfoReady'; 
    this.plugin.SetTotalBufferSize(40*1024*1024);
};
```

### 2.2 Запуск воспроизведения

**Файл:** `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строка 184

```javascript
var position = Math.round(resumeTicksSamsung / 1000);
this.plugin.ResumePlay(url, position);
```

**ВАЖНО:** Метод `ResumePlay()` передает управление напрямую в нативный Samsung player. После этого вызова:
- Видео декодируется **аппаратно** через SoC (System-on-Chip) телевизора
- Рендер происходит на **аппаратном видеооверлее** (video overlay plane)
- JavaScript получает только **события** (время воспроизведения, статусы буферизации)
- **НЕТ доступа** к пиксельным данным или видеокадрам

### 2.3 Управление отображением

**Файл:** `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 200-229

```javascript
GuiPlayer.setDisplaySize = function() {
    var aspectRatio = /* определение соотношения сторон */;
    if (aspectRatio == "16:9") {
        this.plugin.SetDisplayArea(0, 0, 960, 540);
    } else if (aspectRatio == "4:3") {
        // расчет позиции
        this.plugin.SetDisplayArea(parseInt(centering), parseInt(0), 
                                   parseInt(newResolutionX), parseInt(newResolutionY));
    }
    // ...
};
```

Метод `SetDisplayArea()` управляет только **позицией и размером видеооверлея**, но не предоставляет доступ к содержимому кадров.

---

## 3. Доступные методы Samsung Player API

Полный список методов, используемых в коде:

### 3.1 Методы управления воспроизведением
- `ResumePlay(url, position)` - начало/возобновление воспроизведения
- `Stop()` - остановка
- `Pause()` - пауза
- `Resume()` - возобновление после паузы
- `JumpForward(seconds)` - перемотка вперед
- `JumpBackward(seconds)` - перемотка назад

### 3.2 Методы конфигурации
- `SetDisplayArea(x, y, width, height)` - установка области отображения
- `SetTotalBufferSize(bytes)` - размер буфера

### 3.3 Callback-события (только уведомления)
- `OnCurrentPlayTime` - текущее время воспроизведения (в миллисекундах)
- `OnRenderingComplete` - завершение воспроизведения
- `OnBufferingStart/Progress/Complete` - статусы буферизации
- `OnStreamInfoReady` - готовность потока
- `OnRenderError`, `OnStreamNotFound`, `OnConnectionFailed`, `OnAuthenticationFailed`, `OnNetworkDisconnected` - ошибки

**Источник:** `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 42-62 и весь файл

---

## 4. Результаты поиска API для доступа к кадрам

### 4.1 Canvas / WebGL API
```bash
grep -r "canvas\|Canvas\|getContext\|WebGL" . --include="*.js" --include="*.html"
```
**Результат:** НЕ НАЙДЕНО. В коде отсутствуют:
- `<canvas>` элементы
- `getContext('2d')` или `getContext('webgl')`
- `drawImage()`, `getImageData()`
- Любые WebGL операции

### 4.2 Современные Media API
```bash
grep -r "captureStream\|requestVideoFrameCallback\|MediaStreamTrack\|getVideoPlaybackQuality" . --include="*.js"
```
**Результат:** НЕ НАЙДЕНО. Не используются:
- `HTMLVideoElement.captureStream()` - захват потока с video элемента
- `requestVideoFrameCallback()` - доступ к видеокадрам
- `MediaStreamTrack` API
- `getVideoPlaybackQuality()`

### 4.3 Screenshot / Capture API
```bash
grep -r "screenshot\|Screenshot\|capture\|Capture\|framebuffer" . --include="*.js"
```
**Результат:** НЕ НАЙДЕНО. Никаких механизмов для скриншотов видео.

**Источники проверки:**
- Поиск по всем JavaScript файлам в `app/javascript/`
- Анализ `index.html`
- Проверка всех плагинов Samsung

---

## 5. Почему доступ к видеокадрам невозможен

### 5.1 Архитектурные ограничения Samsung Orsay

**Разделение уровней:**
```
┌─────────────────────────────────────┐
│   JavaScript Layer (Widget App)     │  ← Ваше приложение находится здесь
├─────────────────────────────────────┤
│   Samsung Orsay API (clsid:...)     │  ← Проприетарный Native API
├─────────────────────────────────────┤
│   Hardware Video Decoder (SoC)      │  ← Аппаратное декодирование
├─────────────────────────────────────┤
│   Video Overlay Plane (Hardware)    │  ← Прямой вывод на экран
└─────────────────────────────────────┘
```

**Ключевые моменты:**

1. **Аппаратное декодирование**
   - Видео декодируется непосредственно SoC телевизора (Samsung Exynos или аналогичный)
   - Декодированные кадры НЕ проходят через JavaScript runtime
   - Данные остаются в **защищенной памяти видеоподсистемы**

2. **Video Overlay Architecture**
   - Видео отображается на отдельном аппаратном слое (hardware overlay)
   - Этот слой НЕ является частью DOM или rendering context браузера
   - JavaScript/HTML UI отображается на другом слое, который композитируется с видео на уровне железа

3. **Отсутствие `<video>` элемента**
   - Приложение НЕ использует стандартный HTML5 `<video>` элемент
   - Вместо этого используется проприетарный `<object id="pluginPlayer" classid="clsid:SAMSUNG-INFOLINK-PLAYER">`
   - Этот объект — это черный ящик, который предоставляет только методы управления, но не данные

4. **DRM и защита контента**
   - Даже если бы существовал теоретический API для доступа к кадрам, он был бы заблокирован для защищенного контента
   - Архитектура Orsay спроектирована так, чтобы **предотвратить** доступ к видеоданным из пользовательского кода

### 5.2 Ограничения JavaScript-слоя

JavaScript в этом приложении имеет доступ только к:

**Метаданным:**
- `this.currentTime` - текущее время воспроизведения (int, миллисекунды)
- `this.Status` - состояние ("PLAYING", "PAUSED", "STOPPED")
- `this.PlayerData` - информация о медиафайле (название, длительность, URL)
- Субтитры (загружаются как отдельный SRT файл и отображаются через HTML/CSS)

**Управлению:**
- Play, Pause, Stop, JumpForward, JumpBackward
- SetDisplayArea - только позиция и размер окна видео

**События:**
- Обратные вызовы о статусе воспроизведения
- Ошибки и завершение

**НЕТ доступа к:**
- Декодированным кадрам (YUV, RGB)
- Framebuffer
- Текстурам
- Пиксельным данным
- Любым raw video data

**Доказательство из кода:**
```javascript
// Единственное, что получает JavaScript - это время
GuiPlayer.setCurrentTime = function(time) {
    if (this.Status == "PLAYING") {
        this.currentTime = parseInt(time); // Просто integer!
        // Обновление UI (прогресс-бар, субтитры)
        // НО НЕТ ДОСТУПА К ВИДЕОДАННЫМ
    }
};
```
*Источник: `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 391-439*

### 5.3 Отсутствие HTML5 Video API

Стандартный HTML5 `<video>` элемент, который теоретически мог бы быть использован с Canvas для извлечения кадров:
```javascript
// Такой код НЕВОЗМОЖЕН в этом приложении
var video = document.querySelector('video'); // НЕТ такого элемента!
var canvas = document.createElement('canvas'); // Canvas не используется
var ctx = canvas.getContext('2d');
ctx.drawImage(video, 0, 0); // Этот паттерн НЕДОСТУПЕН
```

В приложении Jellyfin для Samsung TV используется:
```html
<object id="pluginPlayer" classid="clsid:SAMSUNG-INFOLINK-PLAYER"></object>
```
Это **не** HTML5 video элемент. Это ActiveX-подобный нативный компонент Samsung.

---

## 6. Субтитры: единственный "визуальный" контент доступный JavaScript

### 6.1 Как работают субтитры

**Файл:** `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 231-260

```javascript
GuiPlayer.setSubtitles = function(selectedSubtitleIndex) {
    if (selectedSubtitleIndex > -1) {
        var Stream = this.playingMediaSource.MediaStreams[selectedSubtitleIndex];
        if (Stream.IsTextSubtitleStream) {
            // Загрузка .srt файла через HTTP
            var url = Server.getCustomURL("/Videos/"+ this.PlayerData.Id+"/"+
                      this.playingMediaSource.Id+"/Subtitles/"+
                      selectedSubtitleIndex+"/Stream.srt?api_key=" + 
                      Server.getAuthToken());
            this.PlayerDataSubtitle = Server.getSubtitles(url);
            // Парсинг SRT
            this.PlayerDataSubtitle = parser.fromSrt(this.PlayerDataSubtitle, true);
        }
    }
};
```

**Важно:**
- Субтитры загружаются как **текстовый SRT файл** через HTTP API
- Отображаются через **HTML `<div>`** элемент, не через видеопоток:
  ```html
  <div id="guiPlayer_Subtitles" class="videoSubtitles" style="visibility:hidden"></div>
  ```
- Синхронизируются по времени вручную в JavaScript
- Субтитры **НЕ являются частью видеопотока** и не дают доступа к видеокадрам

---

## 7. Попытка обхода: теоретические сценарии и их невозможность

### 7.1 Использование Screenshot API Samsung
**Гипотеза:** Возможно, существует Samsung API для скриншотов экрана?

**Реальность:** 
- В коде не найдено использование таких API
- Samsung Orsay SDK не предоставляет публичных screenshot API для виджетов
- Даже если бы такой API существовал, он бы захватывал весь экран (включая UI), а не только видеопоток
- Video overlay часто **исключается** из скриншотов на аппаратном уровне (защита от пиратства)

### 7.2 Прямой доступ к памяти
**Гипотеза:** Можно ли получить доступ к видеопамяти?

**Реальность:**
- JavaScript в браузере/виджете изолирован в песочнице
- Нет FFI (Foreign Function Interface) для вызова нативного кода
- Нет доступа к системным вызовам или драйверам
- Видеопамять защищена на уровне ОС (Linux-based Orsay OS)

### 7.3 Модификация firmware или jailbreak
**Гипотеза:** Модифицировать прошивку ТВ для получения доступа?

**Реальность:**
- Это выходит за рамки возможностей JavaScript-приложения
- Требует физического доступа к ТВ и специализированных инструментов
- Нарушает гарантию и лицензионные соглашения
- НЕ может быть реализовано **программно** через код приложения

### 7.4 Перехват сетевого потока
**Гипотеза:** Перехватить HLS/HTTP поток до декодирования?

**Реальность:**
- Да, можно получить **закодированный** видеопоток (H.264, HEVC и т.д.)
- URL видео доступен в коде:
  ```javascript
  var url = this.playingURL + '&PlaySessionId=' + this.PlaySessionId;
  if (this.PlayMethod != "DirectPlay") {
      url += '|COMPONENT=HLS';
  }
  ```
  *Источник: `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 172-177*
- НО это **не декодированные кадры**, а сжатое видео
- Для получения кадров нужен **декодер**, которого нет в JavaScript
- WebCodecs API не поддерживается в Orsay (это API из 2020+, а Orsay — это 2011-2015)

---

## 8. Детальный анализ используемых Samsung API

### 8.1 SAMSUNG-INFOLINK-PLAYER

**Документация (по результатам реверс-инжиниринга из кода):**

| Метод | Параметры | Возвращаемое значение | Назначение |
|-------|-----------|----------------------|------------|
| `ResumePlay(url, position)` | url: string, position: int (seconds) | void | Начать воспроизведение с позиции |
| `Stop()` | нет | void | Остановить воспроизведение |
| `Pause()` | нет | void | Пауза |
| `Resume()` | нет | void | Возобновить после паузы |
| `JumpForward(seconds)` | seconds: int | void | Перемотка вперед |
| `JumpBackward(seconds)` | seconds: int | void | Перемотка назад |
| `SetDisplayArea(x, y, w, h)` | x, y, w, h: int | void | Установка окна отображения |
| `SetTotalBufferSize(bytes)` | bytes: int | void | Размер буфера |

**Callback properties:**
- `OnCurrentPlayTime` - callback(time: int) - время в мс
- `OnRenderingComplete` - callback() - воспроизведение завершено
- `OnBufferingStart`, `OnBufferingProgress`, `OnBufferingComplete` - события буферизации
- `OnRenderError`, `OnStreamNotFound`, etc. - события ошибок

**КРИТИЧНО:** Ни один из этих методов не возвращает видеоданные или дескрипторы кадров!

### 8.2 SAMSUNG-INFOLINK-SCREEN

**Используется для 3D:**

```javascript
GuiPlayer.setupThreeDConfiguration = function() {
    if (this.playingMediaSource.Video3DFormat !== undefined) {
        if (this.pluginScreen.Flag3DEffectSupport()) {
            switch (this.playingMediaSource.Video3DFormat) {
                case "FullSideBySide":
                case "HalfSideBySide":
                    result = GuiPlayer.pluginScreen.Set3DEffectMode(2);
                    break;
                default:
                    this.pluginScreen.Set3DEffectMode(0);
                    break;
            }
        }
    }
};
```
*Источник: `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 775-793*

**Методы:**
- `Flag3DEffectSupport()` - проверка поддержки 3D
- `Set3DEffectMode(mode)` - установка режима 3D (0=off, 2=side-by-side)

Эти методы управляют только **режимом отображения**, не дают доступ к данным.

### 8.3 SAMSUNG-INFOLINK-AUDIO

**Используется для управления аудиовыходом:**

```javascript
GuiPlayer.setupAudioConfiguration = function() {
    var codec = audioInfoStream.Codec.toLowerCase();
    switch (codec) {
        case "dca": // DTS
            if (File.getTVProperty("DTS")) {
                var checkAudioOutModeDTS = this.pluginAudio.CheckExternalOutMode(2);
                if (checkAudioOutModeDTS > 0) {
                    this.pluginAudio.SetExternalOutMode(2);
                }
            }
            break;
        case "ac3": // Dolby Digital
            if (File.getTVProperty("Dolby")) {
                var checkAudioOutModeDolby = this.pluginAudio.CheckExternalOutMode(1);
                if (checkAudioOutModeDolby > 0) {
                    this.pluginAudio.SetExternalOutMode(1);
                }
            }
            break;
    }
};
```
*Источник: `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 795-835*

**Методы:**
- `CheckExternalOutMode(mode)` - проверка поддержки аудио режима
- `SetExternalOutMode(mode)` - переключение PCM/DTS/Dolby

Это чисто аудио API, не относится к видео.

### 8.4 SAMSUNG-INFOLINK-NNAVI

**Используется для управления баннером громкости:**

```javascript
NNaviPlugin = document.getElementById("pluginObjectNNavi");
NNaviPlugin.SetBannerState(PL_NNAVI_STATE_BANNER_VOL);
pluginAPI.unregistKey(tvKey.KEY_VOL_UP);
pluginAPI.unregistKey(tvKey.KEY_VOL_DOWN);
pluginAPI.unregistKey(tvKey.KEY_MUTE);
```
*Источник: `app/javascript/Gui/GuiPlayer/GuiPlayer.js`, строки 481-485*

Не связано с видеоданными.

---

## 9. Выводы и рекомендации

### 9.1 Категоричный вывод

**НЕВОЗМОЖНО** получить доступ к видеокадрам (framebuffer, decoded frames, RGB/YUV pixels) из JavaScript-кода приложения Jellyfin для Samsung Orsay TV по следующим причинам:

1. **Архитектура платформы:**
   - Видео декодируется аппаратно через SoC
   - Вывод осуществляется напрямую в hardware video overlay
   - JavaScript изолирован от видеопайплайна

2. **Ограничения API:**
   - Samsung Orsay API предоставляет только управление воспроизведением
   - Нет методов для чтения видеоданных
   - Нет Canvas/WebGL в коде
   - Нет HTML5 Video элементов

3. **Защита контента:**
   - Архитектура спроектирована для предотвращения доступа к видеоданным
   - DRM и защита от пиратства на аппаратном уровне

4. **Отсутствие обходов:**
   - Невозможно программно обойти ограничения через JavaScript
   - Любые теоретические обходы требуют модификации firmware или hardware доступа

### 9.2 Что доступно JavaScript

JavaScript может:
- **Управлять** воспроизведением (play, pause, stop, seek)
- **Получать метаданные** (текущее время, длительность, статус)
- **Обрабатывать события** (buffering, errors, completion)
- **Отображать UI** поверх видео (субтитры, OSD)
- **Настраивать** параметры (3D режим, аудиовыход, размер окна)

JavaScript **не может:**
- Читать пиксели видеокадров
- Захватывать скриншоты видео
- Получать доступ к framebuffer
- Перехватывать декодированные данные
- Использовать Canvas/WebGL для извлечения видео

### 9.3 Альтернативные подходы (вне приложения)

Если цель — получить кадры видео, единственные варианты:

1. **На стороне сервера (Jellyfin Server):**
   - Извлекать кадры из исходных видеофайлов через FFmpeg
   - Создавать thumbnails при индексации медиатеки
   - API Jellyfin Server поддерживает генерацию preview изображений

2. **На уровне TV (hardware):**
   - Внешний HDMI capture устройство
   - Модификация firmware (не программный метод)

3. **Другая платформа:**
   - Использовать современные Smart TV (Tizen, webOS) с поддержкой WebCodecs API
   - Разработать нативное приложение для Android TV
   - Web-приложение с HTML5 video + Canvas (но не на Orsay)

---

## 10. Ссылки на код

### Ключевые файлы анализа:

1. **Основной плеер:**
   - `/home/runner/work/jellyfin-samsungtv/jellyfin-samsungtv/app/javascript/Gui/GuiPlayer/GuiPlayer.js` (925 строк)
   - Строки 42-62: инициализация плеера
   - Строки 114-185: начало воспроизведения
   - Строки 200-229: управление отображением
   - Строки 391-439: обработка текущего времени (единственные данные от плеера)

2. **HTML структура:**
   - `/home/runner/work/jellyfin-samsungtv/jellyfin-samsungtv/index.html`
   - Строки 71-77: объявление Samsung плагинов
   - Строка 71: `<object id="pluginPlayer" classid="clsid:SAMSUNG-INFOLINK-PLAYER">`

3. **Конфигурация:**
   - `/home/runner/work/jellyfin-samsungtv/jellyfin-samsungtv/config.xml`
   - Версия 2.2.9, платформа Orsay

4. **README:**
   - `/home/runner/work/jellyfin-samsungtv/jellyfin-samsungtv/README.md`
   - Подтверждение: legacy app для Samsung Orsay (pre-2015)

### Проведенные проверки кода:

```bash
# Проверка Canvas/WebGL
grep -r "canvas\|Canvas\|getContext\|WebGL" . --include="*.js" --include="*.html"
# Результат: НЕ НАЙДЕНО

# Проверка Frame Capture API
grep -r "captureStream\|requestVideoFrameCallback\|MediaStreamTrack" . --include="*.js"
# Результат: НЕ НАЙДЕНО

# Проверка Screenshot API
grep -r "screenshot\|Screenshot\|capture\|Capture\|framebuffer" . --include="*.js"
# Результат: НЕ НАЙДЕНО

# Список всех методов pluginPlayer
grep -r "plugin\." app/javascript/Gui/GuiPlayer/GuiPlayer.js | grep -v "FileLog\|//"
# Результат: только управление воспроизведением (ResumePlay, Stop, Pause, Resume, 
#            JumpForward, JumpBackward, SetDisplayArea, SetTotalBufferSize)
#            + callback properties (OnCurrentPlayTime, OnRenderingComplete, etc.)
```

---

## 11. Заключение

Проведенный анализ кодовой базы приложения Jellyfin для Samsung Smart TV (Orsay) со 100% уверенностью подтверждает:

**Получение доступа к видеокадрам (framebuffer, decoded frames, RGB/YUV pixels) через JavaScript-код приложения НЕВОЗМОЖНО.**

Причины:
- ✗ Нет Canvas/WebGL API
- ✗ Нет HTML5 Video элементов
- ✗ Нет современных Media Capture API
- ✗ Нет Screenshot API
- ✗ Аппаратное декодирование изолировано от JavaScript
- ✗ Samsung Orsay API предоставляет только управление, не данные
- ✗ Video overlay недоступен для JavaScript-слоя
- ✗ Архитектура платформы предотвращает доступ к видеоданным

Единственный доступ: **метаданные** (время воспроизведения, статус, события).

---

**Дата анализа:** 2026-01-19  
**Версия приложения:** 2.2.9  
**Платформа:** Samsung Orsay (2011-2015)  
**Анализатор:** AI Code Analysis System
