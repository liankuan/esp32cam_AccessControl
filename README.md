# esp32cam_AccessControl

CameraWebServerOpendoor1004.ino

boolean  sendmsg=false;

line 189:
matched_id = recognize_face(&id_list, aligned_face);
if (matched_id >= 0) {
    Serial.printf("Match Face ID: %u\n", matched_id);
    rgb_printf(image_matrix, FACE_COLOR_GREEN, "Hello Subject %u", matched_id);
    pinMode(2,OUTPUT);
    Serial.println("發現白名單,開門");
    digitalWrite(2,HIGH);
    delay(5000);
    Serial.println("關門");
    digitalWrite(2,LOW);
} else {
    Serial.println("No Match Found");
    rgb_print(image_matrix, FACE_COLOR_RED, "Intruder Alert!");
    matched_id = -1;
    sendmsg=true;
    delay(5000);
}


line 781:

String myLineNotifyToken = "line token";
String sendImage2LineNotify(String msg) {
  camera_fb_t * fb = NULL;
  fb = esp_camera_fb_get();//取得相機影像放置fb
  if (!fb) {
    delay(100);
    Serial.println("Camera capture failed, Reset");
    ESP.restart();
  }
  WiFiClientSecure client_tcp;//啟動SSL wificlient
  Serial.println("Connect to notify-api.line.me");
  if (client_tcp.connect("notify-api.line.me", 443)) {
    Serial.println("Connection successful");
    String head = "--Taiwan\r\nContent-Disposition: form-data; name=\"message\"; \r\n\r\n" + msg + "\r\n--Taiwan\r\nContent-Disposition: form-data; name=\"imageFile\"; filename=\"esp32-cam.jpg\"\r\nContent-Type: image/jpeg\r\n\r\n";
    String tail = "\r\n--Taiwan--\r\n";
    uint16_t imageLen = fb->len;
    uint16_t extraLen = head.length() + tail.length();
    uint16_t totalLen = imageLen + extraLen;
    //開始POST傳送訊息
    client_tcp.println("POST /api/notify HTTP/1.1");
    client_tcp.println("Connection: close");
    client_tcp.println("Host: notify-api.line.me");
    client_tcp.println("Authorization: Bearer " + myLineNotifyToken);
    client_tcp.println("Content-Length: " + String(totalLen));
    client_tcp.println("Content-Type: multipart/form-data; boundary=Taiwan");
    client_tcp.println();
    client_tcp.print(head);
    uint8_t *fbBuf = fb->buf;
    size_t fbLen = fb->len;
    Serial.println("Data Sending....");
    //照片，分段傳送
    for (size_t n = 0; n < fbLen; n = n + 2048) {
      if (n + 2048 < fbLen) {
        client_tcp.write(fbBuf, 2048);
        fbBuf += 2048;
      } else if (fbLen % 2048 > 0) {
        size_t remainder = fbLen % 2048;
        client_tcp.write(fbBuf, remainder);
      }
    }
    client_tcp.print(tail);
    client_tcp.println();
    String getResponse = "", Feedback = "";
    boolean state = false;
    int waitTime = 3000;   // 依據網路調整等候時間，3000代表，最多等3秒
    long startTime = millis();
    delay(1000);
    Serial.print("Get Response");
    while ((startTime + waitTime) > millis())    {
      Serial.print(".");
      delay(100);
      bool jobdone=false;
      while (client_tcp.available())
      {//當有收到回覆資料時
        jobdone=true;
        char c = client_tcp.read();
        if (c == '\n')
        {
          if (getResponse.length() == 0) state = true;
          getResponse = "";
        }
        else if (c != '\r')
          getResponse += String(c);
        if (state == true) Feedback += String(c);
        startTime = millis();
      }
      if (jobdone) break;
    }
    client_tcp.stop();
    esp_camera_fb_return(fb);//清除緩衝區
    return Feedback;
  }
  else {
    esp_camera_fb_return(fb);
    return "Send failed.";
  }
}



void loop() {
  // put your main code here, to run repeatedly:
    if(sendmsg){
    Serial.println("starting to Line");
    String payload = sendImage2LineNotify("有陌生人靠近....");
    Serial.println(payload);
    sendmsg=false; 
    }
    delay(20000);
}
