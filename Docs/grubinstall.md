# Устройство grub

В качестве загрузчика системы используется **grub2**.

Для его установки необходимо поместить директории ```EFI/``` и ```boot/``` в ESP-раздел (раздел загрузки).
Их содержимое:

```
partition-1/                -- ESP-раздел
├── EFI/
│   └── boot/
│       └── bootx64.efi     -- загрузчик GRUB (собранный через grub-mkstandalone)
└── boot/
    └── grub/
        ├── .arcsigfile     -- (опционально) файл-сигнатуры для определения раздела
        └── grub.cfg        -- основной конфиг GRUB (загружается с ESP)
```

Файл ```grub.cfg``` является текстовым; файл ```.arcsigfile``` ничего не содержит. Сама структура загрузчика зашита в ```bootx64.efi``` и представляет из себя пространство из директорий с модулями и встроенным файлом конфигурации. Этот файл служит единственной цели - проводить поиск раздела с ```.arcsigfile```, после которого он выполнит передачу управления пользовательской конфигурации.

## Сборка ```bootx64.efi```

Для корректной работы загрузчика рекомендуется найти шрифт ```unicode.pf2```. сделать это можно при помощи команды:
```
find /usr -name "unicode.pf2" 2>/dev/null
```

Для сборки ```bootx64.efi``` необходимо создать дополнительный ```grub.cfg```, который будет вшит в загрузчик. Содержимое этого файла:
```
loadfont /boot/grub/fonts/unicode.pf2
if search --set=root --file /boot/grub/.arcsigfile; then
    configfile /boot/grub/grub.cfg
else
    configfile (hd0,gpt1)/boot/grub/grub.cfg
fi
```

Для работы загрузчика также необходимо включить модули его функционала:
```part_gpt fat ext2 search gfxterm gfxterm_background png jpeg all_video```
.

Единый блок команд для сборки:
```
cat > grub.cfg << 'EOF'
loadfont /boot/grub/fonts/unicode.pf2
if search --set=root --file /boot/grub/.arcsigfile; then
    configfile /boot/grub/grub.cfg
else
    configfile (hd0,gpt1)/boot/grub/grub.cfg
fi
EOF

grub-mkstandalone -O x86_64-efi -o bootx64.efi \
  --modules="part_gpt fat ext2 search gfxterm gfxterm_background png jpeg all_video" \
  "boot/grub/grub.cfg=grub.cfg" \
  "boot/grub/fonts/unicode.pf2=/usr/share/grub/unicode.pf2"
```

После успешного создания ```bootx64.efi``` файл необходимо поместить в ```EFI/boot/``` в ```ESP-разделе```.
