# RAM boot, QSPI install і recovery

Це протокол відтворення досліду, не універсальний flash-script. Перед кожним записом звірити фізичну плату, фактичний JEDEC, partition offsets/sizes, backup і U-Boot environment. Не підставляти номер mtd без перевірки його назви й offset.

## Layout перевіреної плати

| Offset | Розмір | Призначення |
|---|---|---|
| 0x000000 | 0x100000 / 1 MiB | Заводські FSBL + U-Boot, mtd0 |
| 0x100000 | 0x020000 / 128 KiB | U-Boot environment, mtd1 |
| 0x120000 | 0x0e0000 / 896 KiB | NVMFS/spare, mtd2 |
| 0x200000 | Factory: 0x0e00000 / 14 MiB; Openwifi: 0x1e00000 / 30 MiB | qspi-linux, mtd3 |

Повний 32-MiB backup, partition backups і контрольні суми перевірені та зберігаються приватно. У checkpoint не включені заводські binaries, environment contents, NVMFS або credentials. Копія лише factory mtd0–3 не охоплює верхні 16 MiB. Перед першим 30-MiB erase цю область також зберегли з RAM-booted нового kernel.

## Спочатку RAM

Фактичний заводський U-Boot підтримував loady, bootm, crc32; TFTP path не підтверджений. Його SF і RAM DFU mappings були обмежені 14 MiB, хоча опублікований vendor source містив 30-MiB defaults. Фактичний printenv має пріоритет перед README або source defaults.

1. Через DEBUG UART зафіксувати U-Boot version, bdinfo, printenv і доступні команди. Перевірити, що RAM load region не перекриває relocated U-Boot, stack або інші payloads.
2. loady 0x2080000 921600 запускається на початкових 115200. Після повідомлення switch змінити host UART на 921600 і надіслати CR. Один reader контролює UART; logger не повинен споживати байти YMODEM.
3. Передати один FIT через lrzsz sb -b -k --ymodem. 4-KiB та 64-KiB trials пройшли з точними довжинами/CRC32; -k дає 1024-byte STX packets, які приймає vendor xyzModem.
4. Після transfer U-Boot автоматично повертає початкові 115200 й очікує ESC. Занадто ранній ESC у досліді губився під час завершення transfer; після завершення drain повторний ESC відновлював prompt. baudrate environment не змінювали.
5. Перевірити exact filesize, незалежний host/U-Boot CRC32 та iminfo. Успішний exit loady сам по собі недостатній: vendor wrapper може не передати partial-transfer error.
6. Встановити bootargs лише в RAM і виконати direct bootm 0x2080000#config@0. FIT config@0–10 містили ті самі компоненти. У trial не викликати saveenv, adi_loadvals, qspiboot або flash commands: деякі factory macros можуть записувати environment.

Перезавантаження після RAM trial повертало factory firmware, доки не виконали окремий контрольований запис. Усі rootfs зміни RAM trial є тимчасовими.

## FIT, DFU і фактичний запис

R2 packager поєднує system_top.bit, zImage, DTB та rootfs.cpio.gz у FIT. Верхня межа FIT payload — 0x1e00000; DFU додає 16-byte suffix. Перевіряти довжину кінцевого файлу окремо: payload-cap сам не гарантує, що payload+suffix вмістяться у raw MTD partition.

Виправлено інверсію DFU suffix CRC; незалежний dfu-suffix прийняв фінальний файл. FIT для RAM boot і raw payload контролюється окремими hashes. DFU suffix не є 802.11 FCS і не пов'язаний із RX capture trailer.

З нового RAM kernel перевірено повний 32-MiB chip і qspi-linux 30 MiB. Зібраний target tools/mtd_flash_write перевіряє MEMGETINFO і розмір, стирає весь вибраний partition, записує файл і порівнює readback. Сам upstream Buildroot не встановлював цей helper автоматично. Він не перевіряє model/назву partition і не змінює fit_size. В усіх інсталяціях записували лише qspi-linux; bootchain та NVMFS звіряли незалежними hashes.

Factory read_sf читає fit_size. Якщо він замалий, не покладатися на fallback iminfo: частковий FIT може пройти структурну перевірку. fit_size спочатку цілеспрямовано узгодили з payload через factory U-Boot та перевірили environment CRC/readback. У фінальній ітерації залишено 0x14405a8 = 21235112 байтів; FIT має 21235108 байтів, додаткові 4 байти стерті. Environment у цій останній ітерації не переписували.

Після кожного запису перевіряли payload readback, module hash у firmware, normal Flash boot, чотири IIO devices, monitor/channel/ACK і RX capture. Не вважати RAM module reload доказом, що той самий модуль пережив reboot: це окремий gate.

## Відновлення

Нормальний rollback використовує перевірений приватний backup і правильний firmware offset, збережений заводський U-Boot та узгоджений fit_size. Повернення factory Linux також повертає його старе уявлення про partition layout; 21-MiB Openwifi image не можна прошивати через старий 14-MiB mapping. Не створювати default environment поверх збереженого без окремої потреби.

Upstream scripts для from-source QSPI bootchain replacement відомі як такі, що не завантажуються на цій платі; їх у цьому досліді не прошивали. Не замінювати offset0 заради звичайного Openwifi update. Повне 32-MiB відновлення і firmware-only rollback — різні операції: перше перезаписує також bootchain, environment і NVMFS та потребує окремого обґрунтування.

Немає A/B rollback. Стабільне живлення, readback і доступний DEBUG console потрібні під час явного flash operation. Recovery binaries і приватні backups у публічний репозиторій не публікувати.
