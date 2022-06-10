# Chromium

## Сборка chromium с помощью clang+lto.

На текущий момент для компилированис с clang достаточно просто включиь флаг `lto`.

Также добавим поддержку аппаратного декодирования.

Настраиваем:

В `package.use` вписываем:

```
www-client/chromium widevine official vaapi custom-cflags lto
net-libs/nodejs inspector
sys-devel/llvm gold
```

И пересобираем:

```
emerge -1 chromium
```

## Оптимизация настроек chromium

Для того, чтобы у нас был полный праздник, надо ещё дать правильные параметры.

Создаём `/etc/chromium/optimize`:

```
CHROMIUM_FLAGS="${CHROMIUM_FLAGS}
--ignore-gpu-blocklist
"

CHROMIUM_FLAGS="${CHROMIUM_FLAGS}
--canvas-msaa-sample-count=2
--enable-accelerated-2d-canvas
--enable-checker-imaging
--enable-experimental-canvas-features
--enable-fast-unload
--enable-gpu-compositing
--enable-gpu-rasterization
--enable-gpu-memory-buffers
--enable-oop-rasterization
--enable-raw-draw
--enable-tcp-fastopen
--enable-webgl-image-chromium
--enable-zero-copy
--force-display-list-2d-canvas
--force-gpu-rasterization
--javascript-harmony
--ui-enable-zero-copy
--use-skia-deferred-display-list
--use-skia-renderer
--use-vulkan
"

CHROMIUM_FLAGS="${CHROMIUM_FLAGS}
--v8-cache-options=code
--v8-cache-strategies-for-cache-storage=aggressive
"

CHROMIUM_FLAGS="${CHROMIUM_FLAGS}
--enable-accelerated-video
--enable-accelerated-video-decode
--enable-accelerated-vpx-decode=3
--enable-accelerated-mjpeg-decode
--enable-vp9-kSVC-decode-acceleration
--enable-features=VaapiVideoDecoder,VaapiVideoEncoder,CanvasOopRasterization,VaapiIgnoreDriverChecks
--disable-features=UseChromeOSDirectVideoDecoder
--use-gl=egl
"
```

Запускаем chromium, заходим на страницу chrome://flags и включаем следующие опции:

```
#canvas-oop-rasterization
#enable-cast-streaming-av1
#enable-drdc
#enable-experimental-web-platform-features
#enable-future-v8-vm-features
#enable-javascript-harmony
#enable-quic
#enable-raw-draw
#enable-vp9-kSVC-decode-acceleration
#enable-zero-copy
#ignore-gpu-blocklist
#overlay-scrollbars
#scrollable-tabstrip
#smooth-scrolling
```

Перезапускаем chromium, заходим на chrome://gpu и радуемся тому, что видим :)

Также я заметил, что chromium ругается на один kernel параметр для i915, но его не получится вписать
в sysctl.conf потому, что драйвер ещё не загружен в тот момент, когда sysctl.conf применяется.
В моих настройках есть powersave скрипт в init.d в него можно вставить в функцию старт следующее:

```
sysctl -w dev.i915.perf_stream_paranoid=0
```

## VaapiVideoDecoder vs VDAVideoDecoder

На текущий момент есть два декодеры - VAAPI и VDA. На самом деле оба они используют hardware acceleration
и vaapi внутри, но первый эту умеет делать миную дополнительный текстуру-буфер, а для второго буфер нужен.
У меня пока на моём железе первый вариант не запустился нормально. Для его запуска надо следующие параметры:

```
--enable-features=VaapiVideoDecoder,VaapiVideoEncoder,CanvasOopRasterization,Vulkan,UseChromeOSDirectVideoDecoder,VaapiIgnoreDriverChecks
```

И убрать:

```
--disable-features=UseChromeOSDirectVideoDecoder
```
