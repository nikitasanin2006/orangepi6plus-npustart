## Инструкция по закуску NPU на ALT Linux под управлением Orange Pi 6 Plus

Текущие дистрибутивы ALT Linux не поддерживают управление NPU. Поскольку существует запрос на установку отечественного ПО, а также локальный запуск ИИ, вопрос о поддержке NPU встаёт сам собой. <br>

## Этап 1: Установка ОС

**Важно:** вам потребуются два дистрибутива: Ubuntu и ALT Рабочая станция 11. 

По ссылке [https://www.altlinux.org/Releases] загрузите ALT Рабочая станция версии 11 с ядром 6.12. После этого установите ОС, например, записав образ на SD-карту или SSD-диск.
**Примечание:** автор инструкции столкнулся с уже установленным дистрибутивом, потому такие параметры CIX System Manager, как режим загрузки, неизвестны. Уже установленный образ, однако, работает и с ACPI, и с Device Tree.
<br>
Далее, скачайте образ Ubuntu для Orange Pi 6 Plus по ссылке [https://drive.google.com/drive/folders/14vrKI_sFjvkravyDRIKskRYVMDfJB2kK] и распакуйте его на другой диск.
<br>
Оба диска должны быть вставлены в компьютер.

## Этап 2: Настройка ALT Linux

Поскольку вам потребуются исходники ядра (linux-headers), проще всего загрузить их последнюю доступную версию. Как правило, основная их версия (6.12) совпадает с версией ядра ALT. 
<br>
Выполните следующие команды:
```bash
sudo apt-get update
sudo apt-get dist-upgrade
sudo apt-get install -y git gcc make kernel-headers-modules-6.12 curl
```
Перезагрузите компьютер, затем выполните следующую команду: 
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```
uv потребуется для поднятия userspace с Python 3.12.

## Этап 3: Подготовка файлов

Вам потребуются следующее:
- Исходный код драйвера aipu-5.11.0 (находится в Ubuntu, обычно по пути /usr/src)
- Исходники используемого ядра ALT Linux (находится в ALT, обычно также по пути /usr/src)
- Дополнительные компоненты для работы с NPU (находятся в Ubuntu (в папке cix), обычно по пути /usr/share)

Скопируйте их в отдельную папку для удобства работы.

## Этап 4: Изменение файлов

Откройте консоль, перейдите в папку с драйвером aipu-5.11.0 и выполните следующий блок команд:

```bash
sed -i 's/static int default_remove(struct platform_device \*p_dev)/static void default_remove(struct platform_device *p_dev)/' \
    armchina-npu/default/default.c

sed -i 's/return armchina_aipu_remove(p_dev);/armchina_aipu_remove(p_dev);/' \
    armchina-npu/default/default.c

sed -i 's/static int sky1_npu_remove(struct platform_device \*p_dev)/static void sky1_npu_remove(struct platform_device *p_dev)/' \
    armchina-npu/sky1/sky1.c

python3 -c "
content = open('armchina-npu/sky1/sky1.c').read()
content = content.replace('#endif /* CONFIG_PM */\n\treturn 0;\n}', '#endif /* CONFIG_PM */\n}', 1)
open('armchina-npu/sky1/sky1.c', 'w').write(content)
"

python3 << 'EOF'
content = open('armchina-npu/sky1/sky1.c').read()
content = content.replace(
    'pre_freq = scmi_device_get_freq(cix_aipu_priv->opp_pmdomain);',
    'pre_freq = 0; /* disabled: not available in kernel 6.12 */')
content = content.replace(
    'ret = scmi_device_set_freq(cix_aipu_priv->opp_pmdomain, *freq);',
    'ret = 0; /* disabled: not available in kernel 6.12 */')
content = content.replace(
    '*freq = scmi_device_get_freq(cix_aipu_priv->opp_pmdomain);',
    '*freq = 0; /* disabled: not available in kernel 6.12 */')
content = content.replace(
    'stat->current_frequency = scmi_device_get_freq(cix_aipu_priv->opp_pmdomain);',
    'stat->current_frequency = 0; /* disabled: not available in kernel 6.12 */')
content = content.replace(
    'ret = scmi_device_opp_table_parse(cix_aipu_priv->opp_pmdomain);',
    'ret = 0; /* disabled: not available in kernel 6.12 */')
open('armchina-npu/sky1/sky1.c', 'w').write(content)
print("Done!")
EOF

```

Далее нужно отредактировать несколько файлов вручную. Ниже предоставлен список требуемых изменений.


armchina-npu/aipu.c

Добавлено на строке 29:
```c
#ifndef KMD_VERSION
#define KMD_VERSION "5.11.0"
#endif
```

armchina-npu/aipu_priv.c

Начиная со строки 80, следующее содержимое:

```c

#ifdef CONFIG_ARMCHINA_NPU_ARCH_V3_1
	if (version == AIPU_ISA_VERSION_ZHOUYI_V3_1) {
		aipu->ops = get_v3_1_priv_ops();
		aipu->core_reset_delay_us = AIPU_CONFIG_CORE_RESET_DELAY_US;
	}
#endif

#ifdef CONFIG_ARMCHINA_NPU_ARCH_V3
	if (version == AIPU_ISA_VERSION_ZHOUYI_V3)
		aipu->ops = get_v3_priv_ops();
#endif

#if (defined CONFIG_ARMCHINA_NPU_ARCH_V1) || (defined CONFIG_ARMCHINA_NPU_ARCH_V2)
	if (version > 0 && version <= AIPU_ISA_VERSION_ZHOUYI_V2_2)
		aipu->ops = get_v1v2_priv_ops();
#endif

	if (!aipu->ops) {
		ret = -EINVAL;
		dev_err(aipu->dev, "unidentified hardware version number: %d\n", version);
		goto finish;
	}

```
было изменено на:
```c
	aipu->ops = get_v3_priv_ops();
```


armchina-npu/zhouyi/v3.c
На строке 309 добавлено следующее содержимое (добавить в функцию static void zhouyi_v3_initialize(...)):
```c

	for (iter = 0; iter < partition->cluster_cnt; iter++) {
                u32 cid = partition->clusters[iter].id;
                u32 core_cnt = partition->clusters[iter].core_cnt;
                zhouyi_v3_set_partition(partition, cid);
                zhouyi_v3_enable_core_cnt(partition, cid, core_cnt);
        }
```

armchina-npu/sky1/sky1.c
На строке 212 изменён тип sky1_npu_devfreq_remove на void, в конце функции стоит return; вместо return 0;

Начиная со строки 414, следующее содержимое:
```c
#ifdef CONFIG_PM
    sky1_npu_pm_runtime_put(&p_dev->dev);
#endif /* CONFIG_PM */
```

было изменено на:
```c
#ifdef CONFIG_PM
    dev_info(&p_dev->dev, "skip pm_runtime_put after probe to keep NPU cores enabled\n");
#endif /* CONFIG_PM */
```

На строке 457 return 0; изменено на return;

## Этап 5: Установка драйвера

После изменения файлов выполните следующий блок команд, предварительно заменив path/to/ на нужные пути:
```bash
cd path/to/aipu-5.11.0/
export AIPU_INC="-Wno-error -I$(pwd)/armchina-npu/include -I$(pwd)/armchina-npu -I$(pwd)/armchina-npu/zhouyi"

make -C path/to/linux-6.12.112-6.12-alt1 M="$(pwd)" clean
make -C path/to/linux-6.12.112-6.12-alt1 M="$(pwd)" modules \
  BUILD_AIPU_VERSION_KMD=BUILD_ZHOUYI_V3 \
  BUILD_TARGET_PLATFORM_KMD=BUILD_PLATFORM_SKY1 \
  BUILD_NPU_DEVFREQ=n \
  EXTRA_CFLAGS="$AIPU_INC"


sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp -a path/to/aipu-5.11.0/aipu.ko /lib/modules/$(uname -r)/extra/aipu.ko
sudo depmod -a

sudo rmmod aipu 2>/dev/null || true
sudo insmod ./aipu.ko

sudo dmesg | grep -Ei 'aipu|armchina|cluster|scheduler' | tail -n 120
```

Ожидаемые строки вывода в конце:
```
[ 3669.242315] armchina CIXH4000:00: sky1_npu_probe: NPU core num is 3
[ 3669.243980] armchina CIXH4000:00: AIPU KMD (v5.11.0) probe start...
[ 3669.244431] armchina CIXH4000:00: GM policy is shared
[ 3669.244443] armchina CIXH4000:00: AIPU is behind an IOMMU
[ 3669.332476] armchina CIXH4000:00: init MEMORY region done: iova [0x780000000, 0x7bffffffe]
[ 3669.403069] armchina CIXH4000:00: init MEMORY region done: iova [0x740000000, 0x77ffffffe]
[ 3669.471590] armchina CIXH4000:00: init MEMORY region done: iova [0x700000000, 0x73ffffffe]
[ 3669.560655] armchina CIXH4000:00: init MEMORY region done: iova [0x680000000, 0x6bffffffe]
[ 3669.630778] armchina CIXH4000:00: init MEMORY region done: iova [0x640000000, 0x67ffffffe]
[ 3669.699370] armchina CIXH4000:00: init MEMORY region done: iova [0x600000000, 0x63ffffffe]
[ 3669.699383] armchina CIXH4000:00: set ASID 0 done: base: 0x780000000, range 0x3fffffff
[ 3669.699386] armchina CIXH4000:00: set ASID 1 done: base: 0x740000000, range 0x3fffffff
[ 3669.699389] armchina CIXH4000:00: set ASID 2 done: base: 0x700000000, range 0x3fffffff
[ 3669.699391] armchina CIXH4000:00: set ASID 3 done: base: 0x680000000, range 0x3fffffff
[ 3669.699393] armchina CIXH4000:00: set ASID 4 done: base: 0x640000000, range 0x3fffffff
[ 3669.699395] armchina CIXH4000:00: set ASID 5 done: base: 0x600000000, range 0x3fffffff
[ 3669.699397] armchina CIXH4000:00: driver mem management is enabled
[ 3669.699507] armchina CIXH4000:00: AIPU detected: zhouyi-v3
[ 3669.699789] armchina CIXH4000:00: configure cluster #0 done: en_core_cnt 3 (0x1314)
[ 3669.699796] armchina CIXH4000:00: ############# ZHOUYI V3 AIPU #############
[ 3669.699797] armchina CIXH4000:00: # Maximum Partition Count: 1
[ 3669.699798] armchina CIXH4000:00: # Maximum Command Pool Count: 1
[ 3669.699800] armchina CIXH4000:00: # Enabled Partition Count: 1
[ 3669.699801] armchina CIXH4000:00: # Enabled Cluster Count: 1
[ 3669.699802] armchina CIXH4000:00: #
[ 3669.699803] armchina CIXH4000:00: # -- Partition #0
[ 3669.699804] armchina CIXH4000:00: # Cluster Count: 1
[ 3669.699805] armchina CIXH4000:00: # Core Count per Cluster: 3
[ 3669.699806] armchina CIXH4000:00: # TEC Count per Core: 4
[ 3669.699807] armchina CIXH4000:00: ##########################################
[ 3669.699809] armchina CIXH4000:00: sky1_npu_probe: armchina_aipu_probe done
[ 3669.699811] armchina CIXH4000:00: skip pm_runtime_put after probe to keep NPU cores enabled
```

Если всё прошло успешно, **добавьте драйвер в автозагрузку:**
```bash
sudo printf 'aipu\n' > /etc/modules-load.d/aipu.conf
sudo modprobe aipu || sudo insmod /lib/modules/$(uname -r)/extra/aipu.ko

sudo uname -r
sudo lsmod | grep aipu
sudo ls -l /dev/aipu
sudo dmesg | grep -i aipu | tail -n 100
```

<br>
Готово! Вы успешно сделали поддержку NPU на ALT Linux под управлением Orange Pi 6 Plus!

## Полезные материалы
Настройка userspace: [instructions/userspace.md]
Запуск моделей: [instructions/model_start.md]
Часто задаваемые вопросы: [instructions/faq.md]

Автор оригинальной инструкции: [https://github.com/stavrmoris/]

