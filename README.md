# test_cochera
Apertura y cierre del porton de la cochera

#include <ESP8266WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>

#define cerraduraPin    D1
#define abrirPortonPin  D2
#define cerrarPortonPin D3
#define abrirTopePin    D5
#define cerrarTopePin   D6

#define BOTtoken "tu_token_id"  
#define chatID "tu_chat_id" // poner su id_chat

volatile int estadoCerradura = LOW;
volatile int estadoAbrir = LOW;
volatile int estadoCerrar = LOW;

char ssid[] = "tu_wifi";     
char password[] = "tu_contraseña"; 

WiFiClientSecure client;
UniversalTelegramBot bot(BOTtoken, client);

int Bot_mtbs = 500; //mean time between scan messages
long Bot_lasttime;   //last time messages' scan has been done

void abrirPorton(){
  estadoAbrir = HIGH ;
  digitalWrite(cerraduraPin, LOW) ;  
  digitalWrite(abrirPortonPin, LOW) ;  
  bot.sendMessage(chatID, "|> Se esta abriendo el porton.", "");
  delay(1500) ; // Pausa de 1 segundo y medio
  digitalWrite(cerraduraPin, HIGH) ;  
}

void cerrarPorton() {
  estadoCerrar = HIGH ;
  digitalWrite(cerrarPortonPin, LOW) ;
  bot.sendMessage(chatID, "|> Se esta cerrando el porton.", "");
}

void abrirTopePorton() {
  if (estadoAbrir == HIGH) {
    estadoAbrir = LOW ;
    digitalWrite(abrirPortonPin, HIGH);
    bot.sendMessage(chatID, "|> Se abrio el porton.", "");
  }
}

void cerrarTopePorton() {
  if (estadoCerrar == HIGH) {
    estadoCerrar = LOW ;
    digitalWrite(cerrarPortonPin, HIGH) ;
    bot.sendMessage(chatID, "|> Se cerro el porton.", "");
  }
}

void estadoPorton(String chat_id) {
  int abierto = digitalRead(abrirTopePin); 
  int cerrado = digitalRead(cerrarTopePin);
  Serial.println(abierto);
  Serial.println(cerrado);

  if (abierto == HIGH) {
      bot.sendMessage(chat_id, "Porton abierto", "");
  } else if (cerrado == HIGH) {
      bot.sendMessage(chat_id, "Porton cerrado", "");
  } else {
      bot.sendMessage(chat_id, "Estado desconocido... cerrando", "");
      cerrarPorton();
  }
}

void handleNewMessages(int numNewMessages) {
//  Serial.println("handleNewMessages");
//  Serial.println(String(numNewMessages));

  for (int i=0; i<numNewMessages; i++) {
    String chat_id = String(bot.messages[i].chat_id);
    String text = bot.messages[i].text;

    String from_name = bot.messages[i].from_name;
    if (from_name == "") from_name = "Anonimo";

    if (text == "/abrir") {
      sendKeyboard(chat_id);
      abrirPorton();
      bot.sendMessage(chat_id, "Abriendo porton", "");
      bot.sendMessage(chatID, "|> " + from_name + " solicito abrir el porton", "");
    }

    if (text == "/cerrar") {
      sendKeyboard(chat_id);
      cerrarPorton() ;
      bot.sendMessage(chat_id, "Cerrando porton", "");
      bot.sendMessage(chatID, "|> " + from_name + " solicito cerrar el porton", "");
    }

    if (text == "/estado") {
      estadoPorton(chat_id) ;
    }

    if (text == "/start") {
      String welcome = "Bienvenido, " + from_name + ".\n";
      bot.sendMessage(chat_id, welcome, "");
      sendKeyboard(chat_id) ;
    }
  }
}

void sendKeyboard(String chat_id) {
  /*
   * [
   *  [{"text":  "Abrir porton",
   *    "callback_data": "/abrir"}],
   *  [{"text": "Cerrar porton",
   *    "callback_data": "/cerrar"}],
   *  [{"text": "Estado porton",
   *    "callback_data": "/estado"}]
   * ]
   */
      String keyboardJson = "[[{ \"text\" : \"Abrir porton\", \"callback_data\" : \"/abrir\" }],[{ \"text\" : \"Cerrar porton\", \"callback_data\" : \"/cerrar\" }],[{ \"text\" : \"Estado porton\", \"callback_data\" : \"/estado\" }]]";
      bot.sendMessageWithInlineKeyboard(chat_id, "Que desea hacer?", "", keyboardJson);
}

void setup() {
  Serial.begin(115200);

  pinMode(cerraduraPin,    OUTPUT);
  pinMode(abrirPortonPin,  OUTPUT);
  pinMode(cerrarPortonPin, OUTPUT);
  pinMode(abrirTopePin,    INPUT_PULLUP);
  pinMode(cerrarTopePin,   INPUT_PULLUP);
  
  digitalWrite(cerraduraPin, !estadoAbrir);
  digitalWrite(abrirPortonPin, !estadoAbrir);
  digitalWrite(cerrarPortonPin, !estadoCerrar) ;
        
  attachInterrupt(abrirTopePin, abrirTopePorton, CHANGE); // CHANGE, RISING, FALLING 
  attachInterrupt(cerrarTopePin, cerrarTopePorton, CHANGE);

  // Set WiFi to station mode and disconnect from an AP if it was Previously
  // connected
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  delay(100);

  // Attempt to connect to Wifi network:
  Serial.print("Connecting Wifi: ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
  
  bot.sendMessage(chatID, "|> iniciando...|", "");
}

void loop() {
  if (millis() > Bot_lasttime + Bot_mtbs)  {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);

    while (numNewMessages) {
      Serial.println("got response");
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }

    Bot_lasttime = millis();
  }
}
