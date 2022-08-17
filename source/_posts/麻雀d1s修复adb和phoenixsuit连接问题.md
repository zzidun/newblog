---
title: 麻雀d1s修复adb和phoenixsuit连接问题
categories:
  - 经验和总结
tags:
  - 全志d1s
  - tina
  - 麻雀开发板
  - riscv
abbrlink: a30c2c5a
date: 2022-02-08 13:22:55
---

* 修复pc连接电脑掉电问题;
* 修复phoenixsuit连接问题;

<!-- more -->

## 修复pc连接电脑后掉电问题

官方的`tina`编译出来之后,连接电脑,如果使用了`otg`口,会导致这个开发板不断的断开和重连.

而且串口不断地打印断连重连的信息.有这个报错信息:

> WARN: get power supply failed\n

看起来是和供电有关的问题,我在网上找到[这个文章](https://blog.csdn.net/qq_45362097/article/details/120710425).

作者说这是因为开发板连接`usb`之后会把某个电流限制到`500ma`.我们可以改一下

在文件`tina-d1-open/lichee/linux-5.4/drivers/usb/sunxi_usb/udc/sunxi_udc.c`中,修改函数`sunxi_set_cur_vol_work`.

```c
void sunxi_set_cur_vol_work(struct work_struct *work)
{
#if !defined(SUNXI_USB_FPGA) && defined(CONFIG_POWER_SUPPLY)
        struct power_supply *psy = NULL;
        union power_supply_propval temp;

        if (of_find_property(g_udc_pdev->dev.of_node, "det_vbus_supply", NULL))
                psy = devm_power_supply_get_by_phandle(&g_udc_pdev->dev,
                                                     "det_vbus_supply");

        if (!psy || IS_ERR(psy)) {
                DMSG_PANIC("%s()%d WARN: get power supply failed\n",
                           __func__, __LINE__);
        } else {
                temp.intval = 2500; //改为2500ma

                power_supply_set_property(psy,
                                        POWER_SUPPLY_PROP_INPUT_CURRENT_LIMIT, &temp);
        }
#endif
}
```

改完之后编译打包,这个现象不会出现了.

## 修复phoenixsuit连接问题

修改文件`tina-d1-open/lichee/brandy-2.0/u-boot-2018/drivers/sunxi_flash/mmc/sdmmc.c`中的`sunxi_sprite_mmc_probe`函数:

```c
int sunxi_sprite_mmc_probe(void)
{
#ifndef CONFIG_MACH_SUN50IW11
        return sdmmc_init_for_sprite(0, 0); //修改了这里.
#else
        int workmode = uboot_spare_head.boot_data.work_mode;
        if (workmode == WORK_MODE_CARD_PRODUCT)
                return -1;
        else
                return sdmmc_init_for_sprite(0, 0);
#endif
}
```

改了之后就可以直接用phoenixsuit来刷机.而不需要使用读卡器.