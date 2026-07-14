## IO
```c
// 1. 数字输出
digitalWrite(2,HIGH);  //给2号引脚输出高电平
digitalWrite(2,LOW);   //给2号引脚输出低电平

// 2. 数字输入
digtalRead(4);   //读取4号引脚状态（返回值：HIGH/LOW）

// 3. 引脚模式设置
pinMode(2, OUTPUT);  //2号引脚输出模式
pinMode(3, INPUT);   //3号引脚输入模式
pinMode(4, INPUT_PULLUP);   //输入上拉模式

```
--------------
## 模拟信号
```c
//模拟输入
analogRead(4);  //读取4号引脚模拟值

//PWM模拟输出
				analogWrite(pin,duty)  //引脚号，数值
// 1.设置通道、频率、分辨率
ledcSetup(channel,freq,resolution);
// 2.将引脚绑定到通道
ledcAttachPin(pin, channel);
// 3.输出PWM值
ledcWrite(channel, dutyCycle);
```
-----------
## 时间控制函数
```c
//毫秒延时
delay(100)

//微秒延时
delayMicroseconds(100)

//获取运行时间
millis()//获取程序开始运行到现在的时间，单位毫秒（unsigned long）
```
--------------
## 串口通信函数
```c
//初始化串口
serial.begin(115200)   //启动串口通信，设置波特率

//打印信息
Serial.print()     //打印后不换行
Serial.println()   //打印后换行

//读取串口
Serial.read()   //读取从电脑或其他模块发送到ESP32的数据
Serial.available()   //检查串口是否有数据
```
------------
## Wi-Fi
```c
// 连接WiFi
WiFi.begin(ssid, password) //WiFi名称和密码

// 获取连接状态
WiFi.status   //返回值：WL_CONNECTED/WL_DISCONNECTED/WL_CONNECTION_LOST

//获取IP地址
WiFi.localIP()
```
----------------
## 蓝牙串口
```c
// 开启蓝牙
SerialBT.begin("ESP32_LED")   //填设备名称

// 读取蓝牙数据
SerialBT.available()   
```
-----------
## 其他
```c
// 随机数生成
random(1,101)  //生成1-100的随机数

// 映射数值 (将一个范围的值映射到另一个范围)
map(sensorValue,0,4095,0,255) //将传感器0-4095的值，映射到0-255

// 限制值的范围
constrain(value,0,1000) //将值限制在0--1000之间

//
```
