# PlutoSky-R2: Openwifi, R&D checkpoint 26.10.2026

Це історичний звіт про досліди 25–26 жовтня 2026 року. Його укладено з журналів і контрольних сум, а не з нового вимірювання під час публікації. Результат: Openwifi завантажується з QSPI, автоматично запускає RX monitor на каналі 7 / 2442 МГц / HT20; автоматичні ACK вимкнені. TX/AP і тривала стабільність RX залишаються непідтвердженими.

## Матеріали

- [Плата, GPIO, clock і USB](board-and-usb.md).
- [Завантаження, запис і відновлення](recovery.md).
- [Знахідки, помилки та досліди](findings-and-errors.md).
- [Машиночитані результати](validation.json) і [provenance](provenance.json).
- Супровідний код і pipeline: [dscodetesla/openwifi, гілка chore/plutosky-r2-rnd-checkpoint-2026-10-26](https://github.com/dscodetesla/openwifi/tree/chore/plutosky-r2-rnd-checkpoint-2026-10-26). Посилання є цільовою гілкою checkpoint, а не доказом виконання окремого build.
- Цей звіт належить однойменній гілці [dscodetesla/PlutoSky-R2](https://github.com/dscodetesla/PlutoSky-R2/tree/chore/plutosky-r2-rnd-checkpoint-2026-10-26).

## Межі підтвердження

| Перевірка | Зафіксований результат |
|---|---|
| QSPI boot зі збереженим заводським FSBL/U-Boot | PASS |
| Linux 6.12.0-openwifi, LAN/SSH, PHY/ADC/DDS/XADC | PASS |
| S95 monitor, channel 7 HT20, XPU11=0x10 | PASS |
| Фінальний RX capture після Flash boot | 300 кадрів; 248 FPGA FCS good, 52 bad; 248 CRC24 matches; 300 узгоджених metadata; 0 kernel drops |
| RX digital timing, 40 МГц | Робоче вікно |
| RX і TX synthesizer PLL lock | VCO_LOCK на обох |
| TX digital timing, internal PRBS/loopback | FAIL: жодної з 32 перевірених позицій |
| Тривалий RX | Відкрито: після понад 5 годин був stall, reboot відновив RX |
| Native USB | Enumeration не підтверджене; DEBUG FTDI працює |
| RF TX, AP, throughput, зовнішня RF-відсутність TX | NOT_RUN / не кваліфіковано |

FPGA замінює останній байт прийнятого FCS на статус і 7-бітовий лічильник. CRC24 означає порівняння трьох збережених байтів із CRC32, обчисленим по MAC body. Це не незалежна перевірка повного 32-бітового FCS. Вихідні PCAP не змінювалися; у цей публічний checkpoint вони не включені.

## Відтворюваний порядок

1. Зафіксувати source pins і перевірити XSA SHA-256 з provenance.json. Використовувати окремі checkout для різних профілів Buildroot.
2. Прочитати й зберегти фактичні U-Boot environment, DTS, JEDEC/MTD layout; зробити повний 32-MiB backup. Самі mtd0–mtd3 заводського Linux покривали лише нижні 16 MiB.
3. Побудувати R2 firmware зі supplied XSA через ./buildroot-build.sh plutosky_r2 build. Компіляція існуючого XSA не потребує Vivado. Scoped host-build fixes описані в companion-гілці.
4. Перевірити FIT components, DFU suffix CRC, розмір і hashes; спочатку завантажити FIT у RAM через DEBUG UART/YMODEM, перевірити довжину та CRC32, виконати direct bootm.
5. Перевірити RF init, monitor, вимкнення ACK і capture. Виправлення GPIO100 спочатку підтверджене RAM overlay, після чого внесене в постійний DTS.
6. Записати лише перевірений qspi-linux із узгодженими partition size і fit_size; перевірити весь payload readback і незмінність bootchain/NVMFS. Виконати звичайний reboot і повторити runtime/capture gates.
7. Для наступних змін спочатку використовувати RAM module test, потім rebuild/package/readback і повторну перевірку зі Flash.

Наведені результати стосуються конкретних hashes. Загальна підтримка upstream або успішна компіляція самі по собі не підтверджують фізичну плату. Паролі, ключі, адреси пристроїв, raw capture та заводський Flash у публічний пакет не входять.
