#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

// Wi-Fi details
const char* ssid = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";

// ESP32 pins
#define MQ_PIN 34
#define DHT_PIN 4
#define DHT_TYPE DHT22

// LCD configuration
#define LCD_ADDRESS 0x27
#define LCD_COLUMNS 16
#define LCD_ROWS 2

// Raw warning threshold (not official AQI)
const int RAW_WARNING_THRESHOLD = 2000;

// Objects
DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(
  LCD_ADDRESS, LCD_COLUMNS, LCD_ROWS
);
WebServer server(80);

// Sensor values
int mqValue = 0;
float temperature = NAN;
float humidity = NAN;

unsigned long lastSensorRead = 0;

// Web dashboard
const char dashboard[] PROGMEM = R"HTML(
<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Air Quality Monitor</title>

<style>
body {
  margin:0;
  padding:18px;
  font-family:Arial;
  background:#edf2f7;
  color:#1e293b;
}
h2 {text-align:center;}
.grid {
  display:grid;
  grid-template-columns:repeat(auto-fit,minmax(145px,1fr));
  gap:12px;
  max-width:760px;
  margin:auto;
}
.card {
  background:white;
  padding:18px;
  border-radius:12px;
  text-align:center;
  box-shadow:0 2px 8px #0001;
}
.value {
  font-size:27px;
  font-weight:bold;
  margin:10px 0;
}
.small {
  font-size:12px;
  color:#64748b;
}
canvas {
  width:100%;
  height:220px;
}
</style>
</head>

<body>

<h2>Real-Time Air Quality Monitor</h2>
<p style="text-align:center">
ESP32 + MQ-135 + DHT22
</p>

<div class="grid">

<div class="card">
MQ-135 Raw Reading
<div class="value" id="mq">--</div>
<div class="small">ADC: 0-4095</div>
</div>

<div class="card">
Temperature
<div class="value" id="temp">--</div>
</div>

<div class="card">
Humidity
<div class="value" id="hum">--</div>
</div>

<div class="card">
Sensor Status
<div class="value" id="status">Starting</div>
</div>

</div>

<div class="card" style="max-width:724px;margin:14px auto">
<h3>Live MQ-135 Readings</h3>
<canvas id="chart" width="700" height="220"></canvas>
<div class="small">Latest 30 samples</div>
</div>

<p class="small" style="text-align:center">
MQ-135 readings are uncalibrated and are not official AQI values.
</p>

<script>

const samples = [];
const canvas = document.getElementById('chart');
const ctx = canvas.getContext('2d');

function drawChart() {
  const w = canvas.width;
  const h = canvas.height;

  ctx.clearRect(0,0,w,h);

  const maxValue = Math.max(4095,...samples);

  ctx.strokeStyle = '#d5dde7';
  ctx.fillStyle = '#475569';

  for(let i=0;i<=4;i++) {
    const y = 10+i*(h-35)/4;

    ctx.beginPath();
    ctx.moveTo(42,y);
    ctx.lineTo(w-8,y);
    ctx.stroke();

    ctx.fillText(
      Math.round(maxValue*(4-i)/4),2,y+4
    );
  }

  if(samples.length<2) return;

  ctx.strokeStyle='#2563eb';
  ctx.lineWidth=2;
  ctx.beginPath();

  samples.forEach((v,i)=>{
    const x=42+i*(w-55)/(samples.length-1);
    const y=10+(h-35)*(1-v/maxValue);

    if(i===0) ctx.moveTo(x,y);
    else ctx.lineTo(x,y);
  });

  ctx.stroke();
}

async function updateData() {

  try {
    const response = await fetch('/data');
    const d = await response.json();

    document.getElementById('mq').textContent=d.mq;

    document.getElementById('temp').textContent=
      d.temp===null?'--':d.temp+' °C';

    document.getElementById('hum').textContent=
      d.hum===null?'--':d.hum+' %';

    const status=document.getElementById('status');

    status.textContent=d.warning?'Check':'Normal';

    status.style.color=
      d.warning?'#dc2626':'#15803d';

    samples.push(d.mq);

    if(samples.length>30) samples.shift();

    drawChart();

  } catch(error) {

    document.getElementById('status').textContent=
      'Connection lost';
  }
}

updateData();
setInterval(updateData,2000);

</script>
</body>
</html>
)HTML";

// Read sensors
void readSensors() {

  mqValue = analogRead(MQ_PIN);

  float t = dht.readTemperature();
  float h = dht.readHumidity();

  if(!isnan(t)) temperature=t;
  if(!isnan(h)) humidity=h;
}

// Update LCD
void updateLCD() {

  lcd.clear();

  // First row
  lcd.setCursor(0,0);
  lcd.print("MQ:");
  lcd.print(mqValue);

  lcd.setCursor(10,0);
  lcd.print("AIR");

  // Second row
  lcd.setCursor(0,1);

  if(isnan(temperature)) {
    lcd.print("T:--");
  } else {
    lcd.print("T:");
    lcd.print(temperature,1);
  }

  lcd.print("C ");

  if(isnan(humidity)) {
    lcd.print("H:--");
  } else {
    lcd.print("H:");
    lcd.print(humidity,0);
  }

  lcd.print("%");
}

// Webpage
void handleRoot() {
  server.send_P(200,"text/html",dashboard);
}

// JSON data
void handleData() {

  String tempJSON = isnan(temperature)
    ? "null" : String(temperature,1);

  String humJSON = isnan(humidity)
    ? "null" : String(humidity,1);

  bool warning =
    mqValue >= RAW_WARNING_THRESHOLD;

  String json = "{";

  json += "\"mq\":" + String(mqValue) + ",";
  json += "\"temp\":" + tempJSON + ",";
  json += "\"hum\":" + humJSON + ",";
  json += "\"warning\":";
  json += warning ? "true" : "false";

  json += "}";

  server.send(200,"application/json",json);
}

void setup() {

  Serial.begin(115200);

  analogReadResolution(12);
  analogSetPinAttenuation(MQ_PIN,ADC_11db);

  // I2C pins
  Wire.begin(21,22);

  // Start sensors
  dht.begin();

  // Start LCD
  lcd.init();
  lcd.backlight();

  lcd.setCursor(0,0);
  lcd.print("Air Quality");
  lcd.setCursor(0,1);
  lcd.print("Starting...");

  delay(2000);

  // Wi-Fi connection
  WiFi.begin(ssid,password);

  lcd.clear();
  lcd.print("Connecting WiFi");

  while(WiFi.status()!=WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Dashboard IP: ");
  Serial.println(WiFi.localIP());

  // Web server
  server.on("/",HTTP_GET,handleRoot);
  server.on("/data",HTTP_GET,handleData);

  server.begin();

  readSensors();
  updateLCD();
}

void loop() {

  server.handleClient();

  if(millis()-lastSensorRead>=2000) {

    lastSensorRead=millis();

    readSensors();
    updateLCD();
  }
}
