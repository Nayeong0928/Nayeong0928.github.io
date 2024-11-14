---
title: WeatherStation 구조 고민
author: qoskdud0928
date: 2024-11-12
category: bring
layout: post
---

날씨 api 요청 로직에 디자인패턴을 적용해보고 싶었다.

단기예보 OpenApi는 장소와 예보 시간을 통해 요청하면, 기준시간에서 모레까지의 예보 정보를 가져온다.
예보 시간이 1일 8회밖에 안 되니까 A라는 장소에 시간을 등록한 사람들이 날씨를 조회할 때마다 
굳이 API 요청을 보낼 필요 없이 장소별로 8번만 API 요청을 보내고 그 결과를 재활용하면 되는 것이다.

옵저버 패턴을 도입하여 '해당 장소 & 시간의 날씨'를 Subject, 
해당 장소에 스케줄을 등록한 사람들을 Observer로 구성해보기로 했다.

그렇게 다음과 같이 클래스 다이어그램을 설계했다.

![클래스 다이어그램 버전1](/assets/post-img/ver02-3_UML.jpg)

장소별로 기상관측기(WeatherStation) 객체를 두기로 했기 때문에 
Address 엔티티에 관련된 메소드를 구현하기로 했다.
DB에 저장하고 싶지는 않았지만 장소와 관련된 건 최대한 Address 엔티티에 모아보고 싶었다.
마침 `@Transient` 어노테이션의 존재를 알게 되어 다음처럼 Address 엔티티를 구현했다.

``` java
@Entity
@Getter
public class Address {

    @Id @GeneratedValue
    @Column(name = "address_id")
    private Long id;

    private String code;
    private String addr1;
    private String addr2;
    private String addr3;

    @Embedded
    private Location location;

    @Transient
    private WeatherStation weatherStation;
    
    ...
```

엑셀 데이터를 파싱한 주소 데이터가 잘 저장되길래 문제가 없는 줄 알았다.
스케줄이 추가되면 Observer를 생성해서 WeatherStation에 등록하는 코드까지 작성했다.

그런데 스케줄을 추가하려니 WeatherStation이 null이라는 오류가 발생하면서 제대로 추가가 되지 않는 문제가 발생했다.

### `@PostLoad` 적용

검색을 해보니 `@Transient`은 영속성 컨텍스트에서 관리하지 않기 때문에 엔티티가 초기화될 때 
초기화되지 않고 무시된다고 한다.

대신 이런 경우에 사용할 수 있도록 `@PostLoad`라는 어노테이션이 있다.
이 어노테이션을 적용한 메소드는 엔티티가 영속성 컨텍스트에서 조회된 직후 호출된다.
따라서 다음과 같이 코드를 수정했다.

```java

    @PostLoad
    public void initWeatherStation() {
        weatherStation=new WeatherStation(location);

        for(Schedule schedule: schedules){
            weatherStation.addObserver(schedule.getTime(), schedule.getId());
        }
    }

```

엔티티가 영속성 컨텍스트에서 조회된 직후 initWeather() 메소드가 실행되어 WeatherStation이 초기화된다.

Observer는 해당 장소와 시간에 스케줄이 있는 사람만 해당하기 때문에
장소에 등록된 스케줄들을 이용해서 시간별로 Observer를 생성해서 더해주도록 했다. 

![디버깅 화면](/assets/post-img/2024-11-13-debuging.JPG)

'소지품 불러오기' 버튼을 클릭하면 오픈 api를 불러와서 해당 장소의 시간별 날씨 정보가 WeatherInfo로 세팅된다.
`notifyObserver(time)` 를 통해 장소의 time 시간에 스케줄을 등록한 Observer들의 WeatherInfo도 설정되는 것을 확인할 수 있었다.
다음처럼 오픈 api 요청들이 파싱되어 날씨 정보에 잘 세팅된 것을 확인할 수 있다.


### 초기화 문제 

```java

    public Map<Integer, List<String>> itemsBySchedule(Long memberId){
        Member member = memberRepository.findOne(memberId);
        Map<Integer, List<String>> items=new HashMap<>();

        for(Schedule schedule: member.getSchedules()){
            List<String> itemList = schedule.getAddress().getWeatherStation().itemList(schedule.getId());
            items.put(schedule.getTime(), itemList);
        }

        return items;
    }

```

WeatherStation에 등록된 WeatherInfo(기온 등의 구체적인 날씨 정보)를 통해 소지품 목록을 가져오는 로직이다.
날씨 정보도 잘 파싱되었고 스케줄 정보도 저장되었는데 소지품 목록이 뜨지 않았다.
WeatherStation의 WeatherInfo의 값들이 다 사라져버린 것이다.

원인은 영속성 컨텍스트 때문이었다.
영속성 컨텍스트는 트랜잭션이 생성될 때 생성되고, 트랜잭션이 종료될 때 종료된다. 
Service 메소드들에 @Transactional을 설정해주었기 때문에 새로운 http 요청에서는 새로 영속성 컨텍스트가 생성된다. 
@PostLoad는 영속성 컨텍스트가 엔티티를 불러오고 나서 실행되기 때문에 날씨 정보를 저장한 WeatherStation은 새로 초기화되는 것이다.

날씨 정보를 DB에 저장하지 않고 프로젝트 내에서만 사용하려고 @Transient를 사용하려던 건데 엔티티의 생명주기에 종속적인 것 같다. 
WeatherStation 관리 방식을 바꿔야겠다.

