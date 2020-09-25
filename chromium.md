# Chromium

## Предистория

Как-то у меня обновился gcc до версии 10.2.0 и пересобрался chromium (86-й) этим gcc...

После этого я обнаружил, что при запуске google meet, а если быть точнее - при использовании камеры,
chromium начал падать по SIGSEGV. После многих экспериментов до меня дошло, что причиной всему как раз связка
chromium и gcc 10.2.0. Это же всё навело меня на мысль - а как же правильно собирать chromium, чтобы он работал
круто и быстро.

Для начала я попробовал собрать chromium просто с попомщью clang (это был 10-й на тот момент).
После сборки проблема с падением пропала, но вот по бенчмаркам он стал работать на 5% медленней.

В качестве benchmark'ов я использовал два:
* http://chromium.github.io/octane/
* https://web.basemark.com/

Почитав, как всё собирает google, я понял, что надо идти в сторону LTO компиляции. Ну и конечно для начала попробовал
собрать в варианте gcc+lto. Собрать у меня ничего не вышло - сборка заваливалась где-то посередине. Вторая попытка была
в связке с clang+lto и сборка завалилась в том же самом месте...

Clang поддерживает два режима full lto и thin lto. И вот google вроде как раз собирает в режиме thin lto
(он немного упрощён по сравнению с полным, но даёт примерно такие же результаты как и полный).

И опять я провалился с этим на текущем clang - сборка завалилась в самом начале...

Последней попыткой было - обновление clang до 11-й версии. Она ещё не вышла, но уже есть rc3.
И вот тут наконец-то всё получилось. Chromium был собран в режиме thin lto. На бенчмарк тестах производительность
стала на 5% больше, чем в случае просто gcc.

Вообще интересно было бы посмотреть производительность при сборке gcc+lto, но похоже, что придётся подождать новых
версии gcc для этого эксперимента.

## Сборка chromium с помощью clang11+lto.

Кроме всего прочего хочется включить vaapi - чтобы video декодировалось с ускорением на gpu.
Для этого нам понадобится ebuild с патчем. Такой ebuild есть в моём overlay'е (взял я его вот отсюда: https://github.com/FireBurn/Overlay).

Настраиваем:

В `package.use` вписываем:

```
www-client/chromium widevine official vaapi custom-cflags
net-libs/nodejs inspector
sys-devel/llvm gold
```

Добавляем окружение сборки в `/etc/portage/env/chromium-clang`:

```
CC="clang"
CXX="clang++"
CFLAGS="${CFLAGS} -flto=thin"
CXXFLAGS="${CXXFLAGS} -flto=thin"
LDFLAGS="${LDFLAGS} ${CFLAGS}"
#LD="ld.lld"
AR="llvm-ar"
NM="llvm-nm"
RANLIB="llvm-ranlib"

EXTRA_GN="thin_lto_enable_optimizations=true use_lld=true use_thin_lto=true is_cfi=true"
```

Включаем его для chromium в `/etc/portage/package.env/common.env`:

```
www-client/chromium chromium-clang
```

Разрешаем clang 11-й версии. В `/etc/portage/package.accept_keywords/common.accept`:

```
<sys-devel/llvm-11.0.0.9999 **
<sys-devel/lld-11.0.0.9999 **
<sys-devel/clang-11.0.0.9999 **
<sys-devel/clang-runtime-11.0.0.9999 **
<sys-libs/compiler-rt-11.0.0.9999 **
<sys-libs/compiler-rt-sanitizers-11.0.0.9999 **
<sys-libs/libomp-11.0.0.9999 **
<sys-devel/llvmgold-11.0.0.9999 **
```

Также в `/etc/portage/package.unmask/common.unmask`:

```
<sys-devel/llvm-11.0.0.9999
<sys-devel/lld-11.0.0.9999
<sys-devel/clang-11.0.0.9999
<sys-devel/clang-runtime-11.0.0.9999
<sys-libs/compiler-rt-11.0.0.9999
<sys-libs/compiler-rt-sanitizers-11.0.0.9999
<sys-libs/libomp-11.0.0.9999
<sys-devel/llvmgold-11.0.0.9999
```

После этого можно пересобрать clang:

```
emerge -1 llvm clang lld
```

Ну и сам chromium:

```
emerge -1 chromium
```

## Оптимизация настроек chromium

Для того, чтобы у нас был полный праздник, надо ещё дать правильные параметры.

Создаём `/etc/chromium/optimize`:

```
CHROMIUM_FLAGS="${CHROMIUM_FLAGS} --ignore-gpu-blocklist"

CHROMIUM_FLAGS="${CHROMIUM_FLAGS} --enable-experimental-canvas-features --enable-accelerated-2d-canvas --canvas-msaa-sample-count=2 --force-display-list-2d-canvas --force-gpu-rasterization --enable-fast-unload --enable-accelerated-vpx-decode=3 --enable-tcp-fastopen --javascript-harmony --enable-checker-imaging --enable-zero-copy --ui-enable-zero-copy --enable-webgl-image-chromium --enable-accelerated-video --enable-gpu-rasterization --use-skia-deferred-display-list --use-skia-renderer"

CHROMIUM_FLAGS="${CHROMIUM_FLAGS} --use-vulkan"

CHROMIUM_FLAGS="${CHROMIUM_FLAGS} --v8-cache-options=code --v8-cache-strategies-for-cache-storage=aggressive"

CHROMIUM_FLAGS="${CHROMIUM_FLAGS} --enable-gpu-memory-buffers"
```

Запускаем chromium, заходим на страницу chrome://flags и включаем следующие опции:

```
#smooth-scrolling
#enable-quic
#enable-javascript-harmony
#enable-future-v8-vm-features
#enable-oop-rasterization
#enable-experimental-web-platform-features
#enable-accelerated-video-decode
#enable-zero-copy
#tab-groups
#enable-oop-rasterization-ddl
#decode-jpeg-images-to-yuv
#decode-webp-images-to-yuv
#enable-heavy-ad-intervention
```

Перезапускаем chromium, заходим на chrome://gpu и радуемся тому, что видим :)
