# АрСений [ArCeniy]

Тут пока пусто, но скоро появится полноценная операционная система на базе porteus+arch с простой установкой.

Прошу немного подождать :wink:


## Целевая структура

```
partition-1/                               -- загрузочный раздел
└── grub/                                  -- система загрузки
    └── -> partition-2/filesystem/iso/     -- ссылка на систему запуска из iso
        ├── vmlinuz
        └── initrd.xz

partition-2/          -- раздел с файловой системой
└── filesystem/       -- директория файловой системы (или корень раздела)
    ├── kernel/       -- модули ядра (дополнения, а не элементы запуска)
    ├── iso/          -- нераспакованый iso-образ (ядро, initrd и ФС дистрибутива)
    ├── modules/      -- модули для дополнения системы
    └── sigfile       -- (опционально) файл для определения точки загрузки
```

## Альтернативные проекты

Вас так же могут заинтересовать следующие работы:
  - Porteus: [форум](https://forum.porteus.org/), [зеркало](https://mirrors.dotsrc.org/porteus/x86_64/) - популярный модульный дистрибутив.
  - Nemesis: [описание](https://distrowatch.com/table.php?distribution=nemesis), [зеркало](https://sourceforge.net/projects/nemesis-linux/) - "форк" porteus с использованием artix.
  - Puzzle: [вики](https://wiki.puppyrus.org/users_os/puzzle) - "форк" porteus с использованием arch.
  - Arch (base): [вики](https://archlinux.org/download/) - популярный дистрибутив с собственной архитектурой и пакетным менеджером pacman.
  - Artix: [описание](https://artixlinux.org/), [зеркало](https://mirrors.dotsrc.org/artix-linux/iso/) - дистрибутив на базе arch с удаленным systemd.
