# Protokoll 8

**Übungsdatum:** 4.6.2019  

**Name:** Mercedes Wesonig  

**KNr:** 16  

**Gruppe:** 3  


# Aufgabenstellung

Aufgrund von zwei Prüfungen hatten wir leider zu wenig Zeit, das Programm, welches die Temperatur am µC ausgeben soll, fertig zu stellen. Deshalb besprachen wir ein bereits vorhandenes Programm der anderen Gruppe genau durch und versuchten jeden Schritt genau zu verstehen. 

## Temperatursensor im Arduino Nano

Bei diesem Programm wird die Temperaturmesseinheit des Atmega 329p, welche sich direkt am Board befindet, verwendet. Sie wird jedoch nicht oft verwendet, da sie sehr ungenau ist. Wenn der Chip nicht schläft (zb LED ansteuern), steigt die Innentemperatur an. Die Elektronik hält maximal 85° C aus, also sollte ein Programm entwickelt werden, welches den Chip bei 80° C ausschaltet. Hier wäre es von Vorteil, wenn der Sensor kalibriert ist, besonders bei genaue Messungen.  
Die Spannungen die diese Einheit bei unterschiedlichen Messwerten liefert liegt bei 100mV (bei - 45°C) bis zu 500mV (80°C). 

Das Programm für den Arduino Nano wird in der *app.c* und in der *app.h* ausprogrammiert.

## app.c


``` c
1 #include "global.h"
2
3 #include <stdio.h>
4 #include <string.h>
5
6 #include <avr/io.h>
7 #include <avr/interrupt.h>
8 #include <util/delay.h>
9
10 #include "app.h"
11 #include "sys.h"
12
```
Im ersten Teil des Programmes befinden sich wie immer die "Include-Befehle" um die Bibliotheken einzubinden, welche im Programm benötigt werden.

```c
13 // defines
14 ...
15
16
17 // declarations and definations
18
19 volatile struct App app;
20
21
```
Anschließend folgen die Defines und Deklarationen.

``` c
22 // functions
23
24 void app_init (void) {
25 memset((void *)&app, 0, sizeof(app));
26
27 ADMUX = 8; // Multiplexer ADC8 = Temp.
28 ADMUX |= (1 << REFS1) | (1 << REFS0); // VREF=1.1V
29 ADMUX |= (1 << ADLAR); // Left Adj, -> Result in ADCH
30
31 ADCSRA = (1 << ADEN) | 7; // fADC=125kHz
32 ADCSRB = 0;
33
34 }
```
Nun beginnen wir mit den Funktionen. Hier wir der **interne ADC konfiguriert**. 

**Zeile 27** : Hier wird der Multiplexer ADC auf 8 gesetzt. Dort befindet sich nämlich der Temperatursensor.   

**Zeile 28** :  Es wird einen Referenzspannung von 1,1V festgelegt.
Der ADC arbeitet nach dem Prinzip der succsessive Approximation. Das heißt er Vergleich die Eingangsspannung mit internen Spannungsstufen. Der kleinste Wert entspricht GND und der größte die hier eingestellte Referenzspannung. Es können auch andere Spannungen eingestellt werden (zb VCC), wir haben uns jedoch für die Referenzspannung des µCs entschieden, da diese sehr genau ist. Der Wert selber ist absolut gesehen zwar nicht sehr genau aber dafür gibt es fast keine Schwankungen. Die Versorgung würde zu sehr schwanken. Da der Sensor jedoch nur Werte zwischen 100 und 500mV hat, wird dieser Bereich alle Mal mit den 1,1V abgedeckt.  

**Zeile 29**: Das 10-Bit  Ergebnis wir dann im  ADC Data Register ADCH abgespeichert. Der Befehl *ADMUX |= (1 << ADLAR);* bewirkt, dass das Ergebnis linksbündig ausgegeben wird.   

**Zeile 31**: Hier wir der Prescaler auf 128 gesetzt (7 bedeutet hier 128). Dadurch wird eine Frequenz von 125 kHz erreicht, was uns sehr gut passt, da der ADC mit einen geringen Frequenz arbeiten muss/soll.  

**Zeile 32**:  Wird zur Sicherheit deaktiviert.  


```c
35
36
37 //--------------------------------------------------------
38
39 uint16_t app_inc16BitCount (uint16_t cnt) {
40 // sys_setLed(1);
41 return cnt == 0xffff ? cnt : cnt + 1;
42 }
43
```
Counter um eine Zahl hoch zu zählen. Wenn der Counter 0xffff soll er den Counter zurück geben und sonst Counter + 1.
``` c
44 uint8_t hex2int (char h) {
45 if (h >= '0' && h <= '9') {
46 return h - '0';
47 }
48 if (h >= 'A' && h <= 'F') {
49 return h - 'A' + 10;
50 }
51 return 0xff; // Fehler
52 }
53
```

Hier wird kontrolliert, ob das Zeichen *h* ein Hexwert ist. Also ob nur die Ziffern 0 bis 9 beziehungsweise die Buchstaben A bis F vorkommen. 

``` c
54 uint8_t app_handleModbusRequest () {
55 // printf("Request eingetroffen: %s", app.modbusBuffer);
56
57 // Check if valid modbus frame
58 char *b = app.modbusBuffer;
59 uint8_t size = app.bufIndex;
60
61 if (size < 9) { return 1; }
62 if (b[0] != ':') { return 2; }
63 if (b[size - 1] != '\n') { return 3; }
64 if (b[size - 2] != '\r') { return 4; }
65 if ( (size - 3) % 2 != 0) { return 5; }
66 for (uint8_t i = 1; i < (size - 2); i++) {
67 char c = b[i];
68 if (! ((c >= '0' && c <= '9') ||
69 (c >= 'A' && c <= 'F')))
70 return 6;
71 }
``` 
**Zeile 58 und 59**: Hier werden die Variablen aus der Struktur in der app.h verwendet.  

**Zeile 61 - 70**: Hier wird überprüft ob der Frame in Ordnung ist. Also ob beim Über tagen ein Fehler passiert ist. Dies geschieht, indem geschaut wird, ob alle Zeichen wie Startzeichen, Stopzeichen, Größe (min. 9 Bytes)  und Art der Zeichen (Hex oder nicht Hex) den Vorschriften des Protokolls entsprechen. Returns helfen hierbei bei der Fehlersuche.  

```c
72
73 uint8_t lrc = 0;
74 for (uint8_t i = 1; i < (size - 4); i++) {
75 lrc += b[i];
76 }
77 lrc = (uint8_t)( -(int8_t)lrc);
78 char s[3];
79 snprintf(s, sizeof s, "%02X", lrc);
80 if (b[size - 4] != s[0]) { return 7; }
81 if (b[size - 3] != s[1]) { return 7; }
82
83 // printf("Request richtig\n");
``` 
Hier wird der LRC überprüft.  

**Zeile 73 - 79**: Bildung des LRC. Dabei werden alle Werte vom Start bis zur Position des LRCs (size - 4) in einer for-Schleife zusammen gezählt. Anschließend wird das Zweierkompliment dieser Zahl gebildet und in ein Feld gespeichert.  

**Zeile 73 - 79**: Dann wird dieser Wert mit dem LRC im Frame verglichen.  

``` c
84 uint8_t i, j;
85 for (i = 1, j = 0; i < (size - 4); i += 2, j++ ) {
86 uint8_t hn = hex2int(b[i]);
87 uint8_t ln = hex2int(b[i+1]);
88 if (hn == 0xff || ln == 0xff) {
89 return 8;
90 }
91 uint8_t value = hn * 16 + ln;
92 b[j] = value;
93 }
94 size = j;
95
``` 
Wenn hn oder ln 0xff wären, würde das heißen, dass ein Fehler aufgetreten ist (siehe Zeile 51). Es könnte aber auch sein, dass eine ISR hier den Speicher überschrieben hat. Return 8 hilft bei der Fehlersuche.
``` c
96 uint8_t deviceAddress = b[0];
97 if (deviceAddress != 1) {
98 return 0;
99 }
100
```
Wenn die Adresse ungleich 1 ist, wird 0 zurück gegeben.
```c
101 uint8_t funcCode = b[1];
102 switch (funcCode) {
103 case 0x04: {
104 uint16_t startAddress = b[2] << 8 | b[3];
105 uint16_t quantity = b[4] << 8 | b[5];
106 if (quantity < 1 || quantity > 0x7d) {
107 b[1] = 0x80 | b[1]; // error
108 b[2] = 0x03; // quantity out of range
109 size = 3;
110
111 } else if (startAddress != 1 || quantity != 1) {
112 b[1] = 0x80 | b[1]; // error
113 b[2] = 0x02; // wrong start address
114 size = 3;
115
116 } else {
117 b[2] = 2;
118 b[3] = app.mbInputReg01 >> 8;
119 b[4] = app.mbInputReg01 & 0xff;
120 size = 5;
121 }
122 break;
123 }
124
125 default: {
126 b[1] = 0x80 | b[1]; // error
127 b[2] = 0x01; // function code not supported
128 size = 3;
129 }
130 }
131
```
Reaktion auf den Funktionscode. Im Switch Case wird nur der Fall 04 behandelt, da wir das Input Register (16 Bit) lesen wollen. Wir wollen auch nur diesen Funktionscode verwenden, wenn ein anderer im Frame steht, wird dieser im  default-Zweig behandelt. 
Es wird eine Antwort generiert, mit den ganzen Fehlerüberprüfungen und das wird dann in den einzelnen char gespeichert. Es wird also alles einzeln zusammen gesetzt. 
```c
132 lrc = 0;
133 printf(":");
134 for (i = 0; i < size; i++) {
135 printf("%02X", (uint8_t)b[i]);
136 lrc += b[i];
137 }
138 lrc = (uint8_t)(-(int8_t)lrc);
139 printf("%02X", lrc);
140 printf("\r\n");
141 return 0;
142 }
143
```
Ausgabe des Frames.
```c
144 void app_handleUartByte (char c) {
145 if (c == ':') {
146 if (app.bufIndex > 0) {
147 app.errCnt = app_inc16BitCount(app.errCnt);
148 }
149 app.modbusBuffer[0] = c;
150 app.bufIndex = 1;
151
152 } else if (app.bufIndex == 0) {
153 app.errCnt = app_inc16BitCount(app.errCnt);
154
155 } else if (app.bufIndex >= (sizeof(app.modbusBuffer))) {
156 app.errCnt = app_inc16BitCount(app.errCnt);
157
158 } else {
159 app.modbusBuffer[app.bufIndex++] = c;
160 if (c == '\n') {
161 uint8_t errCode = app_handleModbusRequest();
162 if (errCode > 0) {
163 // printf("Fehler %u\n\r", errCode);
164 app.errCnt = app_inc16BitCount(app.errCnt);
165 }
166 app.bufIndex = 0;
167 }
168 }
169 }
170
```
Modbus - Frame abspeicher und nochmaliges suchen nach Fehlerquellen.
```c
171 void app_main (void) {
172
173 ADCSRA |= (1 << ADSC);
174 // _delay_ms(1);
175 // printf("ADCH=%u ", ADCH);
176
177 // Gerade: ModbusRegister = k * ADCH + d
178 // aus Datenblatt:
179 // -45°C -> 242mV -> ADCH=56.79 -> MBR = -45*256 = -11520
180 // 25°C -> 314mV -> ADCH=73.08 -> MBR = 25*256 = 6400
181 // 85°C -> 380mV -> ADCH=88.40 -> MBR = 85*256 = 21760
182 // daraus ergibt sich für:
183 // <=25°C -> MBR = k1 * ADCH + d1
184 // >25°C -> MBR = k2 * ADCH + d2
185 float k1 = 1100.061, d1 = -73992.464;
186 float k2 = 1002.611, d2 = -66870.812;
187
188 // reale Messung bei 22°C -> ADCH=88
189 // interpoliert:
190 // 22°C -> ? -> ADCH=72.38 -> MBR = 22*256 = 5632
191 // Offsetkorrektur um bei ADCH=88 auf 5632 zu kommen
192 // ADCH<=88 -> MBR = k1 * ADCH + d1 + o1
193 // ADCH>88 -> MBR = k2 * ADCH + d2 + o2
194 float o1 = -17180.9;
195 float o2 = -15726.956;
196
197 float mbr;
198 uint8_t adch = ADCH;
199 // adch = 88;
200 if (adch <= 88) {
201 mbr = k1 * adch + d1 + o1;
202 } else {
203 mbr = k2 * adch + d2 + o2;
204 }
205 int16_t mbInputReg01 = (int16_t)mbr;
206 int8_t vk = mbInputReg01 / 256;
207 uint8_t nk = ((mbInputReg01 & 0xff) * 100) / 256;
208
209 app.mbInputReg01 = (uint16_t)mbInputReg01;
210
211 // printf("T=%.1f = %d.%02d MBR = %d = 0x%04x \r", mbr/256.0, vk, (int)nk, mbInputReg01, (uint16_t)mbInputReg01);
212
213 int c = fgetc(stdin);
214 if (c != EOF) {
215 // printf("\r\n %02x\r\n", (uint8_t)c);
216 app_handleUartByte((char) c);
217 }
218
219 }
220
221 //--------------------------------------------------------
222
```
Umrechnung des Wertes den der ADC liefert  zum "richtigen" Temperaturwert. Dabei prüfen wir, ob der Wert sich in einem zulässigen Bereich befindet, ansonsten wird der Maximalwert ausgegeben damit der Benutzer erkennt, dass es sich hier um einen Fehler handeln muss. sollte der Wert in einem Gültigem Bereich liegen, wird dieser gespeichert und anschließend zur Kontrolle ausgegeben.
```c
223 void app_task_1ms (void) {
224 static uint16_t oldErrCnt = 0;
225 static uint16_t timer = 0;
226 if (app.errCnt != oldErrCnt) {
227 oldErrCnt = app.errCnt;
228 timer = 2000;
229 sys_setLed(1);
230 }
231 if (timer > 0) {
232 timer--;
233 if (timer == 0) {
234 sys_setLed(0);
235 }
236 }
237 }
238
239
```
LED abhängig vom Counter blinken lassen.
```c
240 void app_task_2ms (void) {}
241 void app_task_4ms (void) {}
242 void app_task_8ms (void) {}
243 void app_task_16ms (void) {}
244 void app_task_32ms (void) {}
245 void app_task_64ms (void) {}
246 void app_task_128ms (void) {}
```
Bleiben leer.

## app.h

```c
1 #ifndef APP_H_INCLUDED
2 #define APP_H_INCLUDED
3
```
Im ersten Teil des Programmes befinden sich wie immer die "Include-Befehle" um die Bibliotheken einzubinden, welche im Programm benötigt werden.
```c
4 // declarations
5
6 struct App
7 {
8 uint8_t flags_u8;
9 char modbusBuffer[32];
10 uint8_t bufIndex;
11 uint16_t errCnt;
12 uint16_t mbInputReg01;
13 };
14
15 extern volatile struct App app;
16
17
```
Anschließend folgen die Deklarationen. In der Struktur werden einige wichtige Variablen für das Programm festgelegt. 

```c
18 // defines
19
20 #define APP_EVENT_0 0x01
21 #define APP_EVENT_1 0x02
22 #define APP_EVENT_2 0x04
23 #define APP_EVENT_3 0x08
24 #define APP_EVENT_4 0x10
25 #define APP_EVENT_5 0x20
26 #define APP_EVENT_6 0x40
27 #define APP_EVENT_7 0x80
28
29
```
Defines dürfen natürlich auch nicht fehlen. 

```c
30 // functions
31
32 void app_init (void);
33 void app_main (void);
34
35 void app_task_1ms (void);
36 void app_task_2ms (void);
37 void app_task_4ms (void);
38 void app_task_8ms (void);
39 void app_task_16ms (void);
40 void app_task_32ms (void);
41 void app_task_64ms (void);
42 void app_task_128ms (void);
43
44 #endif // APP_H_INCLUDED
```
Bleiben leer.

# Theorie

**ACHTUNG!** Einige Teilgebiete haben wir nicht in der letzen Stunde besprochen, da ich diese jedoch für meine Prüfung gebraucht habe und auch in der folgenden Einheit brauchen werde, hab ich mich dazu entschlossen, diese als Übung hinzuzufügen.

## Begriffsunterscheidung

In der Informatik und Programmierung ist eine **Deklaration** die Festlegung von Dimension, Bezeichner, Datentyp und weiteren Aspekten einer Variable oder eines Unterprogramms.  


**Initialisierung** bezeichnet in der Programmierung die Zuweisung eines Initial- oder Anfangswertes zu einem  Objekt oder einer Variablen.  


Und beim **Definieren** wird ein Speicher beansprucht.  


## Wiederholung Modbus

### Was ist ein Modbus?

Ein Modbus ist ein Kommunikationsprotokoll. Es dient zum Austausch von Informationen zwischen Client und Server. Es basiert auf dem Request/Response Prinzip. Dabei sendet der Master (Client, zb ein PC) eine Anfrage (Request) an den Slave (Server, zb ein Sensor). 

### Welche Formen gibt es?

Die Datenübertragung wird in drei Varianten unterschieden:
* Modbus ASCII
	* Rein textuelle Übertragung (byteweise)
* Modbus RTU
	* binäre Übertragung (byteweise)
* Modbus TCP/IP
	* Übertragung erfolgt in TCP Parketen

ASCII und RTU sind sich sehr ähnlich, da beide byteweise übertragen.

### Funktionsprinzip

Der Master sendet einen Frame an den Slave mit allen nötigen Informationen.

#### ASCII
Zu Beginn wird immer ein Startbit gesendet, der Doppelpunkt (1Byte groß). Anschließend befindet sich die Adresse des Empfängers und der Funktionscode. Dieser sagt aus, was zu tun ist (Lesen, schreiben). Die beiden sind jeweils 2 Byte groß. Nun werden die eigentlichen Daten übertragen, welche in der Länge variieren können (max 2 mal 252 Bytes). Danach folgt der LRC, welcher die Prüfsumme enthält (ebenfalls 2 Byte). Zum Schluss befindet sich das Stopzeichen "\r\n" .



#### RTU 

Beim RTU werden im Gegensatz zum ASCII  binär Codes verschickt. Jeder RTU Code fängt mit einer mindestens 3,5 Zeichen langen Pause an. Danach folgt wieder die Adresse des Empfängers (1 Byte) und der Funktionscode (auch 1 Byte) . Nach den eigentlichen Daten kommt ein 2 Byte großer CRC, mit welcher überprüft wird, ob ein Fehler aufgetreten ist. Am Ende Folgt wieder eine mindestens 3,5 Zeichen lange Pause.

#### TCP

Der TCP enthält keine Kontrollbytes damit die Handhabung von TCP Treibern einfacher ist. Dieser Modbus ist hauptsächlich für das Ethernet gedacht. Der Start besteht aus einer Transaktionsnummer, welcher 2 Byte groß ist. Darauf folgt ein Protokollzeichen (immer gleicher Aufbau, 0x0000). Danach findet man die Zahl, der noch folgenden Bytes sowie die Adresse und die Funktion. 

![enter image description here](http://www.simplymodbus.ca/images/adu_pdu.PNG)

### Was ist ein Modbus Gateway?

Ein Modbus Gateway ist in der Lage verschieden Modbus-Varianten miteinander zu verbinden. (zb Verbindung einer UART Schnittstelle mit einem Sensor, welcher mit TCP/IP erreichbar ist.

### Welche Modbus Daten-Modelle gibt es?

* Discrete Inputs
	* Ein einzelnes Bit lesen
	* zb Taster
* Coils
	* Ein einzelnes Bit lesen und schreiben
	* zb LED
* Input Register
	* 16 Bit Wert lesen
	* zb Sensor
* Hold-Register
	* 16 Bit Wert lesen und schreiben
	* zb PWM Signal






