# Знахідки, помилки й результат дослідів

Результати нижче історичні, за журналами 25–26.10.2026. Негативний дослід не перетворюється на PASS через подальший успішний boot.

## Build і transport

| Дослід / помилка | Фактичний результат і виправлення |
|---|---|
| Generic SD/Openwifi image як шлях для R2 | Відхилено: перевірений R2 boot використовує QSPI FIT і factory bootchain |
| Upstream підтримка R2 | Підтверджена у master software/HW/HW-image; issue508 лишалося відкритим, що не означало відсутності merged support |
| Збірка з supplied XSA | PASS; Linux/Buildroot build без нового Vivado synthesis |
| DTC1.7.2 з GCC15 | Build failure усунено scoped -std=gnu17, Werror збережений |
| U-Boot SPL alignment на Ubuntu26 uutils dd | conv=block,sync без cbs відхилялося; project-local Python доповнює тільки regular build artifact NUL bytes до кратності4. Приклад:137230→137232, незмінний prefix і два NUL |
| Executable permissions | Packager і USB init scripts отримали потрібний executable mode |
| DFU suffix | Початковий CRC неправильний; виправлено CRC inversion, незалежний dfu-suffix PASS |
| Повний ADI Git fetch / source acquisition | Тривалий download/fetch; остаточний kernel archive закріплений на точному commit; config replay перевірений |
| Native USB DFU | End-to-end transfer не підтверджений; не використано як єдиний recovery path |
| TFTP | На фактичному U-Boot не підтверджений; не підміняли припущенням із modern U-Boot source |
| YMODEM 921600 | 4-KiB trial, 64-KiB trial та повний FIT transfer PASS; перевірено exact length/CRC32 |
| U-Boot reset-code interface | Додано kernel interface для контрольованого повернення у збережений factory bootloader |
| Initial ARP без SSH | USB/LAN address confusion; окремий management subnet дав UART-керований доступ |

## RF bring-up і monitor

Неправильний reset GPIO67 залишився від factory HDL EMIO13, тоді як Openwifi wrapper використовує EMIO46/GPIO100. AD9361 RX CP calibration timeout зник після RAM overlay GPIO100; фінальний DTS містить reset100 та EN_AGC98. Clock40MHz і фізичний clock-mux не змінювали. Фінальний boot має PHY, ADC, DDS, XADC без overlay.

Monitor startup: wgd0 без повторного FPGA reload, sdr0 down → monitor → up → channel7 HT20 → AGC. Сам monitor не є гарантією пасивності: FPGA може автоматично відповідати ACK на адресовані пакети. S95 явно записує XPU register11=16 після interface-up та перевіряє readback. Після driver/interface reinitialization цей setting потрібно відновити. AP/hostapd та injector не запускали. Network TX counter=0 не є зовнішнім вимірюванням RF emissions.

## DDS ABI: підтверджена небезпечна діагностика

Openwifi driver/sdr.h копіював старий private layout cf_axi_dds_state. Linux6.12 додав поля й змінив layout; diagnostic читав неправильний version і garbage chip_info->name через %s. Виправлення: opaque forward declaration, лише exported kernel helper, diagnostic через dev_name(). Це усуває неправильний dereference без копіювання ще одного нестабільного private ABI. Call cf_axi_dds_datasel отримував оригінальний pointer і виконувався всередині kernel із власним layout.

Fix пройшов review, збірку, RAM reload та Flash boot. Початковий TX tuning failure виникає до Openwifi probe; ABI fix не оголошено його причиною або вирішенням. Причиною RX stall його також не доведено.

## FCS trailer і per-frame metadata

FPGA byte_to_word_fcs_sn_insert замінює останній байт FCS на {fcs_ok,rx_pkt_sn[6:0]}. Driver зберігає весь trailer й оголошує RX_INCLUDES_FCS. Через це звичайне порівняння усіх чотирьох tail bytes із CRC32 дає хибний висновок про 99/100 пошкоджених кадрів. Валідатор порівнює три збережені байти, hardware bit і radiotap; raw PCAP не переписує.

Після ABI-only firmware capture300 мав 218 radiotap-good /82 bad, 220 CRC24 matches, 298/300 metadata agreements. Дві суперечності були ACK довжиною14: expectedFCS ed1138c9, captured ed1138f3 та ed1138ba. Обидва повні CRC32 не збіглися; три збережені байти збіглися. Спеціального ACK exception не виявлено.

Source review знайшов rx_status, обнулений лише один раз на IRQ, тоді як IRQ loop обробляє кілька DMA buffers. BADFCS/short-GI/A-MPDU flags накопичувалися. Додано memset перед заповненням статусу кожного нового skb; packet bytes і DMA/filter/RF logic не змінювали. RAM gate351:302 good/49bad,302 CRC24,351 agreements,0drops. Фінальний Flash gate300:248good/52bad,248CRC24,300agreements,0drops. Повна 32-bit FCS validity цим не заявляється.

## TX timing і RX stall: відкриті питання

RX digital timing на40MHz має passing window. Internal TX PRBS/loopback має aggregate all-fail 2×16; обидва synthesizer VCO_LOCK встановлені. Driver fallback на delay defaults дозволяє startup; успіх IIO не доводить TX. Після контрольованого trial відновлено delay registers0x006=0x03 та0x007=0x50. Порівняння vendor/Openwifi TX pins, LVDS/framing і pinned ADI interface source не виявило певної mismatch. DDS_DISABLE не вимикає PRBS source9. DO_ODELAY не виправданий: DAC_IODELAY_ENABLE=0. Наступний діагностичний крок — per-channel PN_ERR/PN_OOS/source readback, а не випадковий delay sweep чи приховування warning. TX/AP/OTA transmission не кваліфіковані.

Після понад5h uptime новий capture50s отримав0 кадрів; RX counters і IRQ не змінювалися, image readback був правильним. Retune, interface reset і wgd/module reload не допомогли; full reboot відновив100 кадрів із0drops. Компонент-причина не встановлений. Довгий soak із відомим beacon source ще потрібний; при повторі знімати RF/IRQ/AXI-DMA state до reset та окремо перевіряти RF rebind/FPGA reload. Автоматичний reboot не додавали.

## Інші зафіксовані обмеження

- Native USB: factory host enumeration завершувалося error-71/disconnect; новий gadget не дав carrier. Фізична причина невідома; DEBUG UART і Ethernet працюють.
- До корекції RAM часу boot logs починалися1970; постійний NTP не налаштований. Не трактувати wall-clock цього boot як час експерименту без зовнішнього session timestamp.
- DHCP AP subnet init warning не потрібний static LAN/monitor і не доведений як причина RX stall. FAT dirty, unused ad7291/mwipcore warnings залишені для окремого розбору.
- Physical power-cycle, all-channel RF validation, long soak, AP throughput та зовнішнє RF measurement NOT_RUN.
