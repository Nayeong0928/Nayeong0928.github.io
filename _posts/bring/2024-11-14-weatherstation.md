---
title: WeatherStation 구조 변경
author: qoskdud0928
date: 2024-11-14
category: bring
layout: post
---

### WeatherStationManager


```java

@Component
@Slf4j
public class WeatherStationManager {

    /**
     * weatherStations: 장소-시간별 기상관측
     *      HashMap<Integer, WeatherStation>: 해당시간-관측소
     */
    private HashMap<String, HashMap<Integer, WeatherStation>> weatherStations=new HashMap<>();
    

```

WeatherStation을 관리해주는 WeatherStationManager 라는 객체를 컴포넌트로 새로 등록했다.
hash의 키로 Address 객체의 code 값을 저장하도록 해서
장소를 통해 관측소를 조회할 수 있도록 했다.


```java

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class ScheduleService {

    private final ScheduleRepository scheduleRepository;
    private final MemberRepository memberRepository;
    private final AddressRepository addressRepository;
    private final WeatherStationManager weatherStationManager;

```

해당 빈은 Service에 주입하도록 해서 사용했다.


### 기타 로직 변경


```java

@Getter
public class WeatherStation {

    private int time;
    private WeatherInfo weatherInfo;
    private HashMap<Long, WeatherObserver> observers; // scheduleId-스케줄 구독자 쌍

```


WeatherStation 정보도 수정했다. 
날씨 정보는 시간에 따라 달라지기 때문에 시간과 날씨 정보를 같이 저장하도록 했다.
오픈 api를 읽어와서 weatherInfo가 업데이트되면 observers로 등록해둔 
WeatherObserver들의 정보도 갱신된다. 


```java

@Getter
public class WeatherObserver {

    private List<String> items; // 필요한 소지품

    public void update(WeatherInfo weatherInfo){
        ItemListGenerator listGenerator=new ItemListGenerator();
        items=listGenerator.getPackingList(weatherInfo);
    }

    public String getItemList(){

        if(items==null){
            return "";
        }

        StringBuilder sb=new StringBuilder();
        for(String item: items){
            sb.append(item);
            sb.append("\n");
        }
        return sb.toString();
    }
}

```

WeatherObserver들이 날씨를 구독하는 이유는 날씨에 따른 소지품을 확인하기 위해서이다.
굳이 날씨 정보까지 저장할 필요가 없어보여서 소지품 정보만 저장하는 것으로 수정했다.
기존에는 WeatherInfo가 바뀌었을 때 WeatherObserver의 변수로 저장되던 WeatherInfo를 삭제했다.
날씨가 변화함에 따라 바뀐 소지품 정보를 업데이트하도록 한다.

