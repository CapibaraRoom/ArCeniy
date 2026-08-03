# Устройство grub

В качестве загрузчика системы используется **grub2**.

Для его установки необходимо поместить директории ```EFI/``` и ```boot/``` в ESP-раздел (раздел загрузки).
Их содержимое:

```
partition-1/                -- ESP-раздел
├── EFI/
│   └── BOOT/
│       └── bootx64.efi     -- загрузчик GRUB (собранный через grub-mkstandalone)
└── boot/
    └── grub/
        ├── arcsigfile      -- файл-сигнатуры для определения раздела с загрузчиком
        └── grub.cfg        -- основной конфиг GRUB (загружается с ESP)
```

Файл ```grub.cfg``` является текстовым; файл ```arcsigfile``` ничего не содержит. Сама структура загрузчика зашита в ```bootx64.efi``` и представляет из себя пространство из директорий с модулями и встроенным файлом конфигурации. Этот файл служит единственной цели - проводить поиск раздела с ```arcsigfile```, после которого он выполнит передачу управления пользовательской конфигурации.

## Сборка ```bootx64.efi```

Для сборки ```bootx64.efi``` необходимо создать дополнительный ```embedgrub.cfg```, который будет вшит в загрузчик. Содержимое этого файла:
```
search --set=root --file /boot/grub/artixsig
configfile /boot/grub/grub.cfg
```

Для работы загрузчика также необходимо включить модули его функционала:
```part_gpt ext2 fat search efi_gop efi_uga gfxterm```

Единый блок команд для сборки:
```
echo "search --set=root --file /boot/grub/artixsig
configfile /boot/grub/grub.cfg" > embedgrub.cfg

grub-mkstandalone -O x86_64-efi -o bootx64.efi \
  --modules="part_gpt ext2 fat search efi_gop efi_uga gfxterm" \
  "boot/grub/grub.cfg=embedgrub.cfg"
```
