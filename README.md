# Встроенный микрофон в Ubuntu на HP EliteBook (Bang & Olufsen)

> **English summary:** fix for the built-in digital microphone (DMIC) on HP EliteBook laptops
> with Bang & Olufsen audio and Intel CPUs under Ubuntu. The legacy `snd_hda_intel` driver can't use the DMIC,
> so this switches to the Intel **AVS** driver (`dsp_driver=4`), installs the missing AVS topology files
> from upstream linux-firmware, sets `DMIC Volume` (defaults to 0!) and adds a WirePlumber rule
> against stream stalls. Copy-paste blocks below; rollback included.

## Кому это поможет

У вас **встроенный микрофон «есть», но записывает тишину**, а динамики работают.
Проверено на ноутбуке **HP EliteBook** со звуком **Bang & Olufsen** в Ubuntu.
Скорее всего подойдёт и для других ноутбуков на Intel с цифровым микрофоном.

**Как проверить, что это ваш случай.** Вставьте в терминал:

```bash
journalctl -k -b | grep -iE "Digital mics found|NHLT" ; lspci | grep -i audio
```

Если видите `NHLT` или `Digital mics found`, а в строке аудио есть `Sunrise Point` или `Kaby Lake`,
инструкция для вас.

## Что делает инструкция

Включает драйвер **AVS**, который умеет работать со встроенным цифровым микрофоном,
скачивает для него 3 маленьких файла из официального репозитория
[linux-firmware](https://gitlab.com/kernel-firmware/linux-firmware/-/tree/main/intel/avs)
и настраивает звук, чтобы не было зависаний.

Главная ловушка, из-за которой микрофон пишет тишину даже с правильным драйвером:
громкость `DMIC Volume` по умолчанию стоит на **0**. Шаг 2 это исправляет.

---

## Как пользоваться

1. Откройте **Терминал** (`Ctrl + Alt + T`).
2. Скопируйте **весь** серый блок из шага 1 целиком и вставьте в терминал
   (`Ctrl + Shift + V`), нажмите **Enter**.
3. Введите пароль, когда попросит (буквы при вводе не видны, это нормально).
4. Дождитесь, пока команды закончатся. Ноутбук перезагрузится сам.
5. После перезагрузки сделайте шаг 2.

---

## Шаг 1. Установка (потом ноутбук перезагрузится сам)

```bash
sudo bash -c '
set -e
mkdir -p /lib/firmware/intel/avs
cd /lib/firmware/intel/avs
for f in dmic-tplg.bin hda-generic-1ep-tplg.bin hda-808628xx-3ep-tplg.bin; do
  if [ ! -e "$f" ] && [ ! -e "$f.zst" ] && [ ! -e "$f.xz" ]; then
    curl -fL -o "$f" "https://gitlab.com/kernel-firmware/linux-firmware/-/raw/main/intel/avs/$f"
  fi
done
[ -e hda-generic-tplg.bin ] || [ -e hda-generic-tplg.bin.zst ] || ln -s hda-generic-1ep-tplg.bin hda-generic-tplg.bin
[ -e hda-8086-generic-tplg.bin ] || [ -e hda-8086-generic-tplg.bin.zst ] || ln -s hda-808628xx-3ep-tplg.bin hda-8086-generic-tplg.bin

rm -f /etc/modprobe.d/blacklist-dmic.conf
sed -i -E "/dmic_detect=0|snd-intel-dspcfg/d" /etc/modprobe.d/*.conf 2>/dev/null || true

echo "options snd-intel-dspcfg dsp_driver=4" > /etc/modprobe.d/snd-intel-dspcfg.conf
update-initramfs -u
'

if wireplumber --version 2>&1 | grep -q "libwireplumber 0.4"; then
  mkdir -p ~/.config/wireplumber/main.lua.d
  cat > ~/.config/wireplumber/main.lua.d/51-avs-fix.lua <<'EOF'
table.insert(alsa_monitor.rules, {
  matches = { { { "node.name", "matches", "alsa_*.platform-avs_*" } } },
  apply_properties = {
    ["session.suspend-timeout-seconds"] = 0,
    ["api.alsa.headroom"] = 1024,
    ["audio.rate"] = 48000,
  },
})
EOF
else
  mkdir -p ~/.config/wireplumber/wireplumber.conf.d
  cat > ~/.config/wireplumber/wireplumber.conf.d/51-avs-fix.conf <<'EOF'
monitor.alsa.rules = [
  {
    matches = [ { node.name = "~alsa_.*platform-avs_.*" } ]
    actions = {
      update-props = {
        session.suspend-timeout-seconds = 0
        api.alsa.headroom = 1024
        audio.rate = 48000
      }
    }
  }
]
EOF
fi

sudo reboot
```

---

## Шаг 2. После перезагрузки включить микрофон

Скопируйте и вставьте весь блок:

```bash
amixer -c DMIC cset name='DMIC Volume' 2147483647
wpctl set-volume @DEFAULT_AUDIO_SOURCE@ 2.5
wpctl set-mute @DEFAULT_AUDIO_SOURCE@ 0
sudo alsactl store
```

Проверьте: **Настройки → Звук → Вход**. Скажите что-нибудь, полоска должна двигаться.

---

## Если что-то не так

### Микрофон слишком тихий / слишком громкий

Громче (350%):
```bash
wpctl set-volume @DEFAULT_AUDIO_SOURCE@ 3.5
```

Тише (180%):
```bash
wpctl set-volume @DEFAULT_AUDIO_SOURCE@ 1.8
```

### Звук или микрофон завис (тишина, ничего не играет)

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

### После шага 1 звука нет совсем: вернуть как было

Динамики снова заработают, но встроенный микрофон нет
(тогда используйте USB-микрофон или гарнитуру).

```bash
echo "options snd-intel-dspcfg dsp_driver=1" | sudo tee /etc/modprobe.d/snd-intel-dspcfg.conf && sudo update-initramfs -u && sudo reboot
```

---

## Почему микрофон не работает «из коробки» и как я это выяснял

Расскажу по порядку, как было у меня. Может, кому-то сэкономит полдня.

### Сначала было так

Ставишь Ubuntu, динамики играют, а микрофон вроде бы есть в настройках, но не пишет вообще
ничего. Не тихо, а именно ноль. Я сначала думал, что он просто выключен (он и правда был на mute,
легко задеть клавишу с перечёркнутым микрофоном), но после включения ничего не поменялось.
Тестовая запись: сплошные нули.

### В чём настоящая причина

Микрофон, который видит Ubuntu по умолчанию, это **вход для гарнитуры** (3,5 мм), а не встроенный.
Встроенные микрофоны в этих EliteBook **цифровые** (их называют DMIC) и подключены не к звуковому
чипу, а напрямую к процессору Intel, к его аудио-DSP.

Ubuntu по умолчанию грузит старый, «обычный» драйвер `snd_hda_intel`. Он прекрасно работает
с динамиками и наушниками, но цифровые микрофоны **не умеет вообще**. Поэтому микрофон как бы есть,
а звука с него нет.

Цифровые микрофоны умеет драйвер **AVS** (`snd_soc_avs`). Он уже есть в ядре, но:

1. **Он не включён.** Нужно явно сказать системе: `dsp_driver=4`.
2. **Ему не хватает файлов.** Драйверу нужны так называемые «топологии», маленькие файлы,
   которые описывают, как идёт звук внутри DSP. В пакете `linux-firmware` у Ubuntu их нет,
   а в официальном репозитории linux-firmware они уже лежат. Без них AVS запускается
   и тут же падает, и тогда пропадает **весь** звук, а не только микрофон.
   Нужны три файла:
   - `dmic-tplg.bin` для цифровых микрофонов;
   - `hda-generic-1ep-tplg.bin` для динамиков и наушников (драйвер ищет его под именем
     `hda-generic-tplg.bin`, поэтому делаем ссылку);
   - `hda-808628xx-3ep-tplg.bin` для звука по HDMI (ищется как `hda-8086-generic-tplg.bin`).
3. **Громкость цифрового микрофона стоит на нуле.** Вот это меня добило: драйвер заработал,
   микрофон появился, а запись всё равно ноль. Оказалось, у него есть свой регулятор
   `DMIC Volume`, и по умолчанию он на 0. Подняли на максимум, и всё, голос пошёл.

### На что я потратил время зря (чтобы вы не тратили)

- **Старые «советы из интернета».** У меня в `/etc/modprobe.d/` лежала куча файлов от прошлых
  попыток: `dmic_detect=0`, чёрный список `snd-soc-dmic`, `model=dell-headset-multi` (настройка
  вообще для Dell!), ручное переназначение пинов звукового чипа. Всё это только мешало.
  Инструкция выше эти файлы убирает.
- **Подвох с `dmic_detect=0`.** Если просто удалить эту строку, система сама переключится на AVS,
  а файлов-топологий нет, и звук пропадёт полностью. Поэтому сначала файлы, потом драйвер.
- **Пугающие строчки в логе** `Endpoint format configuration not found`. Это не ошибка.
  Драйвер перебирает 8 вариантов формата, а ноутбук поддерживает один (4 канала, 16 бит, 48 кГц).
  Про остальные семь он и пишет. Я это понял, только когда разобрал таблицу NHLT из BIOS
  и заглянул в исходники драйвера.

### Что я проверял по ходу

- `journalctl -k`: какой драйвер загрузился и какие файлы он не нашёл;
- `arecord -l` и `wpctl status`: какие устройства записи видит система;
- таблицу **NHLT** из BIOS (`/sys/firmware/acpi/tables/NHLT`), там написано, какие микрофоны
  стоят и в каких форматах они работают;
- пробные записи по 3 секунды напрямую с устройства (`arecord -D hw:...`) и через PipeWire,
  считал уровень сигнала, чтобы видеть, ноль там или звук.

### Почему потом зависал звук

После перехода на AVS иногда случалось так: включаешь запись, пропадает звук, или наоборот.
В логах PipeWire было `Обрыв канала` на выходе динамиков. Похоже, драйвер AVS плохо переживает,
когда устройство «засыпает» в паузе, а потом просыпается, или когда программа меняет частоту.
Поэтому в инструкции есть настройка для WirePlumber: не усыплять устройства, держать одну частоту
48 кГц и дать небольшой запас буфера. После этого всё стало стабильно.

### А почему на Windows было лучше с басами

На Windows вместе с драйвером стоит программа Bang & Olufsen Audio Control. Она незаметно
поднимает басы, выравнивает звук под маленькие динамики и не даёт им перегружаться.
В Linux такой обработки нет, динамики получают звук «как есть». Отсюда и «пропали басы»,
и хрип на большой громкости. Это не поломка, просто нет обработки. EasyEffects частично
это возвращает.

---

## Что важно знать

- **Не ставьте громкость динамиков на максимум.** В Linux нет фирменной обработки
  Bang & Olufsen, и на громком звуке с басами динамики хрипят и могут испортиться.
- **Басы** можно вернуть программой EasyEffects (`sudo apt install easyeffects`):
  во вкладке «Выход» добавьте «Усилитель басов» и обязательно «Ограничитель».
- **Если в новой Ubuntu микрофон работает сразу**, ничего делать не нужно.
- Инструкция проверена на свежей Ubuntu. В будущих версиях
  что-то может измениться. Если не помогло, используйте блок «вернуть как было».
- Вы запускаете команды с `sudo` на свой страх и риск. Перед запуском можно прочитать,
  что делает каждая строка, всё описано выше.

Помогло или нашли ошибку? Откройте Issue, это поможет другим.
