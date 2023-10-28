# MQTT-for-ESP

# **1. Thiết bị**
- ESP32, ESP8266
- Một máy tính cài MQTT Broker

# **2. MQTT Broker**
MQTT là một giao thức truyền thông được ứng dụng nhiều trong các hệ thống IoT
- **_MQTT Broker _** là một hub để điều hướng (**_routing_**) các tin nhắn như các người gửi (**_Publisher_**) đến các người nhận (**_Subscriber_**). Bên gửi và bên nhận không cần có sự kết nối trực tiếp để có thể truyền nhận tin nhắn cho nhau. Một cá thể (**_MQTT Client_**) có thể vừa làm Publisher và Subscriber (vd. Server) hoặc chỉ làm Publisher/Subscriber (vd. sensors, ...).
- Các tin nhắn được phân biệt bởi tên của chúng (**_Topic_**) và do Publisher quy định; ngoài ra trên Broker cũng có sẵn các topic của hệ thống. 
- Broker có khả năng quản lý lượng lớn các cá thể truy cập và các tệp tin nhắn; quản lý bảo mật thông qua việc sử dụng tài khoản xác thực (**_Authentication_**) và mã hoá tin nhắn. 
- MQTT có thể được cài đặt trong server và các thiết bị IoT để lập một hệ thống truyền thông IoT. 
- Có thể sử dụng các MQTT Broker như: **_Eclipse Mosquitto_**, **_EMQX_**, ...
- Việc cài đặt trên HĐH Windows đơn giản, chỉ cần tải file .exe từ web và cài đặt [Eclipse Mosquitto](https://mosquitto.org/download/), [EMQX Broker](https://www.emqx.io/downloads); ngoài ra hướng dẫn cài đặt trên các HĐH khác như MacOS, Ubuntu/Linux,... đều được ghi lại trên web download.
**Lưu ý: Sau khi cài đặt MQTT, vào tưởng lửa để mở các port liên quan (1883, 9001, ...)**
***Lưu ý: địa chỉ của MQTT Broker chính là địa chỉ IPv4 của máy chủ cài đặt Broker**

# **3. Wi-Fi module (ESP, ...)**
Trong hướng dẫn này, ta sử dụng Wi-Fi module là NodeMCU ESP8266. Việc lập trình và cài đặt được thực hiện bằng Arduino IDE, viết bằng ngôn ngữ C/C++ phake

## Setup Arduino IDE để lập trình ESP8266
- 1. Cài đặt và mở Arduino IDE
- 2. Truy cập _Preferences_ qua _File -> Preferences_ hoặc cú pháp _Ctrl + ,_
- 3. Trong ô _Additional Boards Manager URLs_, nhập URL "http://arduino.esp8266.com/stable/package_esp8266com_index.json"
- 4. Truy cập phần _Boards Manager_ qua _Tools -> Board:... -> Boards Manager_
- 5. Đợi hệ thống index xong để sử dụng ô tìm kiếm, gõ _ESP8266_ vào ô tìm kiếm, cài đặt bản _esp8266 by ESP8266 Community_ dùng nút _Install_ (sau này có thể update package này qua Boards Manager theo cách tương tự)
- 6. Sau khi đã cài đặt xong, quay lại Arduino IDE để chọn board ESP8266 qua _Tools -> Board:... -> ESP8266 Boards_ và chọn _NodeMCU 1.0 (ESP-12E Module)_. 
**Lưu ý: có thể cần Restart app Arduino IDE sau bước nhập URL của board**

## Setup liên kết giữa ESP8266 và máy tính
Để connect ESP với máy tính, sử dụng dây USB-to-Micro USB
- 1. Kết nối ESP với máy tính qua cổng USB 
- 2. Vào phần _Device Manager_, kiểm tra kết nối trong mục _Ports (COM & LPT)_, sẽ hiện lên cổng COM đang dùng
- 3. Trong Arduino IDE, chọn COM qua _Tools -> Port_ và chọn COM đang dùng
**Lưu ý: nếu không kết nối được ESP với máy tính qua cổng USB**: cài đặt driver CH340 USB to COM, sau đó thử lại kết nối

## Chương trình kết nối Wi-Fi và MQTT cho ESP8266
- 1. Trước tiên, cài đặt các thư viện liên quan qua _Tools -> Manage Libraries_: PubSubClient (Nick O'Leary)
- 2. Viết chương trình cho ESP8266
```
// Thư viện
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>

// các biến authentication cho Wifi và MQTT Broker, 
const char* ssid = "yourNetworkName"; // thay bằng tên wifi
const char* password = "yourNetworkPassword"; // thay bằng password wifi
 
const char* mqttServer = "m11.cloudmqtt.com"; // thay bằng địa chỉ của MQTT Broker, vd. 172.20.10.2 (local)
const int mqttPort = 12948; // thay bằng port Broker (1883)
const char* mqttUser = "yourInstanceUsername"; // tài khoản authenticate của Broker nếu có
const char* mqttPassword = "yourInstancePassword";

// lập WiFi và MQTT Client
WiFiClient espClient;
PubSubClient client(mqtt_server, 1883, callback, espClient);

// Wi-Fi and MQTT connection 
void setupWifi() {
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.hostname();
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Attempt to connect
    if (client.connect(clientId.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      //client.publish();
      // ... and resubscribe
      //client.subscribe(SubTopic);
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

// get MQTT messages
void callback(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }

  Serial.print(message);
}

void setup()  {
  pinMode(BUILTIN_LED, OUTPUT);
  digitalWrite(BUILTIN_LED, LOW);
  
  Serial.begin(9600);

  setupWifi();  

  // client.subscribe();
}

// ###############################################################################

void loop (){
   
  if (!client.connected()) {
    reconnect();
  }

  client.loop();

  if (WiFi.status() != WL_CONNECTED) {
    setupWifi();
  }
  
}
```

## Để gửi tệp tin JSON qua MQTT
