# U-boot-mod-for-RT5350

# 打补丁

先下载这个[[noblepepper/ralink_sdk: RT5350 linux SDK from Ralink (github.com)](https://github.com/noblepepper/ralink_sdk)](链接)

使用patch给其中的Uboot打补丁



# 编译及烧录



环境：Ubuntu 22.04.4 LTS

## 1、make menuconfig

    Cross Compiler Path: 输入toolchain/bin的绝对路径（以/为开头的绝对路径）
     (RT5350) Chip ID
     (256Mb) DRAM Component

## 2、make，根据报错补救


    ./tools/mkimage: invalid entry point -n
        检查下最后执行的指令，./tools/mkimage...
            定位到顶层makefile，readelf -h u-boot | grep "Entry" | awk '{print $$4}'
        执行，readelf -h u-boot，可以发现，因为语言原因，“Entry“变成了“入口点地址”
            中文的输出对应行：
                入口点地址：               0x80200000
            如果将系统语言改为英文，输出如下
                Entry point address:               0x80200000
        把grep后面的“Entry“改成“入口点地址”，根据规律，还要改$$4为$$2
        即更改IN_SPI分支的readelf -h u-boot | grep "Entry" | awk '{print $$4}'为
            readelf -h u-boot | grep "入口点地址" | awk '{print $$2}'

## 3、将得到的文件：uboot.img烧录到SPI Flash即可



# mod说明



## 补丁1：





    实测发现在SecureCRT波特率为57600时执行loadb指令无法下载超过8K的数据，因此更改kermit参数，取消扩展长度数据包特性。
    cmd_load.c：




        //a_b[++length] = tochar (2);    /* only long packets */
        a_b[++length] = tochar (0);    //delete all features by hangbingli




## 补丁2：添加spi flash操纵代码


    spi_flash.c：
        取消注释：#define SPI_FLASH_DBG_CMD 
        添加：spi fread、spi ferase及spi fwrite相关代码




## 补丁3：启用非必要的命令


    /config.mk
        UN_NECESSITY_U_BOOT_CMD_OPEN = ON
        该启用项需要执行 make menuconfig




## 补丁4：考虑取消扩展长度下载速度极慢，添加loadb_fst指令，继承原loadb


    cmd_load.c：
    修改loadb_fst指令中kermit协议扩展数据包最大长度为4096-2=4094
    实测此时比较稳定（CH340转串口）




## 补丁5：启动等待时间调整为5s


    rt2880.h:
        修改 #define CONFIG_BOOTDELAY    30    /* autoboot after 30 seconds    */ 
        为 #define CONFIG_BOOTDELAY    5    /* autoboot after 30 seconds    */



# 4、调试说明



U-BOOT调试：




    波特率57600 ，打开串口，开机按4
    加载img镜像：loadb 或 loadb_fst
    擦除：spi ferase 0 40000            #根据镜像大小设置
    写入：spi fwrite 0 80100000 xxxxx        #xxxxx是loadb下载的大小
    重启：reset

无线参数烧录（镜像文件从原始固件取出，此处参考：https://openwrt.org/toh/7links/px4885）




    spi ferase 40000 10000
    loadb_fst
    spi fwrite 40000 0x80100000 10000

openwrt烧录


    spi ferase 50000 800000
    loadb_fst
    spi fwrite 50000 0x80100000 xxxxxxxx
