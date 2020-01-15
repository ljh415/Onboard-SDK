# DJI Onboard SDK (OSDK) 3.9

###  main를 수정 (버전 3.8에서 실행했었음)

- GPIO핀을 이용해서 초음파센서를 사용

  > 추가한 내용들만 간략하게 적는다

1. 라즈베리파이의 GPIO핀 이용, 센서에 핀 할당

```c++
// 라즈베리파이의 GPIO핀을 이용하기 위해
#include <wiringPi.h>
wiringPiSetup();

// 기존의 dist클래스, calc함수 변형
class dist {
	private :
		float s, e;
	public :
    float result;
		float measures(float i);
		float measuree(float j);
		int calc();
};

float dist::measures(float i) {
	s = i;
}

float dist::measuree(float j) {
	e = j;
}

int dist::calc() {
	result = (e - s) / 58;
	//std::cout << "distance(cm) : " << result << std::endl;
  return result;
}

// 4방향의 초음파 센서에서 값을 저장할 변수
int result1, result2, result3, result4;
dist d1, d2, d3, d4;

// 초음파 센서에 핀을 할당해준다 (똑같이 각각 센서2, 3, 4도 할당)
//ultra 1 -> Forward
pinMode(Trig1, OUTPUT);
pinMode(Echo1, INPUT);
```

2. 값을 받아 들인다

   >- ```cnt```변수를 선언하여 무한루프안에서 ```while(cnt < 12)``` 돌면서 1, 2, 3, 4번 센서의 값들을 3개씩 받아들이는데 이 값이 1000(값을 이상하게 받는 경우)을 넘지않는 정상적인 값들을 ```avg[cnt]```에 넣어준다.
   >
   >- 1000이상의 값이 들어오게 된다면 센서가 잘못 받아들였다고 여겨서 ```goto```를 이용해서 다시 센서값을 받아들이도록 한다(```d1re```부분으로)

```c++
// case a와 b로 나뉘는데 b부분에서 초음파값을 받아들이고
// 그 값들을 계산해서 장애물을 회피하도록 할것

case 'b':
	monitoredTakeoff(vehicle);

	int cnt;
    cnt = 0;
    
while(1) {
    // 센서마다 3개의 값을 받기 위해 cnt<12
    while(cnt < 12){
        
        //1번 초음파
        
        d1re:
        //ultrasonic seonsor_1, calcuating distance    
      	digitalWrite(Trig1,0);
      	digitalWrite(Trig1,1);
      	delayMicroseconds(10);
      	digitalWrite(Trig1,0);
      	while(digitalRead(Echo1) == 0)
        	d1.measures(micros());
      	while(digitalRead(Echo1) == 1)
        	d1.measuree(micros());
        
      	//std::cout<<"1st Sensor : ";
      	result1 = d1.calc();
      
      	if(result1 < 1000){
			avg[cnt] = result1;
		  	cnt++;
	  	}
	    else{
		    goto d1re;
	  	}
        
        // 이후에 나머지 3개의 초음파 센서도 똑같이
    }
    // 반복문 탈출후 cnt를 0으로 초기화
    cnt = 0;
```

3. 센서별로 받아들인 값들의 평균값을 통해서 현재 드론을 움직이게 된다

   > - ```check```함수를 이용해서 ```avg```배열의 값들을 이용해 평균 거리를 계산후 각각의 상황을 ```judge```변수에 저장
   >
   > - 장애물 case

| case |  1   |  2   |  3   |  4   |  5   |  6   |  7   |  8   |  9   |  10  |  11  |  12  |  13  |  14  |  15  |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
|  앞  |  ●   |      |      |      |  ●   |      |      |  ●   |      |  ●   |  ●   |      |  ●   |  ●   |  ●   |
|  뒤  |      |  ●   |      |      |  ●   |  ●   |      |      |  ●   |      |  ●   |  ●   |  ●   |      |  ●   |
|  왼  |      |      |  ●   |      |      |  ●   |  ●   |  ●   |      |      |  ●   |  ●   |      |  ●   |  ●   |
|  오  |      |      |      |  ●   |      |      |  ●   |      |  ●   |  ●   |      |  ●   |  ●   |  ●   |  ●   |



```c++
    int judge = check(avg);

// avoiding : switch_case
	switch (judge){
            case 1 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, -15, 0, 0, 0);     break; //0001
        	case 2 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 15, 0, 0, 0);      break; //0010
       	 	case 4 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 0, 15, 0, 0);      break; //0100
        	case 8 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 0, -15, 0, 0);     break; //1000 
        
        	case 3 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 0, 15, 0, 0);      break; //0011
        	case 6 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 15, 15, 0, 0);      break; //0110
        	case 12 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 15, 0, 0, 0);     break; //1100
        	case 5 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, -15, 15, 0, 0);     break; //0101
        	case 10 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 15, -15, 0, 0);    break; //1010
        	case 9 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, -15, -15, 0, 0);    break; //1001
        
        	case 7 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 0, 15, 0, 0);     break; //0111
        	case 14 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 15, 0, 0, 0);     break; //1110
        	case 11 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 0, -15, 0, 0);     break; //1011
        	case 13 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, -15, 0, 0, 0);    break; //1101
   
        	case 15 : std::cout<<"case : "<<judge<<std::endl; moveByPositionOffset(vehicle, 0, 0, 15, 0);     break; //1111
        	default : moveByPositionOffset(vehicle, 0, 0, 0, 0);
      }
  
    
```

- ```check```함수

  > - ```avg```배열의 값의 평균을 ```avg_d```배열에 저장
  >
  > |    avg_d[0]     |    avg_d[1]     |    avg_d[2]     |    avg_d[3]     |
  > | :-------------: | :-------------: | :-------------: | :-------------: |
  > | 1번 센서 평균값 | 2번 센서 평균값 | 3번 센서 평균값 | 4번 센서 평균값 |
  >
  > - ```re```변수에 위의 각 센서에 대한 평균값들을 기존에 설정해놓은 거리```dis``` 보다 가까운 위치에 있다면 장애물이 있다고 판단, 위의 케이스에 맡게 값들 더해서 맨 마지막에 반환

```c++
int check(float avg[])
{
  int re =0;
  
  float avg_d[4] = {0, };
  
  for(int i = 0; i < 4; i++){
    
    avg_d[i] = (avg[i+0] + avg[i+4] + avg[i+8])/3;
    
  }
  
  for(int i = 0; i < 4; i++)
  {
    std::cout <<i+1<<"'s avg_distance(cm) : " << avg_d[i] << std::endl;
  }
  
  for(int i = 0; i < 4; i++){
    if(avg_d[i] < dis){
      switch(i){
        case 0 : re = re + 1; break;
        case 1 : re = re + 2; break;
        case 2 : re = re + 4; break;
        case 3 : re = re + 8; break;
      }
    }
    else re = re + 0;
  }
  
  return re; 
    
}
```

