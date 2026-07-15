#### GPIO

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
#### 中断

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



#### 定时器函数

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

#### ADC

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

#### 串口

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

#### I2C

```C
```

----

#### SPI

```C
```

-----

#### WI-FI连接函数

**==工作模式==**
**STA模式：** 连接WiFi上网
**AP模式：** 自己当热点

**==连接方式：==** 硬编码配网、SoftAP配网、SmartConfig配网、ESP—Touch配网
###### Station 模式
```c
// 初始化WIFI
esp_netif_create_default_wifi_sta();
wifi_config_t cfg= WOFO_INIT_CONFIG_DEFAULT();
esp_wifi_init(&cfg);

// 配置WiFi
wifi_config_t wifi_config={
 .sta={
	 .ssid="你的WiFi名"
	 .password="你的密码"
	 }，
}；
esp_wifi_set_mode(WIFI_MODE_STA);
esp_wifi_set_config(WIFI_IF_STA,&wifi_config);

// 开始连接
esp_wifi_start();
esp_wifi_connect();
```
###### WiFi 事件处理
```c
// 注册事件处理函数
esp_event_handler_register(WIFI_EVENT,ESP_EVENT_ANY_ID,wifi_event_handler,NULL);
esp_event_handler_register(IP_EVENT,IP_EVENT_STA_GOT_IP,got_ip_handler,NULL);
```




------------

#### BLE

```C
```

-------------

#### MQTT

```C
```

------------
#### FreeRTOS
