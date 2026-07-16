## GPIO

```c
//头文件 (driver为文件夹名称，表示gpio在该文件夹下)
#include "driver/gpio.h" 

void led_init(void)
{
    gpio_config_t gpio_cfg=
    {  //将结构体成员一一引出
        .intr_type= GPIO_INTR_DISABLE, //是否开启GPIO中断
        .mode=GPIO_MODE_INPUT_OUTPUT, //配置输入输出模式
        .pin_bit_mask= 1ull<<GPIO_NUM_38, //引脚掩码，选择控制引脚
        .pull_down_en=GPIO_PULLDOWN_DISABLE, //设置下拉使能或关闭
        .pull_up_en=GPIO_PULLUP_ENABLE, //设置上拉使能或关闭
    };
    esp_err_t err;
    err= gpio_config(&gpio_config);  //将结构体指针传入
     if(err != ESP_OK)
    {
        printf("gpio init error! \n");
    }
}

void app_main(void)
{   
    led_init();
    while (1)
    {
         gpio_set_level(GPIO_NUM_38, 1);  //高电平
         vTaskDelay(500);
         gpio_set_level(GPIO_NUM_38, 1); 
         vTaskDelay(500);
    }
}
```
------------
## 中断

```c
void exti_isr_handler(void*arg)  //中断回调函数
{
    //中断要执行的逻辑
}

void exti_init(void)
{
	gpio_config_t gpio_cfg={
        .intr_type= GPIO_INTR_NEGEDGE,  //下降沿触发中断
        .mode=GPIO_MODE_INPUT, //配置输入模式
        .pin_bit_mask= 1ull<<GPIO_NUM_9, //配置引脚
        .pull_down_en=GPIO_PULLDOWN_DISABLE, //设置下拉使能或关闭
        .pull_up_en=GPIO_PULLUP_ENABLE, //设置上拉使能或关闭
    };
    gpio_config(&gpio_cfg);

    gpio_install_isr_service(ESP_INTR_FLAG_EDGE); //注册中断

    gpio_isr_handler_add(GPIO_NUM_9,exti_isr_handler,NULL);  //将IO口与中断服务函数匹配
    gpio_intr_enable(GPIO_NUM_9);  //开启中断
}
```

---------



## 定时器函数

###### 1.通用定时器（硬件定时器）

```C
#include "driver/gptimer.h"

gptimer_handle_t gptim;  //句柄（给定时器取名字）
uint8_t flag_timer=0;
bool TimerCallback(gptimer_handle_t timer, const gptimer_alarm_event_data_t *edata, void *user_ctx)//回调函数
{
    flag_timer=1;
    return 0;
}

void gptimer_init(void)
{
    gptimer_config_t gptimer_cfg={ 
        .clk_src=GPTIMER_CLK_SRC_DEFAULT,  //选择时钟源
        .direction=GPTIMER_COUNT_UP,  //向上计数
        .flags.intr_shared=0,  
        .intr_priority=0,  //中断优先级
        .resolution_hz=1000000,  //计数器分辨率（多少节拍计数器加一）
    };
    gptimer_new_timer(&gptimer_cfg,gptim);  //配置时钟源和计数器

    gptimer_alarm_config_t gptimer_alarm_cfg={
        .alarm_count=500000,  //比较器（CCR)的目标值
        .flags.auto_reload_on_alarm=1,  //是否打开自动重置开关
        .reload_count=0,   //重置值
    };
    gptimer_set_alarm_action(gptim,&gptimer_alarm_cfg);  //配置比较器

    gptimer_event_callbacks_t event_cfg={
        .on_alarm= TimerCallback,  //配置中断程序
    };
    gptimer_register_event_callbacks(gptim,&event_cfg,NULL);  //配置报警事件

    gptimer_enable(gptim);  //使能通用定时器
    gptimer_start(gptim);  //开启通用定时器
}
```

###### 2.系统定时器（软件定时器）

```c
#include "esp_timer.h"

void esptimer_callback(void*arg)
{
    //中断逻辑
}
void esptim_init(void)
{
    esp_timer_handle_t esptim;

    esp_timer_create_args_t esptim_cfg={
        .arg=NULL,
        .callback=&esptimer_callback,  //中断回调函数
        .dispatch_method=ESP_TIMER_TASK,
        .name="mytim",  //定时器名称
        .skip_unhandled_events=true,
    };
    esp_timer_create(&esptim_cfg,esptim);  //创建定时器

    esp_timer_start_periodic(esptim,500000);  //配置比较器值（CCR），并开启定时器

}
```

###### 2.PWM

```C
#include "driver/ledc.h"

void pwm_init(void)
{
    ledc_timer_config_t ledctimer_cfg={
        .clk_cfg=LEDC_AUTO_CLK,   //时钟源
        .duty_resolution=LEDC_TIMER_10_BIT,  //分辨率（分为多少份，1024=2**10）
        .freq_hz=1000,  //脉冲频率
        .speed_mode=LEDC_LOW_SPEED_MODE,
        .timer_num=LEDC_TIMER_1,  //选定时器
    };
    ledc_timer_config(&ledctimer_cfg);  //配置时钟源和定时器

    ledc_channel_config_t ledc_channel_cfg={
        .channel=LEDC_CHANNEL_1,
        .duty=512,  //高电平所占分辨率
        .flags.output_invert=0,
        .gpio_num=GPIO_NUM_38,
        .hpoint=0,
        .intr_type=LEDC_INTR_DISABLE,
        .speed_mode=LEDC_LOW_SPEED_MODE,
        .timer_sel=LEDC_TIMER_1,
    };
    ledc_channel_config(&ledc_channel_cfg);
}

void duty_set(uint16_t duty)
{
   ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_1,duty);  //修改占空比
   ledc_update_duty(LEDC_LOW_SPEED_MODE,LEDC_CHANNEL_1);  //更新占空比
}
```

-----

## ADC

###### 1.连续转换模式

```c
#include "hal/adc_types.h"
#include "esp_adc/adc_continuous.h"

adc_continuous_handle_cfg_t adc_continuous_handle;

uint8_t *data_value;
bool continuous_callback(adc_continuous_handle_t handle,const adc_continuous_evt_data_t *edata, void *user_data)
{
    data_value=edata->conv_frame_buffer;  //转换帧地址
    if(edata->size==8)
    {
        return true;
    }
    else
    {
        return false;
    }
}

void adc_init(void)
{
    adc_continuous_handle_cfg_t adc_continuous_cfg={
        .conv_frame_size=8,  //转换帧大小
        .max_store_buf_size=1024,   //转换结果的最大值
    };
    adc_continuous_new_handle(&adc_continuous_cfg,&adc_continuous_handle);  //配置转换表

    adc_digi_pattern_config_t adc_digi_arr[]={
        {
            .atten= ADC_ATTEN_DB_11,  //衰减系数
            .bit_width=ADC_BITWIDTH_12,  //分辨率
            .channel= ADC_CHANNEL_3,
            .unit=ADC_UNIT_1,
        },
        {
            .atten=ADC_ATTEN_DB_11,
            .bit_width=ADC_BITWIDTH_12,
            .channel=ADC_CHANNEL_4,
            .unit=ADC_UNIT_1,
        }
    };

    adc_continuous_config_t continuous_config_structure={
        .adc_pattern=adc_digi_arr,  //转换表
        .conv_mode=ADC_CONV_SINGLE_UNIT_1,
        .format=ADC_DIGI_OUTPUT_FORMAT_TYPE2,
        .pattern_num=2,
        .sample_freq_hz=2000,
    };
    adc_continuous_config(&adc_continuous_handle,&continuous_config_structure);  //配置总控制器

    adc_continuous_evt_cbs_t evt_structure={
        .on_conv_done=continuous_callback,  //触发条件
    };
    adc_continuous_register_event_callbacks(&adc_continuous_handle,&evt_structure, NULL);

    adc_continuous_start(&adc_continuous_handle);
}
```

###### 2.单次转换

```c
#include "esp_adc/adc_oneshot.h"

adc_oneshot_unit_handle_t adc_unit_t_handle;
void adc_oneshot_init(void)
{
    adc_oneshot_unit_init_cfg_t oneshot_structure={
        .clk_src=ADC_RTC_CLK_SRC_DEFAULT,
        .ulp_mode=ADC_ULP_MODE_DISABLE,
        .unit_id=ADC_UNIT_1,
    };
    adc_oneshot_new_unit(&oneshot_structure,&adc_unit_t_handle);

    adc_oneshot_chan_cfg_t oneshot_chan_structure={
        .atten=ADC_ATTEN_DB_11,
        .bitwidth=ADC_BITWIDTH_12,
    };
    adc_oneshot_config_channel(adc_unit_t_handle,ADC_CHANNEL_3,&oneshot_chan_structure);
    adc_oneshot_config_channel(adc_unit_t_handle,ADC_CHANNEL_4,&oneshot_chan_structure);
}
adc_oneshot_read(adc_unit_handle,ADC_CHANNEL_3,&adc_value_rl);
```

-----------------

## 串口

```c
#include "driver/uart.h"

void uart_init(void)
{
    uart_config_t uart_structure={
        .baud_rate=115200,  //波特率
        .data_bits= UART_DATA_8_BITS,  //数据位长度
        .flow_ctrl=UART_HW_FLOWCTRL_DISABLE,  
        .parity=UART_PARITY_DISABLE,
        .rx_flow_ctrl_thresh=100,
        .source_clk=UART_SCLK_DEFAULT,  //时钟源
        .stop_bits=UART_STOP_BITS_1 ,  //停止位
    };
    uart_param_config(UART_NUM_1,&uart_structure);

    // 功能：为 UART1 外设分配物理 GPIO 引脚
    uart_set_pin(UART_NUM_1,GPIO_NUM_17,GPIO_NUM_18,-1,-1);

    // 功能：安装UART驱动并配置缓冲区、事件队列等
    uart_driver_install(UART_NUM_1,1024,1024,0,NULL,0);
}

void uart_pron()
{
    int length;
    uint8_t receive_data[1024];
    
    // 从UART1读取数据，返回实际读取到的字节数
    length = uart_read_bytes(UART_NUM_1, receive_data, 1024, 100);

    // 判断是否读取到有效数据
    if(length > 0 )
    {
        // 将读取到的数据通过UART1发送出去
        uart_write_bytes(UART_NUM_1, receive_data, length);
        // 等待UART1发送完成，清空发送缓冲区
        uart_flush(UART_NUM_1);
    }
}
```



------------------

## I2C

```C
```

----

## SPI

```C
```

-----

## WI-FI

#### **==工作模式==**
**STA模式：** 连接WiFi上网
**AP模式：** 自己当热点
**混合模式** 

**==连接方式：==** 硬编码配网、SoftAP配网、SmartConfig配网、ESP—Touch配网

![[Pasted image 20260716164554.png]]

**NVS存储:** 非易失性存储，用于在Flash中存储WiFi的校准数据和用户配置。初始化WiFi前必须先初始化NVS，否则可能导致WiFi校准不准或出现错误。

**LWIP协议栈：** WiFi设备必需遵循的统一标准，只有这样数据才能正确传输和解析。

**循环事件组：** ESP32把WiFi相关的各种状态变化都封装成事件，可以分为 **WiFi事件** 和 **IP事件** ，而循环事件组就是专门用于监听这些事件，一旦触发了这些事件就会执行对应的事件函数。例如：STA模式开启WiFi后要等待连接，什么时候知道连上了呢？就是通过循环事件组进行监听。

#### **==开发流程：**
1. **初始化NVS存储、LWIP协议栈
2. **绑定网络接口：** 建立WiFi模块与LWIP协议栈之间的连接通道，使数据可以在两者间流转。
3. **初始化WiFi配置：** 设置工作模式和相关参数
4. **创建事件循环组并注册事件处理函数：** 监听WiFi状态变化，事件触发时自动执行对应的处理逻辑
5. **启动WiFi并连接



#### Station 模式
```c
//事件函数
void WiFi_sta_event_handler(void* event_handler_arg, esp_event_base_t event_base, int32_t event_id, void* event_data)
{
	if(event_base == WIFI_EVENT)    //先判断事件组
	{
		if(event_id == WIFI_EVENT_STA_START)  //再判断事件编号
		{
			esp_wifi_connect();   //开始连接
		}
		else if(event_id == WIFI_EVENT_STA_CONNECTED)
		{
			//已连接
		}
		else if(event_id == WIFI_EVENT_STA_DISCONNECTED)
		{
			//未连接
		}
	}
	else if(event_base == IP_EVENT)   //事件组：获取IP地址
	{
		if(event_id == IP_EVENT_STA_GOT_IP)
		{
			esp_netif_ip_info_t *event = (esp_netif_ip_info_t *)event_data
		}
	}
}

//初始化
void WiFi_sta_init(void)
{
	esp_netif_init();   //初始化LWIP协议栈
	esp_netif_create_default_wifi_sta();  //将WiFi模块与LWIP技术栈连接
	
	wifi_init_config_t cfg= WIFI_INIT_CONFIG_DEFAULT();  //把WiFi配置成默认的设置，后面的宏定义即默认的WiFi配置
	esp_wifi_init(&cfg);
	esp_wifi_set_mode(WIFI_MODE_STA);     //设置模式
	wifi_config_t wifista_cofig = {
		.sta={
			.ssid= "",
			.password= "",
		}
	}
	esp_wifi_set_config(WIFI_IF_STA,&wifista_config);   //模式配置
	
	esp_event_loop_create_default();  //创建循环事件组用于监听WiFi模块的各种事件
	esp_event_handler_register(WIFI_EVENT,   //要注册的事件组名称
								ESP_EVENT_ANY_ID,    //事件组中具体的事件编号
								&WiFi_sta_event_handler,  //创建事件函数，并将事件组与事件函数匹配起来
								NULL);   //向事件函数的第一个参数传送的数据
	esp_event_handler_register(IP_EVENT,   //要注册的事件组名称
								IP_EVENT_STA_GOT_IP,    //事件组中具体的事件编号
								&WiFi_sta_event_handler,  //创建事件函数，并将事件组与事件函数匹配起来
								NULL);   //向事件函数的第一个参数传送的数据
	
	esp_wifi_start();
}

```

#### AP模式 
```c 
void WiFi_sta_event_handler(void* event_handler_arg, esp_event_base_t event_base, int32_t event_id, void* event_data)
{
	if(event_base == WIFI_EVENT)
	{
		if(event_id == WIFI_EVENT_AP_STACONNECTED)
		{
			
		}
		else if(event_id == WIFI_EVENT_AP_STADISCONNECTED)
		{
			
		}
	}
}

//初始化
void WiFi_sta_init(void)
{
	esp_netif_init();   //初始化LWIP协议栈
	esp_netif_create_default_wifi_ap();  //将WiFi模块与LWIP技术栈连接
	
	wifi_init_config_t cfg= WIFI_INIT_CONFIG_DEFAULT();  //把WiFi配置成默认的设置，后面的宏定义即默认的WiFi配置
	esp_wifi_init(&cfg);
	esp_wifi_set_mode(WIFI_MODE_AP);     //设置模式
	wifi_config_t wifiap_cofig = {
		.ap={
			.ssid= "",
			.password= "",
			.ssid_len=,
			.max_connection=,
			.authmode=,
		}
	}
	esp_wifi_set_config(WIFI_IF_AP,&wifiap_config);   //模式配置
	
	esp_event_loop_create_default();  //创建循环事件组用于监听WiFi模块的各种事件
	esp_event_handler_register(WIFI_EVENT,   //要注册的事件组名称
								ESP_EVENT_ANY_ID,    //事件组中具体的事件编号
								&WiFi_ap_event_handler,  //创建事件函数，并将事件组与事件函数匹配起来
								NULL);   //向事件函数的第一个参数传送的数据
	
	
	esp_wifi_start();
}
```

#### 获取网络时间
```c

```

------------

## BLE

**蓝牙标准：** **GAP连接标准** 和 **GATT通信标准** 

##### GAP连接标准下：
**外围设备：** 通常是传感器、智能灯泡这类==拥有数据的设备==
**中央设备：** 通常是手机、电脑这类==可以主动发起连接的设备==

**广播：** 外围设备会广播自己的存在，等待其他设备的连接
**扫描：** 中央设备会主动扫描周围的广播，以寻找目标设备

##### GATT通信标准下：
**服务：** 代表设备的一项功能，例如：温度计服务，电池服务等                 ==（UUID，特征）==
**特征：** 是蓝牙传输的基本单元，例如：温度、电池电量等。值承载特征的具体数据。==（UUID,值，属性）==
**属性：** 属性定义了特征的操作权限
**UUID:** 通用唯一识别码，用于特征识别
==通信之前，外围设备向中央设备发送配置文件，配置文件中可以包含多个服务，每个服务由一个服务UUID和多个特征组成，每个特征又都有其独有的UUID、值和属性==

![[Pasted image 20260716221903.png]]



![[Pasted image 20260716223442.png]]

**NimBLE协议栈:** 统一的通信标准，让不同设备互相兼容， 
**蓝牙协议循环事件组：** 专门用于监听蓝牙协议栈产生的事件，针对不同的事件触发不同的事件函数，分为连接事件和通信事件

##### 开发流程：
1. **初始化协议栈，配置GAP/GATT，配置设备信息**
2. **注册服务和特征列表**
3. **注册回调函数**
4. **开启广播** 
5. **创建监听任务，并创建事件函数** 



```C

```

-------------

#### MQTT

```C
```

------------
#### FreeRTOS
