# TimeTable
사용자가 시간표를 추가하여 설정된 시간에 진동과 벨소리가 작동되며 알람이 울리도록하는 기능이다.
- 시간표를 추가하여 새로운 알람을 등록한다.
- 알람이 설정된 시간이되면 AlarmReceiver가 호출되고 AlarmService를 실행한다.
- 알람을 종료하기 위해 다시 AlarmReceiver를 호출하여 AlarmService를 정지시킨다.

TimeTable은 (링크)TimeTableView, TimeTableActivity EditActivity, AlarmReceiver, AlarmService 로 나눠서 설명할 것이다.

## TimeTableView
TimeTableView를 사용하기에 앞서 build.gradle(Module:**project**) 파일에 다음을 추가합니다.
~~~java
allprojects {
    repositories {
        ...
        maven { url 'https://jitpack.io'}
    }
}
~~~
> github에서 프로젝트를 Clon or Download 했을 때는 직접 ProjectView를 Project로 변경하면 수정할 수 있습니다.
> ![Projectmodule2](https://user-images.githubusercontent.com/46085058/85412389-7cb93d00-b5a4-11ea-95f1-0d0cf25c5061.png) 

build.gradle(Module:**app**) dependency에 다음을 추가합니다.
~~~java
dependencies {
    implementation 'com.github.tlaabs:TimetableView:1.0.3-fx1'
}
~~~
### TimeTableView in Layout.xml
~~~java
<com.github.tlaabs.timetableview.TimetableView  
    android:id="@+id/timetable"  
    android:layout_width="match_parent"  
    android:layout_height="wrap_content"  
    app:column_count="8"  // 일주일을 표시 
    app:row_count="25"  // 24시를 표시
    app:start_time="0"  // 시작시간을 0으로 표시
    app:header_title="@array/my_header_title"  // header title(요일)을 values/strings.xml에서 받아온다.
    app:header_highlight_color="@color/mainColor"  // 사용자가 별도로 지정한 header색상 변경(ex : 오늘의 요일)
    app:header_highlight_type="color"  // color 또는 image를 지정
  />
~~~
> 더 많은 [TimeTable layout 설정](https://github.com/tlaabs/TimetableView#attribute-descriptions)

#### app:header_title 변경
TimeTableView의 header속성을 변경할 수 있습니다.
~~~java
<string-array name="my_header_title">  
   <item></item> <item>Mon</item>  
   <item>Tue</item>  
   <item>Wed</item>  
   <item>Thu</item>  
   <item>Fri</item>  
   <item>Sat</item>  
   <item>Sun</item>  
</string-array>
~~~
> TimeTableViewLayout.xml 
> ~~~java
> app:row_count = "8" // row_count가 item 갯수 보다 1 더 커야 합니다. 
> ~~~

### TimeTableActivity.Java
TimeTableActivity의 주요 기능은 시간표를 생성하고 알람을 등록해주는 역할을 수행합니다.
기존 AlarmManger에 등록된 알람과의 충돌을 방지하기 위해서 기존의 알람을 모두 삭제한 후 재등록합니다.
~~~java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_timetable);
    // init firebase
    ...
    // AlarmManger Service
    alarmManager = (AlarmManager) getSystemService(ALARM_SERVICE);
    
    init(); // TimeTableView의 view 객체 및 listener 선언
    checkPictureCount(); // 이미지매칭용 사진이 저장되어있는지 확인
    dayCheckZero(); // TimeTable을 처음 사용할 때 출석 값 0으로 초기화
    
    // addValueEventListener를 선언하여 시간표가 수정될 때마다 Alarm을 갱신 해줍니다.
    mDatabaseReference.addValueEventListener(new ValueEventListener() {
        @Override
        public void onDataChange(@NonNull DataSnapshot dataSnapshot) {
            if(dataSnapshot.child("count").getValue(Integer.class) != null){
                count = dataSnapshot.child("count").getValue(Integer.class);
                alarmOff(count);
            }
            if(dataSnapshot.child("table").getValue(String.class) != null){
                timetable.load(dataSnapshot.child("table").getValue(String.class));
                AddAlarm(dataSnapshot.child("table").getValue(String.class));
            }
        }
        @Override
        public void onCancelled(@NonNull DatabaseError databaseError) { }
    });
}
~~~
알람을 등록, 수정, 삭제 기능을 수행하기 위해서 EditActivity에서 받아온 data를 활용합니다.
받아온 data는 timetable.createSaveData() 를 활용하여 Json형식으로 저장할 수 있습니다.
Json형식으로 바뀐 data를 Database에 저장합니다.
~~~java
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    switch (requestCode) {
        case REQUEST_ADD:
            if (resultCode == EditActivity.RESULT_OK_ADD) {
                ArrayList<Schedule> item = (ArrayList<Schedule>) data.getSerializableExtra("schedules");
                timetable.add(item); // 받아온 Schedule List을 Json 형식으로 저장합니다.
                mDatabaseReference.child("table").setValue(timetable.createSaveData()); // Database에 Json형식으로 저장합니다.
                Toast.makeText(this, "시간표가 추가 되었습니다.", Toast.LENGTH_SHORT).show();
            }
            break;
        case REQUEST_EDIT:
            /** Edit -> Submit */
            if (resultCode == EditActivity.RESULT_OK_EDIT) {
                int idx = data.getIntExtra("idx", -1);
                ArrayList<Schedule> item = (ArrayList<Schedule>) data.getSerializableExtra("schedules");
                timetable.edit(idx, item); // 받아온 data로 idx의 Schedule을 수정합니다.
                mDatabaseReference.child("table").setValue(timetable.createSaveData()); // Database에 Json형식으로 저장합니다.
                Toast.makeText(this, "시간표가 수정 되었습니다.", Toast.LENGTH_SHORT).show();
            }
            /** Edit -> Delete */
            else if (resultCode == EditActivity.RESULT_OK_DELETE) {
                int idx = data.getIntExtra("idx", -1);
                timetable.remove(idx); // 특정 idx의 Schedule을 삭제합니다.
                mDatabaseReference.child("table").setValue(timetable.createSaveData()); // Database에 Json형식으로 저장합니다.
                Toast.makeText(this, "시간표가 삭제 되었습니다.", Toast.LENGTH_SHORT).show();
            }
            break;
        }
    }
~~~
> Json형식
> ![Json형식](https://user-images.githubusercontent.com/46085058/85429692-12ab9280-b5ba-11ea-882b-b958e299604f.PNG)

#### Method
출석률을 초기화해주는 메서드(링크)
~~~java
public void dayCheckZero(){...}
~~~
출석체크의 이미지매칭을 위한 등록된 사진갯수를 확인하는 메서드(링크)

~~~java
public void checkPictureCount(){...}
~~~
JsonParsing 후 알람을 등록하는 메서드 ( 알람의 등록과 삭제는 따로 다루도록 한다.) (링크)
~~~java
public void AddAlarm(String json){...}
~~~
> JsonParse 참고 : [https://jang8584.tistory.com/185](https://jang8584.tistory.com/185)
> 

알람을 삭제하는 메서드 ( 알람의 등록과 삭제는 따로 다루도록 한다.) (링크)
~~~java
public void alarmOff(int tmpcount){...}
~~~
#### Alarm등록 및 삭제 
Alarm의 등록 및 삭제는 중요한 몇 가지가 기능을 좌우하기 때문에 별도로 설명하도록 한다.
> AlarmReceiver.class가 있다는 가정하에 설명하겠다.

알람의 object 선언
~~~java
//Alarm object  
private AlarmManager alarmManager;  
private PendingIntent pendingIntent;
~~~
> PendingIntent은 별도의 컴포넌트에게 내가 보내고자하는 intent를 대신 전달하고자 할 때 선언한다.
> 주로 외부에서 Activity, Broadcast, Service를 사용해야 할 때 선언한다.
> 여기에서는 Broadcast 컴포넌트로 다뤄보도록 하겠다.
> 
Pendingintent 설정
~~~java
Intent intent = new Intent(getApplicationContext(), AlarmReceiver.class);  
intent.putExtra("weekday", obj3.get("day").getAsInt());  
intent.putExtra("state", "on"); // state 값이 on 이면 알람시작, off 이면 중지, day는 Receiver에서 구분  
intent.putExtra("reqCode", i);  
pendingIntent = PendingIntent.getBroadcast(getApplicationContext(), i, intent, PendingIntent.FLAG_CANCEL_CURRENT);
~~~
> intent에 담아야할 정보들을 Intent.putExtra() 로 담아주고, Broadcast 컴포넌트를 받아온다.
> 이 때 **FLAG**에 주목해야 하는데 FLAG는 Pendingintent의 충돌을 막아주는 아주 중요한 역할을 수행한다.
> - FLAG_CANCEL_UPDATE : 이전에 생성된 pendingintent가 있으면 삭제하고 새롭게 생성한다.
> - FLAG_UPDATE_CURRENT : 이전에 생성된 pedingintent의 내용을 갱신 한다.
> - FLAG_ONE_SHOT : 일회용으로 생성 pendingintent를 단 1회만 사용한다. ( 등록한 위젯이 있다면 1회 클릭 후에는 반응하지 않는다. )
> - FLAG_NO_CREATE : 이미 생성된 것이 있다면 삭제한다.
> 
> Make You Study에서는 알람시간 변경 및 추가가 계속 이루어져야 함으로 FLAG_CANCEL_UPDATE 또는 FLAG_UPDATE_CURRENT를 사용한다.

AlarmManger 설정
AlarmManger는 Pendingintent를 어느 시점에 실행할지 결정해주는 역할을 수행한다.
~~~java
Calendar calendar = Calendar.getInstance(); // Calendar 객체 생성
calendar.set(Calendar.HOUR_OF_DAY, obj4.get("hour").getAsInt()); // 시간 설정
calendar.set(Calendar.MINUTE, obj4.get("minute").getAsInt());  // 분 설정
calendar.set(Calendar.SECOND, 0); // 초 설정(되도록이면 Default로 0을 둔다.)
~~~
> obj4.get("...").getAsInt() 는 MakeYouStudy의 시간을 불러오는 예제이다. 원하는 시간대로 설정해주면 된다.
> 
시간을 설정했으면 알람을 등록 해주어야 한다.  이때 **API**마다 실행 방식이 다르기 때문에 꼭 아래와 같이 작성해 주어야 한다.
~~~java
if(Build.VERSION.SDK_INT < Build.VERSION_CODES.M){
    if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT){
        // API 19이상 API 23미만
        alarmManager.setExact(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), pendingIntent);
    }else{
        // API 19미만
        alarmManager.set(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), pendingIntent);
    }
}else{
    // API 23이상
    alarmManager.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), pendingIntent);
}
~~~
![extends_AlarmReceiver](https://user-images.githubusercontent.com/46085058/85444766-a3d83480-b5cd-11ea-9f89-153a5bbdac48.png)


![test](https://user-images.githubusercontent.com/46085058/85445065-f0237480-b5cd-11ea-93ef-efc1a6e6ea13.PNG)
