# Плата, GPIO, reference clock і USB

## Ідентифікація та межі

Перевірений runtime: XC7Z020, 1 GiB DDR, Linux 6.12.0-openwifi, supplied XSA для xc7z020clg484-2 / Vivado 2022.2. AD9361 підтверджений конфігурацією й роботою драйвера; фізичне маркування AD9361/AD9363 не оглянуте. Назва model у DT не замінює фізичної ідентифікації.

Flash визначений за JEDEC ef4019 як Winbond w25q256, 32 MiB. Заводський DTS експонував лише нижні 16 MiB; це обмеження partition description, а не доказ 16-MiB мікросхеми.

## GPIO: підтверджена помилка порту

У supplied Openwifi wrapper gpio_i/o/t мають ширину 64 і підключені до PS EMIO. Для Zynq GPIO controller EMIO0 відповідає offset54. Не плутати ці controller offsets із динамічними глобальними Linux GPIO numbers.

| Сигнал | Factory HDL | Openwifi R2 HDL | Factory/static R2 DTS до fix | Виправлений R2 DTS |
|---|---|---|---|---|
| AD9361 reset | EMIO13 → GPIO67 | EMIO46 → GPIO100 | 67 | 100 |
| EN_AGC | EMIO12 → GPIO66 | EMIO44 → GPIO98 | 66 | 98 |

До fix AD9361 probe завершувався -110 із Calibration TIMEOUT (0x244,0x80). ADI definitions: 0x244 = RX_CAL_STATUS, bit7 = CP_CAL_VALID; це RX synthesizer charge-pump calibration, не LVDS timing. RAM configfs overlay для reset GPIO100 усунув timeout; з'явилися PHY/ADC/DDS разом із XADC. Виправлення потім внесене до постійного DTS. Reference clock при цьому не змінювали.

EN_AGC mapping приведений у відповідність HDL, але pinned ADI driver не запитує en_agc GPIO як керований GPIO descriptor. Саме виправлення reset, а не довільний AGC tuning, підтверджене як вирішення probe failure. Фінальний boot не залежить від тимчасового overlay.

## Clock і FPGA provenance

Factory і Openwifi DTS задають external-reference input 40 MHz та adi,xo-disable-use-ext-refclk-enable. Це властивість джерела для RF chip; вона не означає, що користувач обов'язково має подати зовнішній лабораторний генератор. Фізичну reference частоту окремим приладом не вимірювали.

Factory Rev.C DTS містить дві альтернативні MIO48 clock hog: external output-high та internal output-low. Vendor U-Boot повинен видалити непотрібну, але опублікований код робить це лише для model Z7010/AD9363, тоді як R2 model є Z7020/AD9361. Не переносити обидві hog без перевірки фактичного booted DT та GPIO state. Успішний reset-only дослід не доводить потреби змінювати clock mux.

Openwifi ADF4001 init використовує divider8 від i_clk та слова 1F8093, 1F8092, 000004, 000401; порівняння vendor source не виявило відмінності цих слів. Board constraint для i_clk задає 25 ns / 40 MHz. TX clock/frame/data pin assignments, LVDS standard і framing properties збіглися з vendor source; певної TX mismatch не знайдено.

XSA HWH підтверджує 64-bit EMIO, LVDS, 2R2T, DAC_CLK_EDGE_SEL=0, DAC_INIT_DELAY=0, DAC_IODELAY_ENABLE=0, DAC_DDS_DISABLE=1. XSA не містить top HDL або implemented DCP: pad mapping спирається на recorded source commit, не на незалежне reverse engineering bitstream. Вимкнений DDS не вимикає окреме PRBS source9 для цифрового TX test.

## USB і LAN

| Фізичний інтерфейс | Роль і фактичний результат |
|---|---|
| Native USB-C Zynq | Gadget/DFU/RNDIS залежно від firmware; успішна host enumeration не підтверджена |
| USB-C DEBUG | Незалежний FTDI UART/JTAG; UART використаний, JTAG programming не виконували |
| RJ45 | Надійний management path для SSH, IIO і передачі образів/capture |

Factory USB-network subnet відрізнявся від factory Ethernet subnet. Спроба доступу до USB address через LAN дала ARP без робочого TCP/ICMP; це не доводило стан Linux і створювало асиметричну маршрутизацію. Тимчасова окрема Ethernet subnet відновила керований доступ; Openwifi надалі використовував static LAN без default gateway на management link.

Native USB не з'явився після gadget unbind/rebind. Пізніше UDC був powered, speed UNKNOWN, usb0 NO-CARRIER. Host journal містив коротке factory enumeration з error -71 і disconnect ще до Openwifi. Звіти configured у попередній спробі не дорівнюють підтвердженому end-to-end link. Причина не локалізована до кабелю, порту, PHY або плати.

Наступні фізичні перевірки: інший перевірений USB2 data-кабель, потім окремо інший прямий host port. Критерій: vendor/product 0456:b673, стабільна enumeration, відома speed, UDC configured і carrier usb0. Не змінювати одночасно кілька чинників.
