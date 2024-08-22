---
layout: post
title:  "NodeMCU门铃扩展"
date:   2024-08-22 12:30:00 +0800
category: Dev
---

# 缘起
去年我买了个京东鹰眼智能门锁。作为一个入门款门锁，它支持指纹解锁，ID卡解锁，密码解锁，还支持可视门铃功能。用了一段时间之后，我发现一个问题，它的门铃喇叭好像是装在门外的，这就导致有人按门铃时，门外听得清楚，但是在屋里的人，却很难听到门铃声，只能看到液晶门板亮了。由于家里日常老人在家，听力不好，很难听到外卖或者快递的按门铃的声音。这个问题就如鲠在喉，一直想找办法解决。

# 抓包尝试
一开始我的想法是截获门锁和服务端的通信信息来感知有人按门铃，然后想办法播放声音。通过在路由器上抓包，我可以大致根据特征（上报的ip，数据包的大小）分析出门铃上报的消息。

当然，路由器的性能孱弱，把所有的逻辑放在路由器上跑我总感觉有点不靠谱，我又找了个办法，通过ssh将tcpdump生成的标准输出转到虚拟机上用twireshark处理，例如：
```bash
ssh admin@192.168.1.1 '/usr/sbin/tcpdump -i br0 -w - -s 0 host 192.168.1.211' |tshark -T fields -e frame.time -e ip.src -e ip.dst -e data  -i -
```
通过这种转接，后续的定制脚本处理就简化很多。

但是这种办法也有缺点，且不说在路由器上跑tcpdump时间长了会不会有性能问题，标准输出本身还有buffer刷新问题，并不是每个包都能实时打印出来，这就引入一定的时延，说不定等收到门铃上报的数据包时，按门铃的人早就离开了。

于是，我又打算通过iptables，将相关的数据包转到虚拟机，虚拟机再转到真正后端。这个方案有可能可行，但是它引入了系统复杂度，增加了门锁和它的服务端断链的可能性。

# 官方渠道
这时我又从产品本身入手，这个门锁是通过一个叫智能生活的App管控的，几年前有个网文，指出可以通过安卓App安装包逆向获取token，通过token来发送指令管理这个App接管的物联网设备。这个文章有段时间，不知道可信度高不高。不过我顺藤摸瓜，倒是发现其实这个智能生活App是涂鸦平台的。涂鸦其实是有开发者平台的，我就注册了一个开发者平台，试用了它的API。在涂鸦开发者平台将智能生活App管理的设备托管给该平台之后，可以通过涂鸦API获取设备上报的信息。但是其中有些参数是产品侧自行定义的，在没有产品提供开发手册情况下，自行摸索有点难度。还有一点，涂鸦的API试用一个月后需要付费使用，虽然体验版是0元，但是如果需要每个月订购一次，还是很麻烦。

然后，我又发现涂鸦的设备其实支持Pulsar消息队列，通过消息队列可以获取到设备上报消息。
参考的消息订阅脚本如下：
```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import base64
import sys
import hashlib
import ssl
import websocket
import json
import time
import traceback
import requests
try:
    import thread
except ImportError:
    import _thread as thread

# env
MQ_ENV_PROD = "event"
MQ_ENV_TEST = "event-test"

# accessId, accessKey，serverUrl，MQ_ENV
ACCESS_ID = "xxxxxx"
ACCESS_KEY = "xxxxxxxxxx"
WSS_SERVER_URL = "wss://mqe.tuyacn.com:8285/"
MQ_ENV = MQ_ENV_TEST

# basic config
WEB_SOCKET_QUERY_PARAMS = "?ackTimeoutMillis=3000&subscriptionType=Failover"
SSL_OPT = {"cert_reqs": ssl.CERT_NONE}

CONNECT_TIMEOUT_SECONDS = 3
CHECK_INTERVAL_SECONDS = 3

PING_INTERVAL_SECONDS = 30
PING_TIMEOUT_SECONDS = 3

RECONNECT_MAX_TIMES = 1000

# global variable
global ws
ws = None

global reconnect_count
reconnect_count = 1

global connect_status
connect_status = 0

global bell_time
bell_time = 0


def play_song():
    url = "http://192.168.23.229/play"
    payload = '2='
    headers = {
        'Content-Type': 'application/x-www-form-urlencoded'
        }
    response = requests.request("POST", url, headers=headers, data=payload)
    print(response.text)

# topic env
def get_topic_url():
    return WSS_SERVER_URL + "ws/v2/consumer/persistent/" + ACCESS_ID + "/out/" + MQ_ENV + "/" + ACCESS_ID + "-sub" + WEB_SOCKET_QUERY_PARAMS


# handler message
def message_handler(payload):
    # print("payload:%s" % payload)
    try:
        global bell_time
        dataMap = json.loads(payload)
        toBedecryptedContentDataStr = dataMap['data']
        decryptContentDataStr = decrypt_by_aes(toBedecryptedContentDataStr, ACCESS_KEY)
        print("\ndecryptContentData={}".format(decryptContentDataStr))
        sys.stdout.flush()
        if decryptContentDataStr.find('doorbell')>-1:
            print("Doorbell detected.")
            current_time = int(time.time())
            if current_time - bell_time > 10:
                bell_time = current_time
                play_song()
        # message_object = json.loads(decryptContentDataStr.strip())
        # print(message_object.get('bizCode',''))
    except Exception as e:
        print(e)
        traceback.print_exc()


# decrypt
def decrypt_by_aes(raw, key):
    import base64
    from Crypto.Cipher import AES
    raw = base64.b64decode(raw)
    key = key[8:24]
    cipher = AES.new(key.encode('utf-8'), AES.MODE_ECB)
    raw = cipher.decrypt(raw)
    res_str = raw.decode('utf-8')
    res_str = eval(repr(res_str).replace('\\r', ''))
    res_str = eval(repr(res_str).replace('\\n', ''))
    res_str = eval(repr(res_str).replace('\\f', ''))
    return res_str


def md5_hex(md5_str):
    md = hashlib.md5()
    md.update(md5_str.encode('utf-8'))
    return md.hexdigest()


def gen_pwd():
    md5_hex_key = md5_hex(ACCESS_KEY)
    mix_str = ACCESS_ID+md5_hex_key
    return md5_hex(mix_str)[8:24]


def on_error(ws, error):
    print("on error is: %s" % error)


def reconnect():
    global reconnect_count
    print("ws-client connect status is not ok.\ntrying to reconnect for the %d time" % reconnect_count)
    reconnect_count += 1
    if reconnect_count < RECONNECT_MAX_TIMES:
        thread.start_new_thread(connect, ())


def on_message(ws, message):
    message_json = json.loads(message)
    payload = base64_decode_as_string(message_json["payload"])
    print("---\nreceived message origin payload: %s" % payload)
    # handler payload
    try:
        message_handler(payload)
    except Exception as e:
        print("handler message, a business exception has occurred,e:%s" % e)
    send_ack(message_json["messageId"])


def on_close(obj):
    print("Connection closed!")
    obj.close()
    global connect_status
    connect_status = 0


def connect():
    print("---\nws-client connecting...")
    sys.stdout.flush()
    ws.run_forever(sslopt=SSL_OPT, ping_interval=PING_INTERVAL_SECONDS, ping_timeout=PING_TIMEOUT_SECONDS)


def send_ack(message_id):
    json_str = json.dumps({"messageId": message_id})
    ws.send(json_str)


def base64_decode_as_string(byte_string):
    byte_string = base64.b64decode(byte_string)
    return byte_string.decode('ascii')


def main():
    header = {"Connection": "Upgrade",
              "username": ACCESS_ID,
              "password": gen_pwd()}

    websocket.setdefaulttimeout(CONNECT_TIMEOUT_SECONDS)
    global ws
    ws = websocket.WebSocketApp(get_topic_url(),
                                header=header,
                                on_message=on_message,
                                on_error=on_error,
                                on_close=on_close)

    thread.start_new_thread(connect, ())
    while True:
        time.sleep(CHECK_INTERVAL_SECONDS)
        global reconnect_count
        global connect_status
        try:
            if ws.sock.status == 101:
                print("ws-client connect status is ok.")
                sys.stdout.flush()
                reconnect_count = 1
                connect_status = 1
        except Exception:
            connect_status = 0
            reconnect()


if __name__ == '__main__':
    main()
```
如上面所示的脚本中，一旦探测到门铃消息，立马触发一个http调用播放音乐。
# wifi音乐播放器
## 物料
我们现在缺的就是一个响应http调用后播放音乐的系统，这个系统可以由如下几个元件构建：

1. NodeMCU ESP8622-12E
2. DFPlayer Mini
3. HS0038B
4. 闲置电视遥控器
5. 喇叭

NodeMCU是控制模块，提供Wifi和WebServer服务，并负责向DFPlayer Mini发送控制指令；HS0038B是一个红外接收模块，用于接收遥控器的信号，以暂停门铃音乐的播放，同时也可以让这个门铃兼做音乐播放器，通过遥控选曲和音量控制。
DFPlayer Mini这个模块主要由主芯片YS5200-24SS，功放模块8002D和TF卡槽构成。YS5200-24SS可以通过RX接口接收NodeMCU发送过来的串口指令，根据这些指令播放音乐。它有DAC_R和DAC_I这两个DAC输出，声音较小，适合耳机。SPK_1和SPK2是通过8002D驱动后的PWM信号，支持最高3W的输出，可以接喇叭。这个模块还可以通过IO1和IO2口接不同电阻作为输入，控制不同的功能。

## 接线

| ==NodeMCU== | ==DFPlayer Mini== |
| ----------- | ----------------- |
| VIN         | VCC               |
| GND         | GND               |
| D1          | BUSY              |
| D4          | RX                |

| ==DFPlayer Mini== |          |
| ----------------- | -------- |
| SPK_1             | speaker+ |
| SPK_2             | speaker- |


| NodeMCU | HS0038B |
| ------- | ------- |
| 3V3     | VCC     |
| GND     | GND     |
| D5      | OUT     |
## 代码
```C++
#include <Arduino.h>
#include <IRremoteESP8266.h>
#include <IRrecv.h>
#include <IRutils.h>
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include <ESP8266mDNS.h>
#include <DFPlayerMini_Fast.h>


//
DFPlayerMini_Fast myMP3;

#define DEBUG 0

#if DEBUG == 0
#define debug(x) Serial.print(x)
#define debugln(x) Serial.println(x)

#else
#define debug(x)
#define debugln(x)
#endif


String ssid = "xxxxx";
String password = "xxxx";

int mp3_volume = 20;
int mp3file_index = 1;

ESP8266WebServer server(80);
const int BUSY_PIN = 5;

void pause_mp3(){
    bool NotBusy = digitalRead(BUSY_PIN);  // this represents the busy pin of the dfmini
    if (NotBusy) {
      myMP3.resume();
      debugln("music resumed");
    }
    else {
      myMP3.pause();
      debugln("music paused");
    }
}

void play_mp3(){
    debugln("playMp3FolderTrack");
    myMP3.loop(mp3file_index);
    debugln("mp3 start");
}


void server_handles() {

  server.on("/play", []() {
      String response = server.arg("plain");
      mp3file_index = atoi(response.c_str());
      myMP3.loop(mp3file_index);
      server.send(200, "text/plain", "successfully play the music!\r\n");
    });

  server.on("/pause", []() {
      pause_mp3();
      server.send(200, "text/plain", "successfully pause/resume the music!\r\n");
    });

    server.on("/volume", []() {
      String response = server.arg("plain");
      mp3_volume = atoi(response.c_str());
       if(mp3_volume!=0)// atoi returns 0 if string is not a number
        // so if it is a number then it must be the volume
       {
          myMP3.volume(mp3_volume);
       }
       server.send(200, "text/plain", "successfully set the volume!\r\n");
    });

}


// An IR detector/demodulator is connected to GPIO pin 14 (D5 on a NodeMCU
// board).
// Note: GPIO 16 won't work on the ESP8266 as it does not have interrupts.
const uint16_t kRecvPin = 14;
unsigned long key_value = 0;


IRrecv irrecv(kRecvPin);

decode_results results;

void setup() {
  Serial.begin(115200);

  Serial1.begin(9600);
  myMP3.begin(Serial1, true);
  delay(1000);
  myMP3.volume(mp3_volume);

  pinMode(BUSY_PIN, INPUT);



    // Connect to Wi-Fi network with SSID and password
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid,password);
  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }
  // Print local IP address and start web server
  Serial.println("");
  Serial.println("WiFi connected.");


  IPAddress myIP = WiFi.localIP();
  Serial.println("IP address: ");
  Serial.println(myIP);


  if (!MDNS.begin("esp8266")) {             // Start the mDNS responder for esp8266.local
    Serial.println("Error setting up MDNS responder!");
  }
  Serial.println("mDNS responder started");

  server_handles();
  server.begin();
  debugln("HTTP server started");
  MDNS.addService("http", "tcp", 80);

  irrecv.enableIRIn();  // Start the receiver
  while (!Serial)  // Wait for the serial connection to be establised.
    delay(50);
  Serial.println();
  Serial.print("IRrecvDemo is now running and waiting for IR message on Pin ");
  Serial.println(kRecvPin);

}

void loop() {
  if (irrecv.decode(&results)) {
    // print() & println() can't handle printing long longs. (uint64_t)
    serialPrintUint64(results.value, HEX);
    Serial.println("");
    Serial.println(results.value);

    switch(results.value){
          case 0x40BD48B7:
          Serial.println("UP");
          myMP3.incVolume();
          break ;
          case 0x40BDC837:
          Serial.println("Down");
          myMP3.decVolume();
          break ;
          case 0x40BD8877:
          Serial.println("Left");
          myMP3.playPrevious();
          break ;
          case 0x40BD08F7:
          Serial.println("Right");
          myMP3.playNext();
          break ;
          case 0x40BD12ED:
          Serial.println("OK");
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BD00FF:
          Serial.println("1");
          mp3file_index = 1;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BD807F:
          Serial.println("2");
          mp3file_index = 2;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BD40BF:
          Serial.println("3");
          mp3file_index = 3;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BDC03F:
          Serial.println("4");
          mp3file_index = 4;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BD20DF:
          Serial.println("5");
          mp3file_index = 5;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BDA05F:
          Serial.println("6");
          mp3file_index = 6;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BD609F:
          Serial.println("7");
          mp3file_index = 7;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BDE01F:
          Serial.println("8");
          mp3file_index = 8;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BD10EF:
          Serial.println("9");
          mp3file_index = 9;
          myMP3.loop(mp3file_index);
          break ;
          case 0x40BDB04F:
          Serial.println("recyle");
          myMP3.startRepeat();
          break ;
          case 0x40BD728D:
          Serial.println("play");
          pause_mp3();
          break ;
        }
        key_value = results.value;

    irrecv.resume();  // Receive the next value
  }

  MDNS.update();
  server.handleClient();

  delay(10);
}

```

在串口调试模式下，按遥控器的各个按钮，记录下results.value的值，根据这些值来配置对应的代码。