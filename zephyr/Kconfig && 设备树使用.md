## Kconfig 参数

#### 语法

```
# 定义一个菜单，用于在 menuconfig 中进行逻辑分组
menu "My Project Configurations"

# 1. 功能参数开关 (通常在 prj.conf 中设置)
config MY_APP_ENABLE_FEATURE
    bool "Enable advanced application feature"
    default n
    help
      "选中此项将启用应用程序的高级业务逻辑。"

# 2. 带依赖关系的数值参数
config MY_APP_BUFFER_SIZE
    int "Size of the internal message buffer"
    default 128
    range 64 1024
    depends on MY_APP_ENABLE_FEATURE
    help
      "定义应用层缓存区大小，仅在启用高级功能时有效。"

# 3. 硬件相关参数 (通常与板级支持或设备驱动相关)
# 这里的参数虽然定义在 Kconfig，但其值往往根据硬件特性在配置文件中指定
config MY_HW_SAMPLING_RATE_HZ
    int "Hardware sensor sampling rate (Hz)"
    default 100
    help
      "设置特定硬件传感器的采样频率。"

# 4. 选择型配置 (枚举)
choice MY_HW_COMM_PROTOCOL
    prompt "Hardware communication protocol"
    default MY_HW_USE_UART

    config MY_HW_USE_UART
        bool "Use UART for communication"

    config MY_HW_USE_SPI
        bool "Use SPI for communication"
endchoice

# 5. 反向依赖 (Select)
config MY_HW_INIT_MODULE
    bool "Initialize hardware module"
    select GPIO  # 强制开启 Zephyr 原生的 GPIO 驱动支持
    help
      "开启此项会自动选中并启用 Zephyr 的 GPIO 子系统。"

endmenu
```

#### 最佳实践

* **系统级** 根目录下的Kconfig
  * 核心内容：全局功能开关
  * 调用方式：通过 `source` 引入模块Kconfig，`source "src/module/pid/Kconfig"`
* **模块级** 比如卡尔曼滤波参数
  * 核心内容：私有参数、硬件依赖声明。包括Kconfig 和 Kconfig.defconfig（默认值）



## 硬件描述

#### 基础设备树 .dts

```
/* --- 第一部分：包含文件 (The Includes) --- */
/dts-v1/;
#include <st/f4/stm32f407Xg.dtsi> // SoC 的定义文件（包含所有的寄存器地址、中断等“硬”信息）
#include <st/f4/stm32f407v(e-g)tx-pinctrl.dtsi> // 引脚复用定义（预定义了 USART_TX_PA2 等标签）
#include <zephyr/dt-bindings/input/input-event-codes.h> // 包含标准常量定义（如输入事件代码、GPIO 极性）

/ {
	/* 板级识别信息 */
    model = "My Engineering Project Board";
    compatible = "my-vendor,my-stm32-board";

    /* --- 第二部分：系统指定配置 (Chosen) --- */
    // chosen节点作用：告诉 Zephyr 内核，系统组件该去用哪一个具体的硬件
    chosen {
        zephyr,console = &usart2;    // printk 的输出去向
        zephyr,shell-uart = &usart2; // Shell 交互界面的去向
        zephyr,sram = &sram0;        // 内存起始位置
        zephyr,flash = &flash0;      // 代码存储位置
    };

    /* --- 第三部分：硬件定义节点(总线) --- */
    leds {
        compatible = "gpio-leds"; // 宏定义，生成C语言宏：这一组子节点都是 LED
        // 子节点定义
        blue_led: led_0 {
            // 引用 gpio 端口 D，第 15 号脚，高电平点亮
            gpios = <&gpiod 15 GPIO_ACTIVE_HIGH>; 
            label = "User LED Blue";
        };
    };

    gpio_keys {
        compatible = "gpio-keys";
        // 子节点定义 user_button
        user_button: button {
            gpios = <&gpioa 0 GPIO_ACTIVE_HIGH>;
            zephyr,code = <INPUT_KEY_0>; // 定义按下时触发的逻辑键值
        };
    };

    /* --- 第四部分：别名 (Aliases) --- */
    // 提供统一的代号，让 C 代码实现“跨板兼容”：不管它在 A 板是 PD15 还是 B 板的 PA1
    aliases {
        led0 = &blue_led;
        sw0 = &user_button;
    };
};

/* --- 第五部分：底层硬件参数配置 (Hardware Tuning) --- */
// 作用：使用 &标签 语法，进入 SoC 的预定义节点修改具体参数

// 1. 配置时钟树（这是 SoC 运行的基础）
&clk_hse {
    clock-frequency = <DT_FREQ_M(8)>; // 告诉内核：我板子上焊的是 8MHz 晶振
    status = "okay";
};

&pll {
    div-m = <8>;
    mul-n = <336>;
    div-p = <2>;
    clocks = <&clk_hse>; // PLL 的输入源是 HSE
    status = "okay";
};

// 2. 配置外设引脚 (Pinctrl)
&usart2 {
    // pinctrl-0 定义默认状态下的引脚分配
    pinctrl-0 = <&usart2_tx_pa2 &usart2_rx_pa3>; 
    pinctrl-names = "default";
    current-speed = <115200>;
    status = "okay"; // 显式开启设备，否则 SoC 默认关闭以节能
};

// 3. 总线上的二级设备 (I2C/SPI 节点)
&i2c1 {
    pinctrl-0 = <&i2c1_scl_pb6 &i2c1_sda_pb9>;
    pinctrl-names = "default";
    status = "okay";

    // 定义挂在 I2C 总线上的具体芯片
    my_sensor: sensor@4a {
        compatible = "bosch,bme280"; // 匹配传感器驱动
        reg = <0x4a>;                // 传感器的 I2C 器件地址
    };
};
```

#### 设备树覆盖 .overlay

本质语法同上，只是一个额外的补丁

```
/* --- 场景 1：修改已有外设的参数、逻辑定义 --- */
// 覆盖原有参数
&usart2 {
    current-speed = <9600>; 
};

// 修改原有逻辑极性
&blue_led {
    gpios = <&gpiod 15 GPIO_ACTIVE_LOW>;
};

//关掉原有功能
&zephyr_udc0 {
    status = "disabled";
};

/* --- 场景 2：在已有总线上新增节点 --- */
&i2c1 {
    status = "okay"; // 确保总线是开启的

    /* 这是一个新节点，原本板级的 .dts 里没有它 */
    my_new_sensor: bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
        label = "EXT_SENSOR";
    };
};

/* --- 场景 3：新增设备 --- */
&usart1 {
    status = "okay";
    pinctrl-0 = <&usart1_tx_pa9 &usart1_rx_pa10>;
    pinctrl-names = "default";
    current-speed = <115200>;
};
```

#### 使用第三方开发板：

* 参考官方适配好的板子：`zephyr/boards/<arch>/<board_name>/`
* 按如下结构添加自定义开发板

```
my_custom_project/
└── boards/
    └── <vendor>/
        └── <board_name>/
            ├── <board_name>.dts          # 设备树定义
            ├── <board_name>_defconfig    # 默认内核配置
            ├── Kconfig.board             # Kconfig 定义
            ├── board.yml                 # 板子元数据 (Zephyr v3.7+)
            └── CMakeLists.txt            # 板级构建脚本
```