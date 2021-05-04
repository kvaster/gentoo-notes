# Использование pipewire вместо pulseaudio

Настраиваем `pipewire` и `pulseaudio` для правильной работы с BT первого в файле `/etc/portage/package.use/...`:

```
media-sound/pulseaudio -bluetooth -alsa-plugin
net-wireless/bluez deprecated extra-tools experimental
media-video/pipewire aac aptx bluetooth extra gstreamer ldac v4l pipewire-alsa
```

Пересобираем `pulseaudio` и собираем `pipewire`.

Для правильной работы `pipewire` убираем `pulseaudio` из автозапуска:

В файле `/etc/xdg/autostart/pulseaudio.desktop` изменяем:

```
;Exec=start-pulseaudio-x11
```

В файле `/etc/pulse/client.conf` изменяем:

```
autospawn = no
```

Добавляем в автозапуск в `~/.xprofile`:

```
pipewire &
```

Редактируем настройки сессии в `/etc/pipewire/media-session.d/media-session.conf`:

```
session.modules = {
  default = [
    ...
    bluez5
    logind
    ...
  ]
}
```

Редактируем настройки bluetooth в `/etc/pipewire/media-session.d/bluez-monitor.conf` с учётом того,
что HSP профиль вообще не влючаем (он deprecated и многие наушники с ним плохо работают):

```
properties = {
  bluez5.sbc-xq-support = true
  bluez5.headset-roles = [ hfp_hf hfp_ag ] # hsp не включаем
}

rules = [
  {
      ...
      actions = {
        update-props = {
          ...
          bluez5.reconnect-profiles = [ hfp_hf a2dp_sink hfp_ag a2dp_source ] # hsp опять не включаем
          bluez5.msbc-support = true
        }
      }
      ...
  }
]
```

Настройки готовы и мы можем просто ребутнуться / перезапустить сессию.

## Переключение между a2dp (хорошее качество для слушания) на hfp (handsfree режим)

Следующим скриптом можно переключать быстро в разные режиме BT наушники:

```
#!/bin/sh

if [ "$1" == "mic" ]; then
  profile="headset-head-unit-msbc"
else
  profile="a2dp-sink-aptx"
fi

card=`LC_ALL=en_US pactl list cards | grep bluez_card -m1 | sed -rn "s/^.*(bluez_card[^ ]+)/\1/p"`
pactl set-card-profile $card $profile
```

## Автоматическое переключение на проводные наушники при подключении

У себя я заметил, что перестало работать автоматическое переключение на проводные наушники при их подключении.
Вылечить такое смог с помощью изменения alsa профила в `/usr/share/alsa-card-profile/mixer/paths/analog-output-headphones.conf`.
В этом файле надо поменять `priority` на 101 вместо 99 - чтобы наушники были важнее обычного выхода, у которого `priority` выставлен в 100.
