# 参考文档

[泰山派rk3566 spi驱动st7789 tft lcd屏幕\_rk3566 spi屏幕-CSDN博客](https://blog.csdn.net/weixin_73794909/article/details/139270013)

[六、触摸驱动 - 飞书云文档](https://lceda001.feishu.cn/wiki/AhDkwcP5RiwUHXkzMVOc1JMPnBg)

[Orangepi3B使用st7789v SPI屏（一） - 哔哩哔哩](https://www.bilibili.com/opus/949536446456266753)

[Orangepi3B使用st7789v SPI屏（二） - 哔哩哔哩](https://www.bilibili.com/opus/950262291667877910)

# 将驱动编译成模块

进入kernel目录

    make menuconfig

按照顺序依次选择Device Driver->Staging drivers->Support for small TFT LCD display modules

![](https://i-blog.csdnimg.cn/blog_migrate/823f6561d996eabf722635806f0e9595.png)

保存到.config，执行下面命令

    make clean
    sudo make modules ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

# 也可以选择驱动编译进内核

# 驱动目录

生成后的ko文件放在/kernel/drivers/staging/fbtft

# 显示图片到spi屏幕

    sudo fbi -T 1 -a -d /dev/fb0 -noverbose logo.jpg

# 教程

    Linux驱动标准SPI TFT屏幕ST7789V电容触摸屏FT6336U
    1. 实验物料
    一个驱动芯片使用ST7789V的TFT显示屏+FT6336U为主控的电容触摸

    2. Framebuffer
    在 Linux 系统中通过 Framebuffer 驱动程序来控制 LCD。Frame 是帧的意思，buffer 是缓冲的意思，这意味着 Framebuffer 就是一块内存，里面保存着一帧图像。Framebuffer 中保存着一帧图像的每一个像素颜色值，假设 LCD 的 分辨率是 1024x768，每一个像素的颜色用 32 位来表示，那么 Framebuffer 的 大小就是：1024x768x32/8=3145728 字节。

    帧缓冲(framebuffer)是Linux系统中的一种显示驱动接口，它将显示设备(如LCD)进行抽象、屏蔽了不同显示设备硬件的实现，对应用层抽象为一块显示内存(显存)，它允许上层应用程序直接对显示缓冲区进行读写操作，而用户不必关心物理显存的位置等具体细节，这些都由Framebuffer设备驱动来完成。

    FrameBuffer设备对应的设备文件为/dev/fbX(X为数字，0、1、2、3等)，Linux下可支持多个FrameBuffer设备，最多可达32个，分别为/dev/fb0到/dev/fb31，开发板出厂系统中，/dev/fb0设备节点便是LCD屏。

    2.1 简要分析fbtft框架
    例子为ST7789V

    对应的驱动C文件路径是内核下路径为： drivers/staging/fbtft/fb\_st7789v.c

    以下是复制源码片段

    static struct fbtft_display display = {
     .regwidth = 8,
     .width = 128,
     .height = 160,
     .init_sequence = default_init_sequence,
     .gamma_num = 2,
     .gamma_len = 16,
     .gamma = DEFAULT_GAMMA,
     .fbtftops = {
      //.init_display = init_display,
      .set_addr_win = set_addr_win,
      .set_var = set_var,
      .set_gamma = set_gamma,
     },
    };
    struct fbtft_display 用于描述一款 TFT-LCD，包括硬件参数+硬件访问操作函数，需要根据 LCD 驱动 IC 的手册进行填写：

    regwidth: LCD 驱动 IC 寄存器的位宽;
    width / height: LCD 的分辨率;
    init_sequence： 初始化序列;
    fbtftops: LCD 操作函数集，init_display 和 init_sequence 一般只需要设置其中一项就行了;
    代码中

    FBTFT\_REGISTER\_DRIVER(DRVNAME, "sitronix,st7789v", &display);

    这句话是宏定义中实现了一个驱动程序，其中根据设备树匹配规则，可以匹配SPI设备和平台总线设备。

    设备树里的带有 "sitronix,st7735r" 属性的节点匹配上，触发 fbtft-core.c / fbtft\_probe\_common()。

    static int __init fbtft_driver_module_init(void)                           \
    {                                                                          \
        int ret;                                                           \
                                           \
        ret = spi_register_driver(&fbtft_driver_spi_driver);               \
        if (ret < 0)                                                       \
            return ret;                                                \
        ret = platform_driver_register(&fbtft_driver_platform_driver);     \
        if (ret < 0)                                                       \
            spi_unregister_driver(&fbtft_driver_spi_driver);           \
        return ret;                                                        \
    }  
    根据分析probe函数可以得出其中调用了 drivers/staging/fbtft/fbtft-core.c 文件

    核心代码为：

    /**
     * fbtft_probe_common() - Generic device probe() helper function
     * @display: Display properties
     * @sdev: SPI device
     * @pdev: Platform device
     *
     * Allocates, initializes and registers a framebuffer
     *
     * Either @sdev or @pdev should be NULL
     *
     * Return: 0 if successful, negative if error
     */
    int fbtft_probe_common(struct fbtft_display *display,
                   struct spi_device *sdev,
                   struct platform_device *pdev)
    其中参数 struct spi_device *sdev 也就是说支持SPI外设驱动。

    2.2 fbtft_probe_common 函数分析
    关键代码 ：

    1241行: pdata = fbtft\_properties\_read(dev);

    在这句代码中将会从device设备信息中这里应该是从设备树中读取配置信息，进入函数内可查看支持的配置项

    也就是说我们可以在设备树中写如下的配置，将会使用fbtft_property_value进行读取并填充到 struct fbtft_platform_data 结构体中保存

    static struct fbtft_platform_data *fbtft_properties_read(struct device *dev)
    {
        struct fbtft_platform_data *pdata;

        if (!dev_fwnode(dev)) {
            dev_err(dev, "Missing platform data or properties\n");
            return ERR_PTR(-EINVAL);
        }

        pdata = devm_kzalloc(dev, sizeof(*pdata), GFP_KERNEL);
        if (!pdata)
            return ERR_PTR(-ENOMEM);

        pdata->display.width = fbtft_property_value(dev, "width");
        pdata->display.height = fbtft_property_value(dev, "height");
        pdata->display.regwidth = fbtft_property_value(dev, "regwidth");
        pdata->display.buswidth = fbtft_property_value(dev, "buswidth");
        pdata->display.backlight = fbtft_property_value(dev, "backlight");
        pdata->display.bpp = fbtft_property_value(dev, "bpp");
        pdata->display.debug = fbtft_property_value(dev, "debug");
        pdata->rotate = fbtft_property_value(dev, "rotate");
        pdata->bgr = device_property_read_bool(dev, "bgr");
        pdata->fps = fbtft_property_value(dev, "fps");
        pdata->txbuflen = fbtft_property_value(dev, "txbuflen");
        pdata->startbyte = fbtft_property_value(dev, "startbyte");
        device_property_read_string(dev, "gamma", (const char **)&pdata->gamma);

        if (device_property_present(dev, "led-gpios"))
            pdata->display.backlight = 1;
        if (device_property_present(dev, "init"))
            pdata->display.fbtftops.init_display =
                fbtft_init_display_from_property;

        pdata->display.fbtftops.request_gpios = fbtft_request_gpios;

        return pdata;
    }
    关于字段的描述信息有如下说明

     * struct fbtft_display - Describes the display properties
     * @width: Width of display in pixels
     * @height: Height of display in pixels
     * @regwidth: LCD Controller Register width in bits
     * @buswidth: Display interface bus width in bits
     * @backlight: Backlight type.
     * @fbtftops: FBTFT operations provided by driver or device (platform_data)
     * @bpp: Bits per pixel
     * @fps: Frames per second
     * @txbuflen: Size of transmit buffer
     * @init_sequence: Pointer to LCD initialization array
     * @gamma: String representation of Gamma curve(s)
     * @gamma_num: Number of Gamma curves
     * @gamma_len: Number of values per Gamma curve
     * @debug: Initial debug value
     
     
     *struct fbtft_display- 描述显示属性
    *@width：以像素为单位的显示宽度
    *@height：以像素为单位的显示高度
    *@regwidth:LCD控制器寄存器宽度（以位为单位）
    *@buswidth：以位为单位显示接口总线宽度
    *@backlight：背光类型。
    *@fbtftops:FBTFT操作由驱动程序或设备提供（platform_data）
    *@bpp：每像素位
    *@fps：每秒帧数
    *@txbuflen：传输缓冲区大小
    *@init_sequence：指向LCD初始化数组的指针
    *@gamma：gamma曲线的字符串表示
    *@gamma_num：伽玛曲线数
    *@gamma_len：每条gamma曲线的值数
    *@debug：初始调试值


     * struct fbtft_platform_data - Passes display specific data to the driver
     * @display: Display properties
     * @gpios: Pointer to an array of pinname to gpio mappings
     * @rotate: Display rotation angle
     * @bgr: LCD Controller BGR bit
     * @fps: Frames per second (this will go away, use @fps in @fbtft_display)
     * @txbuflen: Size of transmit buffer
     * @startbyte: When set, enables use of Startbyte in transfers
     * @gamma: String representation of Gamma curve(s)
     * @extra: A way to pass extra info
     
     
     *struct fbtft_platform_data-将显示特定的数据传递给驱动程序
    *@display：显示属性
    *@gpios:pinname到gpio映射数组的指针
    *@rotate：显示旋转角度
    *@bgr:LCD控制器bgr位
    *@fps：每秒帧数（这将消失，在@fbtft_display中使用@fps）
    *@txbuflen：传输缓冲区大小
    *@startbyte：设置后，启用在传输中使用startbyte
    *@gamma：gamma曲线的字符串表示
    *@extra：一种传递额外信息的方式
    我们继续往下分析代码

    在执行到这里的部分将会初始化 framebuffer

    info = fbtft_framebuffer_alloc(display, dev, pdata);
        if (!info)
            return -ENOMEM;
    这里会优先选择display结构体内的参数为默认参数，如果设备数有配置将会覆盖display结构体原先相同的配置

    3. 编写设备树
    需要注意的是，fbtft的驱动只有在内核framebuffer相关驱动开启后才会出现，所以要先开启fb驱动支持
    Device Drivers -> Graphics support -> Frame buffer Devices --->

    然后寻找fbtft驱动，路径如下：
    Device Drivers -> Staging drivers -> Support for small TFT LCD display modules --->，编译进内核



    3.1 编写TFT设备树
    修改内容

    首先确定屏幕的引脚使用的有哪些，排查是否有其他功能被占用，将被占用的功能注释掉。
    添加SPI屏幕需要的引脚并设置好复用例如： rest、dc、bl
    在 &pinctrl 节点内添加如下内容设置GPIO的复用关系

    设置SPI复用

    	spi3_tft{
    		spi3_tft:spi3-tft{
    			rockchip,pins =
    			/* spi3_clkm1 */
    			<4 RK_PC2 2 &pcfg_pull_up_drv_level_1>,
    			/* spi3_cs */
    			<4 RK_PC5 RK_FUNC_GPIO &pcfg_pull_none>,
    			/* spi3_mosim1 */
    			<4 RK_PC3 2 &pcfg_pull_up_drv_level_1>;
    		};
    	};
    配置TFT

    /* 配置TFT */
    	tft_rst {
    		gpio3_pb6:gpio3-pb6 {
    			rockchip,pins = <3 RK_PB6 RK_FUNC_GPIO &pcfg_output_high>;
    		};
    	};

    	tft_dc {
    		gpio3_pb0:gpio3-pb0 {
    			rockchip,pins = <3 RK_PB0 RK_FUNC_GPIO &pcfg_pull_none>;
    		};
    	};

    	tft_bl {
    		gpio3_pc2:gpio3-pc2 {
    			rockchip,pins = <3 RK_PC2 RK_FUNC_GPIO &pcfg_output_low>;
    		};
    	};
    因为我使用的是SPI3的复用管脚所以开启SPI3

    &spi3{
    	status = "okay";
    	pinctrl-names = "default";
    	pinctrl-0 = <&spi3_tft &gpio3_pb6 &gpio3_pb0 &gpio3_pc2>;
    	#address-cells = <1>;
    	#size-cells = <0>;
    	tft: lcd@0{
    		compatible = "sitronix,st7789v"; //固定的写法使用sitronix,st7789v驱动
    		spi-max-frequency = <100000000>; //设置SPI的速率
    		reg = <0>;//SPI序号
    		spi-cpol;
    		spi-cpha;
    		rotate = <90>;// 旋转角度，lcd驱动里会读取并设置对应寄存器
    		fps = <100>;
    		width = <240>;
    		height = <320>;
    		invert;
    		// bgr = <0>;
    		buswidth = <8>;//SPI 8位数据长度
    		led-gpios = <&gpio3 RK_PC2 GPIO_ACTIVE_LOW>;//BL
    		dc-gpios = <&gpio3 RK_PB0 GPIO_ACTIVE_LOW>;//DC
    		reset-gpios = <&gpio3 RK_PB6 GPIO_ACTIVE_HIGH>;//RES
    		cs-gpios = <&gpio4 RK_PC5 GPIO_ACTIVE_LOW>;//CS
    		// debug = <0x7>;
    	};

    };
    3.1.1 测试刷屏
    //搜索内容日志的打印
    # dmesg | grep fb_


    // 测试花屏
    # cat /dev/urandom > /dev/fb0
    // 测试清屏
    # cat /dev/zero > /dev/fb0
    使用屏幕颜色与动画测试工具

    https://github.com/ccy-studio/fb-test-app

    此工具需要自行编译

    3.2 触摸驱动 FT6636U适配
    首先要在内核中开启配置 CONFIG_TOUCHSCREEN_FTS 可以内核中在menuconfig搜索然后打上星

    Device Drivers -> Input Device Support ->Touchscreens -> Focaltech Touchscreen

    在 &pinctrl 节点内添加如下内容设置GPIO的复用关系、

    /* 配置触摸 */
    	touch_int {
    		gpio3_pa1:gpio3-pa1 {
    			rockchip,pins = <3 RK_PA1 RK_FUNC_GPIO &pcfg_pull_up>;
    		};
    	};

    	touch_rst {
    		gpio1_pa4:gpio1-pa4 {
    			rockchip,pins = <1 RK_PA4 RK_FUNC_GPIO &pcfg_output_high>;
    		};
    	};

    由于我使用的是I2C2的复用管脚所以使用i2c2

    &i2c2{
    	status = "okay";
    	focaltech@38{
    		status = "okay";
    		compatible = "focaltech,fts";
    		pinctrl-names = "default";
    		pinctrl-0 = <&gpio3_pa1 &gpio1_pa4>;
    		reg = <0x38>;//I2C地址
    		interrupt-parent = <&gpio3>;
    		interrupts = <RK_PA1 IRQ_TYPE_LEVEL_LOW>;
    		focaltech,reset-gpio = <&gpio1 RK_PA4 GPIO_ACTIVE_LOW>;
    		focaltech,irq-gpio = <&gpio3 RK_PA1 GPIO_ACTIVE_HIGH>;

    		focaltech,max-touch-number = <1>;//最大触摸数量-多点触摸配置
    		focaltech,display-coords =  <0 0 240 320>;//分辨率
    	};
    };
    3.2.1 测试触摸
    查看内核打印日志是否有报错

    dmesg | grep i2c

    如果有错误会打印error信息

    查看I2C挂载是否成功

    输入命令： ls /sys/bus/i2c/devices

    # ls /sys/bus/i2c/devices
    0-0038    4-0030    4-0030-1  4-0030-2  i2c-0     i2c-4

    可以看出来有x-0038这个地址的节点。 x代表使用到的 I2C 编号

    查看是否生成了输入子系统节点

    使用命令： cd /dev/input/


    # ls
    by-path  event0   event1

    这里我们依次测试event,过滤到那个是属于我们的触摸屏节点

    使用命令 hexdump event0 先测试0节点的


    # hexdump event0
    0000000 04ed 0000 a2d7 000a 0003 0039 0033 0000
    0000010 04ed 0000 a2d7 000a 0003 0035 0066 0000
    0000020 04ed 0000 a2d7 000a 0003 0036 0082 0000
    0000030 04ed 0000 a2d7 000a 0001 014a 0001 0000
    0000040 04ed 0000 a2d7 000a 0000 0000 0000 0000
    0000050 04ed 0000 cae6 000a 0003 0039 ffff ffff
    0000060 04ed 0000 cae6 000a 0001 014a 0000 0000
    0000070 04ed 0000 cae6 000a 0000 0000 0000 0000
    0000080 04ef 0000 c3d7 0004 0003 0039 0034 0000
    0000090 04ef 0000 c3d7 0004 0003 0035 004d 0000
    00000a0 04ef 0000 c3d7 0004 0003 0036 006c 0000
    00000b0 04ef 0000 c3d7 0004 0001 014a 0001 0000
    00000c0 04ef 0000 c3d7 0004 0000 0000 0000 0000
    00000d0 04ef 0000 ed72 0005 0003 0039 ffff ffff
    00000e0 04ef 0000 ed72 0005 0001 014a 0000 0000
    00000f0 04ef 0000 ed72 0005 0000 0000 0000 0000
    0000100 04f0 0000 6d20 0008 0003 0039 0035 0000
    0000110 04f0 0000 6d20 0008 0003 0035 003f 0000
    0000120 04f0 0000 6d20 0008 0003 0036 00a4 0000
    0000130 04f0 0000 6d20 0008 0001 014a 0001 0000
    0000140 04f0 0000 6d20 0008 0000 0000 0000 0000
    0000150 04f0 0000 dad8 0008 0003 0039 ffff ffff
    0000160 04f0 0000 dad8 0008 0001 014a 0000 0000
    0000170 04f0 0000 dad8 0008 0000 0000 0000 0000

    点了一下屏幕发现有日志打印说明event0就是我们的触摸屏节点，后面的就不测了因为我们已经找到了。

# 修改kernel/arch/arm64/configs/rockchip\*\_*linux*\_\*defconfig文件

    CONFIG_DEFAULT_HOSTNAME="localhost"
    CONFIG_SYSVIPC=y
    CONFIG_NO_HZ=y
    CONFIG_HIGH_RES_TIMERS=y
    CONFIG_PREEMPT_VOLUNTARY=y
    CONFIG_IKCONFIG=y
    CONFIG_IKCONFIG_PROC=y
    CONFIG_LOG_BUF_SHIFT=18
    CONFIG_CGROUPS=y
    CONFIG_CGROUP_SCHED=y
    CONFIG_CFS_BANDWIDTH=y
    CONFIG_CGROUP_FREEZER=y
    CONFIG_CPUSETS=y
    CONFIG_CGROUP_DEVICE=y
    CONFIG_CGROUP_CPUACCT=y
    CONFIG_NAMESPACES=y
    CONFIG_USER_NS=y
    CONFIG_BLK_DEV_INITRD=y
    # CONFIG_ROCKCHIP_ONE_INITRD is not set
    CONFIG_CC_OPTIMIZE_FOR_SIZE=y
    CONFIG_EMBEDDED=y
    # CONFIG_COMPAT_BRK is not set
    CONFIG_PROFILING=y
    CONFIG_ARCH_ROCKCHIP=y
    CONFIG_PCI=y
    CONFIG_PCIEPORTBUS=y
    CONFIG_PCIE_DW_ROCKCHIP=y
    # CONFIG_ARM64_ERRATUM_826319 is not set
    # CONFIG_ARM64_ERRATUM_827319 is not set
    # CONFIG_ARM64_ERRATUM_824069 is not set
    # CONFIG_ARM64_ERRATUM_819472 is not set
    # CONFIG_ARM64_ERRATUM_832075 is not set
    # CONFIG_CAVIUM_ERRATUM_22375 is not set
    # CONFIG_CAVIUM_ERRATUM_23154 is not set
    CONFIG_SCHED_MC=y
    CONFIG_NR_CPUS=8
    CONFIG_HZ_300=y
    CONFIG_SECCOMP=y
    CONFIG_ARMV8_DEPRECATED=y
    CONFIG_SWP_EMULATION=y
    CONFIG_CP15_BARRIER_EMULATION=y
    CONFIG_SETEND_EMULATION=y
    # CONFIG_EFI is not set
    CONFIG_COMPAT=y
    CONFIG_PM_DEBUG=y
    CONFIG_PM_ADVANCED_DEBUG=y
    CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y
    CONFIG_ENERGY_MODEL=y
    CONFIG_CPU_IDLE=y
    CONFIG_ARM_CPUIDLE=y
    CONFIG_CPU_FREQ=y
    CONFIG_CPU_FREQ_DEFAULT_GOV_INTERACTIVE=y
    CONFIG_CPU_FREQ_GOV_POWERSAVE=y
    CONFIG_CPU_FREQ_GOV_USERSPACE=y
    CONFIG_CPU_FREQ_GOV_ONDEMAND=y
    CONFIG_CPU_FREQ_GOV_CONSERVATIVE=y
    CONFIG_CPUFREQ_DT=y
    CONFIG_ARM_ROCKCHIP_CPUFREQ=y
    CONFIG_ARM_SCMI_PROTOCOL=y
    CONFIG_ROCKCHIP_SIP=y
    CONFIG_ARM64_CRYPTO=y
    CONFIG_CRYPTO_SHA1_ARM64_CE=y
    CONFIG_CRYPTO_SHA2_ARM64_CE=y
    CONFIG_CRYPTO_GHASH_ARM64_CE=y
    CONFIG_CRYPTO_AES_ARM64_CE_CCM=y
    CONFIG_CRYPTO_AES_ARM64_CE_BLK=y
    CONFIG_MODULES=y
    CONFIG_MODULE_FORCE_LOAD=y
    CONFIG_MODULE_UNLOAD=y
    CONFIG_MODULE_FORCE_UNLOAD=y
    CONFIG_PARTITION_ADVANCED=y
    # CONFIG_COMPACTION is not set
    CONFIG_DEFAULT_MMAP_MIN_ADDR=32768
    CONFIG_CMA=y
    CONFIG_ZSMALLOC=y
    CONFIG_NET=y
    CONFIG_PACKET=y
    CONFIG_UNIX=y
    CONFIG_XFRM_USER=y
    CONFIG_NET_KEY=y
    CONFIG_INET=y
    CONFIG_IP_MULTICAST=y
    CONFIG_IP_ADVANCED_ROUTER=y
    CONFIG_IP_MROUTE=y
    CONFIG_SYN_COOKIES=y
    # CONFIG_INET_XFRM_MODE_TRANSPORT is not set
    # CONFIG_INET_XFRM_MODE_TUNNEL is not set
    # CONFIG_INET_XFRM_MODE_BEET is not set
    # CONFIG_INET_DIAG is not set
    # CONFIG_INET6_XFRM_MODE_TRANSPORT is not set
    # CONFIG_INET6_XFRM_MODE_TUNNEL is not set
    # CONFIG_INET6_XFRM_MODE_BEET is not set
    # CONFIG_IPV6_SIT is not set
    CONFIG_NETFILTER=y
    CONFIG_IP_NF_IPTABLES=y
    CONFIG_IP_NF_MANGLE=y
    CONFIG_BT=y
    CONFIG_BT_RFCOMM=y
    CONFIG_BT_HIDP=y
    CONFIG_BT_HCIBTUSB=y
    CONFIG_BT_HCIUART=y
    CONFIG_BT_HCIUART_ATH3K=y
    CONFIG_BT_HCIBFUSB=y
    CONFIG_BT_HCIVHCI=y
    CONFIG_BT_MRVL=y
    CONFIG_BT_MRVL_SDIO=y
    CONFIG_NL80211_TESTMODE=y
    CONFIG_CFG80211_DEBUGFS=y
    CONFIG_CFG80211_WEXT=y
    CONFIG_MAC80211_LEDS=y
    CONFIG_MAC80211_DEBUGFS=y
    CONFIG_MAC80211_DEBUG_MENU=y
    CONFIG_MAC80211_VERBOSE_DEBUG=y
    CONFIG_RFKILL=y
    CONFIG_DEVTMPFS=y
    CONFIG_DEVTMPFS_MOUNT=y
    CONFIG_DEBUG_DEVRES=y
    CONFIG_DMA_CMA=y
    CONFIG_CONNECTOR=y
    CONFIG_MTD=y
    CONFIG_MTD_CMDLINE_PARTS=y
    CONFIG_MTD_BLOCK=y
    CONFIG_MTD_UBI=y
    CONFIG_ZRAM=y
    CONFIG_BLK_DEV_LOOP=y
    CONFIG_BLK_DEV_RAM=y
    CONFIG_BLK_DEV_RAM_COUNT=1
    CONFIG_BLK_DEV_NVME=y
    CONFIG_SRAM=y
    CONFIG_BLK_DEV_SD=y
    CONFIG_BLK_DEV_SR=y
    CONFIG_SCSI_SCAN_ASYNC=y
    CONFIG_SCSI_SPI_ATTRS=y
    CONFIG_ATA=y
    CONFIG_SATA_AHCI=y
    CONFIG_SATA_AHCI_PLATFORM=y
    # CONFIG_ATA_SFF is not set
    CONFIG_MD=y
    CONFIG_NETDEVICES=y
    # CONFIG_NET_VENDOR_3COM is not set
    # CONFIG_NET_VENDOR_ADAPTEC is not set
    # CONFIG_NET_VENDOR_AGERE is not set
    # CONFIG_NET_VENDOR_ALTEON is not set
    # CONFIG_NET_VENDOR_AMD is not set
    # CONFIG_NET_VENDOR_ARC is not set
    # CONFIG_NET_VENDOR_ATHEROS is not set
    # CONFIG_NET_VENDOR_BROADCOM is not set
    # CONFIG_NET_VENDOR_BROCADE is not set
    # CONFIG_NET_VENDOR_CAVIUM is not set
    # CONFIG_NET_VENDOR_CHELSIO is not set
    # CONFIG_NET_VENDOR_CISCO is not set
    # CONFIG_NET_VENDOR_DEC is not set
    # CONFIG_NET_VENDOR_DLINK is not set
    # CONFIG_NET_VENDOR_EMULEX is not set
    # CONFIG_NET_VENDOR_EZCHIP is not set
    # CONFIG_NET_VENDOR_HISILICON is not set
    # CONFIG_NET_VENDOR_HP is not set
    # CONFIG_NET_VENDOR_INTEL is not set
    # CONFIG_NET_VENDOR_MARVELL is not set
    # CONFIG_NET_VENDOR_MELLANOX is not set
    # CONFIG_NET_VENDOR_MICREL is not set
    # CONFIG_NET_VENDOR_MICROCHIP is not set
    # CONFIG_NET_VENDOR_MYRI is not set
    # CONFIG_NET_VENDOR_NATSEMI is not set
    # CONFIG_NET_VENDOR_NVIDIA is not set
    # CONFIG_NET_VENDOR_OKI is not set
    # CONFIG_NET_VENDOR_QLOGIC is not set
    # CONFIG_NET_VENDOR_QUALCOMM is not set
    # CONFIG_NET_VENDOR_RDC is not set
    # CONFIG_NET_VENDOR_REALTEK is not set
    # CONFIG_NET_VENDOR_RENESAS is not set
    # CONFIG_NET_VENDOR_ROCKER is not set
    # CONFIG_NET_VENDOR_SAMSUNG is not set
    # CONFIG_NET_VENDOR_SEEQ is not set
    # CONFIG_NET_VENDOR_SILAN is not set
    # CONFIG_NET_VENDOR_SIS is not set
    # CONFIG_NET_VENDOR_SMSC is not set
    CONFIG_STMMAC_ETH=y
    # CONFIG_NET_VENDOR_SUN is not set
    # CONFIG_NET_VENDOR_SYNOPSYS is not set
    # CONFIG_NET_VENDOR_TEHUTI is not set
    # CONFIG_NET_VENDOR_TI is not set
    # CONFIG_NET_VENDOR_VIA is not set
    # CONFIG_NET_VENDOR_WIZNET is not set
    CONFIG_ROCKCHIP_PHY=y
    CONFIG_RK630_PHY=y
    CONFIG_USB_RTL8150=y
    CONFIG_USB_RTL8152=y
    CONFIG_USB_NET_CDC_MBIM=y
    # CONFIG_USB_NET_NET1080 is not set
    # CONFIG_USB_NET_CDC_SUBSET is not set
    # CONFIG_USB_NET_ZAURUS is not set
    CONFIG_LIBERTAS_THINFIRM=y
    CONFIG_MWIFIEX=m
    CONFIG_MWIFIEX_SDIO=m
    CONFIG_WL_ROCKCHIP=y
    CONFIG_WIFI_BUILD_MODULE=y
    # CONFIG_WIFI_LOAD_DRIVER_WHEN_KERNEL_BOOTUP  is not set
    CONFIG_AP6XXX=m
    CONFIG_USB_NET_RNDIS_WLAN=y
    CONFIG_INPUT_FF_MEMLESS=y
    CONFIG_INPUT_EVDEV=y
    CONFIG_KEYBOARD_ADC=y
    # CONFIG_KEYBOARD_ATKBD is not set
    CONFIG_KEYBOARD_GPIO=y
    CONFIG_KEYBOARD_GPIO_POLLED=y
    CONFIG_KEYBOARD_CROS_EC=y
    # CONFIG_MOUSE_PS2 is not set
    CONFIG_MOUSE_CYAPA=y
    CONFIG_MOUSE_ELAN_I2C=y
    CONFIG_INPUT_TOUCHSCREEN=y
    CONFIG_TOUCHSCREEN_ATMEL_MXT=y
    CONFIG_TOUCHSCREEN_GSLX680_VR=y
    CONFIG_TOUCHSCREEN_GSL3673=y
    CONFIG_TOUCHSCREEN_GT9XX=y
    CONFIG_TOUCHSCREEN_ELAN=y
    CONFIG_TOUCHSCREEN_WACOM_W9013=y
    CONFIG_TOUCHSCREEN_USB_COMPOSITE=y
    CONFIG_TOUCHSCREEN_GT1X=y
    CONFIG_TOUCHSCREEN_CYPRESS_CYTTSP5=y
    CONFIG_TOUCHSCREEN_CYPRESS_CYTTSP5_DEVICETREE_SUPPORT=y
    CONFIG_TOUCHSCREEN_CYPRESS_CYTTSP5_I2C=y
    CONFIG_TOUCHSCREEN_CYPRESS_CYTTSP5_DEVICE_ACCESS=y
    CONFIG_TOUCHSCREEN_CYPRESS_CYTTSP5_LOADER=y
    CONFIG_ROCKCHIP_REMOTECTL=y
    CONFIG_ROCKCHIP_REMOTECTL_PWM=y
    CONFIG_INPUT_MISC=y
    CONFIG_INPUT_UINPUT=y
    CONFIG_INPUT_RK805_PWRKEY=y
    # CONFIG_SERIO is not set
    CONFIG_VT_HW_CONSOLE_BINDING=y
    # CONFIG_LEGACY_PTYS is not set
    CONFIG_SERIAL_8250=y
    CONFIG_SERIAL_8250_CONSOLE=y
    # CONFIG_SERIAL_8250_PCI is not set
    CONFIG_SERIAL_8250_NR_UARTS=10
    CONFIG_SERIAL_8250_RUNTIME_UARTS=10
    CONFIG_SERIAL_8250_DW=y
    CONFIG_SERIAL_OF_PLATFORM=y
    CONFIG_HW_RANDOM=y
    CONFIG_HW_RANDOM_ROCKCHIP=y
    CONFIG_TCG_TPM=y
    CONFIG_TCG_TIS_I2C_INFINEON=y
    CONFIG_I2C_CHARDEV=y
    CONFIG_I2C_RK3X=y
    CONFIG_I2C_CROS_EC_TUNNEL=y
    CONFIG_SPI=y
    CONFIG_SPI_BITBANG=y
    CONFIG_SPI_ROCKCHIP=y
    CONFIG_SPI_SPIDEV=y
    CONFIG_PINCTRL_RK805=y
    CONFIG_GPIO_SYSFS=y
    CONFIG_GPIO_GENERIC_PLATFORM=y
    CONFIG_POWER_AVS=y
    CONFIG_ROCKCHIP_IODOMAIN=y
    CONFIG_POWER_RESET_GPIO=y
    CONFIG_POWER_RESET_GPIO_RESTART=y
    CONFIG_SYSCON_REBOOT_MODE=y
    CONFIG_BATTERY_SBS=y
    CONFIG_CHARGER_GPIO=y
    CONFIG_CHARGER_BQ24735=y
    CONFIG_BATTERY_RK817=y
    CONFIG_CHARGER_RK817=y
    CONFIG_THERMAL=y
    CONFIG_THERMAL_WRITABLE_TRIPS=y
    CONFIG_THERMAL_DEFAULT_GOV_POWER_ALLOCATOR=y
    CONFIG_THERMAL_GOV_FAIR_SHARE=y
    CONFIG_THERMAL_GOV_STEP_WISE=y
    CONFIG_CPU_THERMAL=y
    CONFIG_DEVFREQ_THERMAL=y
    CONFIG_ROCKCHIP_THERMAL=y
    CONFIG_WATCHDOG=y
    CONFIG_DW_WATCHDOG=y
    CONFIG_MFD_CROS_EC=y
    CONFIG_MFD_RK618=y
    CONFIG_MFD_RK628=y
    CONFIG_MFD_RK630_I2C=y
    CONFIG_MFD_RK808=y
    CONFIG_MFD_TPS6586X=y
    CONFIG_FUSB_30X=y
    CONFIG_REGULATOR=y
    CONFIG_REGULATOR_DEBUG=y
    CONFIG_REGULATOR_FIXED_VOLTAGE=y
    CONFIG_REGULATOR_ACT8865=y
    CONFIG_REGULATOR_FAN53555=y
    CONFIG_REGULATOR_GPIO=y
    CONFIG_REGULATOR_LP8752=y
    CONFIG_REGULATOR_MP8865=y
    CONFIG_REGULATOR_PWM=y
    CONFIG_REGULATOR_RK808=y
    CONFIG_REGULATOR_TPS65132=y
    CONFIG_REGULATOR_TPS6586X=y
    CONFIG_REGULATOR_XZ3216=y
    CONFIG_MEDIA_SUPPORT=y
    CONFIG_MEDIA_CAMERA_SUPPORT=y
    CONFIG_MEDIA_CEC_SUPPORT=y
    CONFIG_MEDIA_CONTROLLER=y
    CONFIG_VIDEO_V4L2_SUBDEV_API=y
    CONFIG_MEDIA_USB_SUPPORT=y
    CONFIG_USB_VIDEO_CLASS=y
    # CONFIG_USB_VIDEO_CLASS_INPUT_EVDEV is not set
    # CONFIG_USB_GSPCA is not set
    CONFIG_V4L_PLATFORM_DRIVERS=y
    CONFIG_SOC_CAMERA=y
    CONFIG_VIDEO_ROCKCHIP_ISP1=y
    CONFIG_VIDEO_ROCKCHIP_ISP=y
    CONFIG_V4L_MEM2MEM_DRIVERS=y
    CONFIG_VIDEO_ROCKCHIP_RGA=y
    # CONFIG_MEDIA_SUBDRV_AUTOSELECT is not set
    CONFIG_VIDEO_TC35874X=y
    CONFIG_VIDEO_RK628_CSI=y
    CONFIG_VIDEO_LT6911UXC=y
    CONFIG_VIDEO_LT8619C=y
    CONFIG_VIDEO_OS04A10=y
    CONFIG_VIDEO_OV4689=y
    CONFIG_VIDEO_OV5695=y
    CONFIG_VIDEO_OV7251=y
    CONFIG_VIDEO_OV13850=y
    CONFIG_VIDEO_GC8034=y
    # CONFIG_VGA_ARB is not set
    CONFIG_DRM=y
    CONFIG_DRM_IGNORE_IOTCL_PERMIT=y
    CONFIG_DRM_LOAD_EDID_FIRMWARE=y
    CONFIG_DRM_ROCKCHIP=y
    CONFIG_ROCKCHIP_ANALOGIX_DP=y
    CONFIG_ROCKCHIP_CDN_DP=y
    CONFIG_ROCKCHIP_DW_HDMI=y
    CONFIG_ROCKCHIP_DW_MIPI_DSI=y
    CONFIG_ROCKCHIP_INNO_HDMI=y
    CONFIG_ROCKCHIP_LVDS=y
    CONFIG_ROCKCHIP_DRM_TVE=y
    CONFIG_ROCKCHIP_RGB=y
    CONFIG_DRM_ROCKCHIP_RK618=y
    CONFIG_DRM_PANEL_SIMPLE=y
    CONFIG_DRM_RK630_TVE=y
    CONFIG_DRM_SII902X=y
    CONFIG_DRM_DW_HDMI_I2S_AUDIO=y
    CONFIG_DRM_DW_HDMI_CEC=y
    CONFIG_MALI400=y
    CONFIG_MALI450=y
    # CONFIG_MALI400_PROFILING is not set
    CONFIG_MALI_SHARED_INTERRUPTS=y
    CONFIG_MALI_DT=y
    CONFIG_MALI_DEVFREQ=y
    CONFIG_MALI_MIDGARD=y
    CONFIG_MALI_EXPERT=y
    CONFIG_MALI_PLATFORM_THIRDPARTY=y
    CONFIG_MALI_PLATFORM_THIRDPARTY_NAME="rk"
    CONFIG_MALI_DEBUG=y
    CONFIG_MALI_PWRSOFT_765=y
    CONFIG_MALI_BIFROST=y
    CONFIG_MALI_BIFROST_DEVFREQ=y
    CONFIG_MALI_PLATFORM_NAME="rk"
    CONFIG_BACKLIGHT_LCD_SUPPORT=y
    # CONFIG_LCD_CLASS_DEVICE is not set
    CONFIG_BACKLIGHT_CLASS_DEVICE=y
    CONFIG_BACKLIGHT_PWM=y
    CONFIG_DUMMY_CONSOLE_COLUMNS=320
    CONFIG_DUMMY_CONSOLE_ROWS=240
    CONFIG_FRAMEBUFFER_CONSOLE=y
    CONFIG_ROCKCHIP_RGA2=y
    CONFIG_ROCKCHIP_MPP_SERVICE=y
    CONFIG_ROCKCHIP_MPP_RKVDEC=y
    CONFIG_ROCKCHIP_MPP_RKVDEC2=y
    CONFIG_ROCKCHIP_MPP_RKVENC=y
    CONFIG_ROCKCHIP_MPP_VDPU1=y
    CONFIG_ROCKCHIP_MPP_VEPU1=y
    CONFIG_ROCKCHIP_MPP_VDPU2=y
    CONFIG_ROCKCHIP_MPP_VEPU2=y
    CONFIG_ROCKCHIP_MPP_IEP2=y
    CONFIG_ROCKCHIP_MPP_JPGDEC=y
    CONFIG_SOUND=y
    CONFIG_SND=y
    CONFIG_SND_HRTIMER=y
    CONFIG_SND_DYNAMIC_MINORS=y
    # CONFIG_SND_SUPPORT_OLD_API is not set
    CONFIG_SND_SEQUENCER=y
    CONFIG_SND_SEQ_DUMMY=y
    # CONFIG_SND_PCI is not set
    # CONFIG_SND_SPI is not set
    CONFIG_SND_USB_AUDIO=y
    CONFIG_SND_SOC=y
    CONFIG_SND_SOC_ROCKCHIP=y
    CONFIG_SND_SOC_ROCKCHIP_I2S_TDM=y
    CONFIG_SND_SOC_ROCKCHIP_PDM=y
    CONFIG_SND_SOC_ROCKCHIP_SPDIF=y
    CONFIG_SND_SOC_ROCKCHIP_MAX98090=y
    CONFIG_SND_SOC_ROCKCHIP_MULTICODECS=y
    CONFIG_SND_SOC_ROCKCHIP_RT5645=y
    CONFIG_SND_SOC_ROCKCHIP_RT5651_RK628=y
    CONFIG_SND_SOC_ROCKCHIP_HDMI=y
    CONFIG_SND_SOC_DUMMY_CODEC=y
    CONFIG_SND_SOC_ES7202=y
    CONFIG_SND_SOC_ES7243E=y
    CONFIG_SND_SOC_ES8311=y
    CONFIG_SND_SOC_ES8316=y
    CONFIG_SND_SOC_RK3328=y
    CONFIG_SND_SOC_RK817=y
    CONFIG_SND_SOC_RK_CODEC_DIGITAL=y
    CONFIG_SND_SOC_RT5616=y
    CONFIG_SND_SOC_RT5640=y
    CONFIG_SND_SOC_SPDIF=y
    CONFIG_SND_SIMPLE_CARD=y
    CONFIG_HID_BATTERY_STRENGTH=y
    CONFIG_HIDRAW=y
    CONFIG_UHID=y
    CONFIG_HID_KENSINGTON=y
    CONFIG_HID_MULTITOUCH=y
    CONFIG_USB_HIDDEV=y
    CONFIG_I2C_HID=y
    CONFIG_USB_ANNOUNCE_NEW_DEVICES=y
    # CONFIG_USB_DEFAULT_PERSIST is not set
    CONFIG_USB_OTG=y
    CONFIG_USB_MON=y
    CONFIG_USB_XHCI_HCD=y
    CONFIG_USB_EHCI_HCD=y
    CONFIG_USB_EHCI_ROOT_HUB_TT=y
    CONFIG_USB_EHCI_HCD_PLATFORM=y
    CONFIG_USB_OHCI_HCD=y
    # CONFIG_USB_OHCI_HCD_PCI is not set
    CONFIG_USB_OHCI_HCD_PLATFORM=y
    CONFIG_USB_ACM=y
    CONFIG_USB_STORAGE=y
    CONFIG_USB_UAS=y
    CONFIG_USB_DWC3=y
    CONFIG_USB_DWC2=y
    CONFIG_USB_SERIAL=y
    CONFIG_USB_SERIAL_GENERIC=y
    CONFIG_USB_SERIAL_CH341=y
    CONFIG_USB_SERIAL_CP210X=y
    CONFIG_USB_SERIAL_FTDI_SIO=y
    CONFIG_USB_SERIAL_KEYSPAN=y
    CONFIG_USB_SERIAL_PL2303=y
    CONFIG_USB_SERIAL_OTI6858=y
    CONFIG_USB_SERIAL_QUALCOMM=y
    CONFIG_USB_SERIAL_SIERRAWIRELESS=y
    CONFIG_USB_SERIAL_OPTION=y
    CONFIG_USB_GADGET=y
    CONFIG_USB_GADGET_DEBUG_FILES=y
    CONFIG_USB_GADGET_VBUS_DRAW=500
    CONFIG_USB_CONFIGFS=y
    CONFIG_USB_CONFIGFS_UEVENT=y
    CONFIG_USB_CONFIGFS_ACM=y
    CONFIG_USB_CONFIGFS_MASS_STORAGE=y
    CONFIG_USB_CONFIGFS_F_FS=y
    CONFIG_USB_CONFIGFS_F_UVC=y
    CONFIG_MMC=y
    CONFIG_MMC_BLOCK_MINORS=32
    CONFIG_MMC_TEST=y
    CONFIG_SDIO_KEEPALIVE=y
    CONFIG_MMC_SDHCI=y
    CONFIG_MMC_SDHCI_PLTFM=y
    CONFIG_MMC_SDHCI_OF_ARASAN=y
    CONFIG_MMC_SDHCI_OF_DWCMSHC=y
    CONFIG_MMC_DW=y
    CONFIG_MMC_DW_ROCKCHIP=y
    CONFIG_NEW_LEDS=y
    CONFIG_LEDS_CLASS=y
    CONFIG_LEDS_GPIO=y
    CONFIG_LEDS_IS31FL32XX=y
    CONFIG_LEDS_TRIGGER_TIMER=y
    CONFIG_RTC_CLASS=y
    CONFIG_RTC_DRV_HYM8563=y
    CONFIG_RTC_DRV_RK808=y
    CONFIG_DMADEVICES=y
    CONFIG_PL330_DMA=y
    CONFIG_STAGING=y
    CONFIG_FIQ_DEBUGGER=y
    CONFIG_FIQ_DEBUGGER_NO_SLEEP=y
    CONFIG_FIQ_DEBUGGER_CONSOLE=y
    CONFIG_FIQ_DEBUGGER_CONSOLE_DEFAULT_ENABLE=y
    CONFIG_FIQ_DEBUGGER_TRUST_ZONE=y
    CONFIG_RK_CONSOLE_THREAD=y
    CONFIG_FB_TFT=y
    CONFIG_FB_TFT_ST7789V=y
    CONFIG_FB_TFT_FBTFT_DEVICE=y
    CONFIG_COMMON_CLK_RK808=y
    CONFIG_COMMON_CLK_SCMI=y
    CONFIG_MAILBOX=y
    CONFIG_ROCKCHIP_IOMMU=y
    CONFIG_CPU_PX30=y
    CONFIG_CPU_RK1808=y
    CONFIG_CPU_RK3328=y
    CONFIG_CPU_RK3399=y
    CONFIG_CPU_RK3568=y
    CONFIG_ROCKCHIP_PM_DOMAINS=y
    CONFIG_ROCKCHIP_PVTM=y
    CONFIG_ROCKCHIP_SUSPEND_MODE=y
    CONFIG_ROCKCHIP_VENDOR_STORAGE_UPDATE_LOADER=y
    CONFIG_PM_DEVFREQ=y
    CONFIG_DEVFREQ_GOV_PERFORMANCE=y
    CONFIG_DEVFREQ_GOV_POWERSAVE=y
    CONFIG_DEVFREQ_GOV_USERSPACE=y
    CONFIG_ARM_ROCKCHIP_BUS_DEVFREQ=y
    CONFIG_ARM_ROCKCHIP_DMC_DEVFREQ=y
    CONFIG_ARM_ROCKCHIP_DMC_DEBUG=y
    CONFIG_DEVFREQ_EVENT_ROCKCHIP_NOCP=y
    CONFIG_MEMORY=y
    CONFIG_IIO=y
    CONFIG_IIO_BUFFER=y
    CONFIG_IIO_KFIFO_BUF=y
    CONFIG_IIO_TRIGGER=y
    CONFIG_ROCKCHIP_SARADC=y
    CONFIG_SENSORS_ISL29018=y
    CONFIG_SENSORS_TSL2563=y
    CONFIG_TSL2583=y
    CONFIG_IIO_SYSFS_TRIGGER=y
    CONFIG_PWM=y
    CONFIG_PWM_ROCKCHIP=y
    CONFIG_PHY_ROCKCHIP_CSI2_DPHY=y
    CONFIG_PHY_ROCKCHIP_DP=y
    CONFIG_PHY_ROCKCHIP_EMMC=y
    CONFIG_PHY_ROCKCHIP_INNO_HDMI_PHY=y
    CONFIG_PHY_ROCKCHIP_INNO_MIPI_DPHY=y
    CONFIG_PHY_ROCKCHIP_INNO_USB2=y
    CONFIG_PHY_ROCKCHIP_INNO_USB3=y
    CONFIG_PHY_ROCKCHIP_INNO_VIDEO_COMBO_PHY=y
    CONFIG_PHY_ROCKCHIP_NANENG_COMBO_PHY=y
    CONFIG_PHY_ROCKCHIP_NANENG_EDP=y
    CONFIG_PHY_ROCKCHIP_PCIE=y
    CONFIG_PHY_ROCKCHIP_SNPS_PCIE3=y
    CONFIG_PHY_ROCKCHIP_TYPEC=y
    CONFIG_PHY_ROCKCHIP_USB=y
    CONFIG_ANDROID=y
    CONFIG_ROCKCHIP_EFUSE=y
    CONFIG_ROCKCHIP_OTP=y
    CONFIG_TEE=y
    CONFIG_OPTEE=y
    CONFIG_RK_FLASH=y
    CONFIG_RK_SFC_NAND=y
    CONFIG_RK_SFC_NAND_MTD=y
    CONFIG_RK_SFC_NOR=y
    CONFIG_RK_SFC_NOR_MTD=y
    CONFIG_RK_HEADSET=y
    CONFIG_ROCKCHIP_RKNPU=y
    CONFIG_EXT4_FS=y
    CONFIG_EXT4_FS_POSIX_ACL=y
    CONFIG_EXT4_FS_SECURITY=y
    CONFIG_XFS_FS=y
    # CONFIG_DNOTIFY is not set
    CONFIG_FUSE_FS=y
    CONFIG_ISO9660_FS=y
    CONFIG_JOLIET=y
    CONFIG_ZISOFS=y
    CONFIG_VFAT_FS=y
    CONFIG_FAT_DEFAULT_CODEPAGE=936
    CONFIG_FAT_DEFAULT_IOCHARSET="utf8"
    CONFIG_NTFS_FS=y
    CONFIG_TMPFS=y
    CONFIG_TMPFS_POSIX_ACL=y
    CONFIG_JFFS2_FS=y
    CONFIG_UBIFS_FS=y
    CONFIG_UBIFS_FS_ADVANCED_COMPR=y
    CONFIG_SQUASHFS=y
    CONFIG_PSTORE=y
    CONFIG_PSTORE_CONSOLE=y
    CONFIG_PSTORE_RAM=y
    CONFIG_NFS_FS=y
    CONFIG_NFS_V3_ACL=y
    CONFIG_NFS_V4=y
    CONFIG_NFS_SWAP=y
    CONFIG_NLS_DEFAULT="utf8"
    CONFIG_NLS_CODEPAGE_437=y
    CONFIG_NLS_CODEPAGE_936=y
    CONFIG_NLS_ASCII=y
    CONFIG_NLS_ISO8859_1=y
    CONFIG_NLS_UTF8=y
    CONFIG_UNICODE=y
    # CONFIG_CRYPTO_ECHAINIV is not set
    CONFIG_CRYPTO_SHA512=y
    CONFIG_CRYPTO_TWOFISH=y
    CONFIG_CRYPTO_ANSI_CPRNG=y
    CONFIG_CRYPTO_USER_API_HASH=y
    CONFIG_CRYPTO_USER_API_SKCIPHER=y
    CONFIG_CRYPTO_DEV_ROCKCHIP=y
    CONFIG_CRYPTO_DEV_ROCKCHIP_DEV=y
    CONFIG_CRC_CCITT=y
    CONFIG_CRC_T10DIF=y
    CONFIG_CRC7=y
    # CONFIG_XZ_DEC_X86 is not set
    # CONFIG_XZ_DEC_POWERPC is not set
    # CONFIG_XZ_DEC_IA64 is not set
    # CONFIG_XZ_DEC_SPARC is not set
    CONFIG_PRINTK_TIME=y
    CONFIG_DYNAMIC_DEBUG=y
    CONFIG_DEBUG_INFO=y
    CONFIG_MAGIC_SYSRQ=y
    CONFIG_MAGIC_SYSRQ_DEFAULT_ENABLE=0
    CONFIG_SCHEDSTATS=y
    CONFIG_DEBUG_SPINLOCK=y
    CONFIG_DEBUG_CREDENTIALS=y
    CONFIG_RCU_CPU_STALL_TIMEOUT=60
    CONFIG_FUNCTION_TRACER=y
    CONFIG_BLK_DEV_IO_TRACE=y
    CONFIG_LKDTM=y

# 显示Ubuntu桌面到屏幕

    sudo FRAMEBUFFER=/dev/fb0 startx
    sudo systemctl restart gdm

/etc/X11/下配置xorg.conf

    Section "Device"
        Identifier "FBDev"
        Driver "fbdev"
    EndSection

    Section "Monitor"
        Identifier "BuiltinMonitor"
        HorizSync 30-70
        VertRefresh 50-75
    EndSection

    Section "Screen"
        Identifier "BuiltinScreen"
        Monitor "BuiltinMonitor"
        Device "FBDev"
        DefaultDepth 16
        SubSection "Display"
            Depth 16
        EndSubSection
    EndSection

    Section "ServerLayout"
        Identifier "BuiltinLayout"
        Screen "BuiltinScreen"
    EndSection

