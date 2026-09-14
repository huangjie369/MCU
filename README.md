这是一个。。。。。，主要由MCU最小系统、电源管理、风扇驱动及状态检测等模块组成。

首先是MCU最小系统，以STM32F103为核心控制器，负责风扇电压、电流等参数采集，检测风扇连接状态，并通过PWM信号实现风扇转速控制。



其次是晶振电路（8MHz），为MCU提供稳定的时钟源，保证系统可靠运行。

复位电路用于系统上电初始化和异常状态恢复，确保MCU能够稳定启动和运行。

电源管理电路负责将输入的12V电源转换为系统所需的低压电源。其中采用LM5161PWPR和XC6206P332，分别完成12V转5V及5V转3.3V，为MCU及外围器件提供稳定供电。

风扇驱动电路用于将MCU输出的PWM控制信号转换为可驱动风扇的大功率驱动信号，其核心通过MOSFET实现控制信号与功率输出的转换。

最后是电压监测与通信模块，用于实时监测12V、5V及3.3V电源状态。当电压异常时输出红色告警指示，电压正常时输出绿色状态指示。同时将采集的电压、电流数据以及风扇控制指令通过串口上传至上位机，实现数据监控与状态显示。


/*
 * STM32F103C8T6 风扇电源监控下位机（PA8硬件PWM版本）
 * 修改内容：
 * 1. PB12软件PWM改为PA8(TIM1_CH1)硬件PWM
 * 2. 删除softPwmUpdate()
 * 3. 收到Pxx命令后直接更新PWM输出
 * 4. 修复 Serial.print(V12,2) -> Serial.print(v12,2)
 */

#define V12_PIN   PA1
#define V5_PIN    PA2
#define V33_PIN   PA3
#define CUR_PIN   PA0

#define R12_PIN   PA4
#define G12_PIN   PA5
#define R5_PIN    PA6
#define G5_PIN    PA7
#define R33_PIN   PB0
#define G33_PIN   PB1

#define FAN_PWM_PIN PA8
#define FAN_DET_PIN PB13

#define ADC_MAX 4095.0f
#define VREF    3.3f

float defTh[3][2] = {{10.0f,13.0f},{4.0f,5.7f},{2.9f,3.9f}};
float pcTh[3][2];
bool pcThValid[3]={false,false,false};

uint8_t pwmDuty=0;
bool lastFanState=true;

float readVoltage(uint8_t pin,float ratio)
{
  uint32_t sum=0;
  for(int i=0;i<8;i++) sum+=analogRead(pin);
  return (sum/8.0f)/ADC_MAX*VREF*ratio;
}

char updateLeds(int ch,float v,uint8_t rPin,uint8_t gPin)
{
  float lo=pcThValid[ch]?pcTh[ch][0]:defTh[ch][0];
  float hi=pcThValid[ch]?pcTh[ch][1]:defTh[ch][1];

  if(v<lo){
    digitalWrite(rPin,HIGH);
    digitalWrite(gPin,LOW);
    return 'R';
  }
  else if(v<hi){
    digitalWrite(rPin,LOW);
    digitalWrite(gPin,HIGH);
    return 'G';
  }
  else{
    digitalWrite(rPin,LOW);
    digitalWrite(gPin,LOW);
    return 'O';
  }
}

void handleCommand(char *cmd)
{
  if(cmd[0]=='P'){
    int d=atoi(cmd+1);
    pwmDuty=constrain(d,0,100);

    uint16_t pwmValue=map(pwmDuty,0,100,0,255);
    analogWrite(FAN_PWM_PIN,pwmValue);

    Serial.print("RECV PWM=");
    Serial.println(pwmDuty);
  }
  else if(cmd[0]=='T'){
    int ch=cmd[1]-'1';
    if(ch>=0 && ch<3){
      float lo,hi;
      if(sscanf(cmd+2,":%f:%f",&lo,&hi)==2){
        pcTh[ch][0]=lo;
        pcTh[ch][1]=hi;
        pcThValid[ch]=true;
      }
    }
  }
  else if(cmd[0]=='R'){
    int ch=cmd[1]-'1';
    if(ch>=0 && ch<3) pcThValid[ch]=false;
  }
}

void setup()
{
  Serial.begin(115200);
  analogReadResolution(12);

  pinMode(R12_PIN,OUTPUT);
  pinMode(G12_PIN,OUTPUT);
  pinMode(R5_PIN,OUTPUT);
  pinMode(G5_PIN,OUTPUT);
  pinMode(R33_PIN,OUTPUT);
  pinMode(G33_PIN,OUTPUT);

  pinMode(FAN_PWM_PIN,PWM);
  pinMode(FAN_DET_PIN,INPUT);

  analogWrite(FAN_PWM_PIN,0);
}

void loop()
{
  static char buf[32];
  static uint8_t idx=0;

  while(Serial.available()){
    char c=Serial.read();

    if(c=='\n' || c=='\r'){
      if(idx>0){
        buf[idx]=0;
        handleCommand(buf);
        idx=0;
      }
    }
    else if(idx<sizeof(buf)-1){
      buf[idx++]=c;
    }
  }

  static uint32_t lastSend=0;

  if(millis()-lastSend>=200){
    lastSend=millis();

    float v12=readVoltage(V12_PIN,12.0f);
    float v5=readVoltage(V5_PIN,2.0f);
    float v33=readVoltage(V33_PIN,2.0f);
    float vout=readVoltage(CUR_PIN,1.0f);
    float fanI=vout/1.2f;

    char l1=updateLeds(0,v12,R12_PIN,G12_PIN);
    char l2=updateLeds(1,v5,R5_PIN,G5_PIN);
    char l3=updateLeds(2,v33,R33_PIN,G33_PIN);

    bool fan=digitalRead(FAN_DET_PIN);

    if(fan!=lastFanState){
      lastFanState=fan;
      if(!fan) Serial.println("ALARM:FAN_LOST");
    }

    Serial.print("V1="); Serial.print(v12,2);
    Serial.print(";V2="); Serial.print(v5,2);
    Serial.print(";V3="); Serial.print(v33,2);
    Serial.print(";L1="); Serial.print(l1);
    Serial.print(";L2="); Serial.print(l2);
    Serial.print(";L3="); Serial.print(l3);
    Serial.print(";I="); Serial.print(fanI,2);
    Serial.print(";FAN="); Serial.print(fan?1:0);
    Serial.print(";PWM="); Serial.println(pwmDuty);
  }
}
