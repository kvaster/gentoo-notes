# Настраиваем nvidia prime

Подготавливаем ядро: https://wiki.gentoo.org/wiki/NVIDIA/nvidia-drivers.
У меня было почти всё кроме ipmi handlers. А вот fb_efi не удалял - вроде и так работает.

В `/etc/portage/make.conf` дописываем:

```
VIDEO_CARDS="intel i965 nvidia"
```

В `package.use` добавляем:

```
x11-drivers/nvidia-drivers kms uvm gtk3
```

Собираем изменения мира: `emerge -avDuN @world`

Выполняем пункт 'Automated Setup' по ссылке: https://download.nvidia.com/XFree86/Linux-x86_64/440.64/README/dynamicpowermanagement.htmloptions

Опцию в nvidia.conf надо ДОБАВИТЬ, а не заменить полностью весь файл.

Настраиваем X11:

Создаём `/etc/X11/xorg.conf.d/01-nvidia-offload.conf`:

```
Section "ServerLayout"
  Identifier "layout"
  Option "AllowNVIDIAGPUScreens"
EndSection
```

Создаём `/etc/X11/xorg.conf.d/10-modesetting.conf`:

```
Section "Device"
  Identifier "Intel Graphics"
  Driver "modesetting"
  Option "Accel" "True"
  Option "AccelMethod" "glamor"
  Option "DRI" "3"
EndSection
```

Создаём `/etc/X11/xorg.conf.d/20-nvidia.conf`:

```
Section "Module"
  Load "modesetting"
EndSection

Section "Device"
  Identifier "nvidia"
  Driver "nvidia"
  BusID "PCI:1:0:0"
  Option "AllowEmptyInitialConfiguration"
EndSection
```

В последних версиях перестал работать suspend. Данная проблема описана [тут](https://bugs.gentoo.org/763129).
Для решения проблемы надо в `/etc/conf.d/nvidia` добавить: `NVreg_PreserveVideoMemoryAllocations=0`.

И финал: для того, чтобы запустить приложение на nvidia, используем следующий скрипт:

```
#!/bin/bash
__NV_PRIME_RENDER_OFFLOAD=1 __VK_LAYER_NV_optimus=NVIDIA_only __GLX_VENDOR_LIBRARY_NAME=nvidia "$@"
```
