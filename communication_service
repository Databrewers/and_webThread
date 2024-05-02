package com.example.kdcommunicator_impulseversion;

import android.app.Notification;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.PendingIntent;
import android.app.Service;
import android.bluetooth.BluetoothAdapter;
import android.bluetooth.BluetoothDevice;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.graphics.Color;
import android.media.AudioAttributes;
import android.media.AudioManager;
import android.media.RingtoneManager;
import android.media.SoundPool;
import android.net.Uri;
import android.os.Binder;
import android.os.Build;
import android.os.Handler;
import android.os.IBinder;
import android.os.Message;
import android.os.PowerManager;
import android.speech.tts.TextToSpeech;
import android.support.v4.app.NotificationCompat;
import android.util.Log;

import java.util.Locale;
import java.util.UUID;

import static android.speech.tts.TextToSpeech.ERROR;
import static com.example.kdcommunicator_impulseversion.ServiceFormat.CHANNEL_ID;

public class CommunicationService extends Service {

    UUID uuid = java.util.UUID.fromString("00001101-0000-1000-8000-00805f9b34fb"); // 내가 사용할 서비스의 고유 인식번호
    static final String APPNAME = "KDcommunicator";
    public BluetoothAdapter bluetoothAdapter;
   // Notification notification;

    //BT STATUS FLAGS
    final int STATE_CONNECTING = 1; //이것은 오직 메인에게만 보내는 것으로, 메인이 커넥팅 중에는 서비스 종료를 하지 못하게 한다.
    final int STATE_CONNECTION_FAILED = 2; //bt 연결 실패. 메인에게도 보낸다
    final int STATE_CONNECTED = 3; //bt 연결 성공. 메인에게도 보낸다
    final int STATE_BT_MESSAGE_RECEIVED = 4; //BT mainThread가 핸들러를 통해 전달하는 플래그로, bt 메세지 수신 상태이다.
    final int STATE_WEB_MESSAGE_RECEIVED = 5; // webThread가 핸들러를 통해 전달하는 플래그로, 웹 메세지 수신 상태이다

    final  int STATE_NOT_PAIRED = 11; //웹에서 sender(220)이 설정한 260 receiverChannel 개수 > 현재 웹에 접속된 260 receiver 개수, 이것을 메인 핸들러에 보낸다
    final  int STATE_IS_PAIRED = 12;//웹에서 sender(220)이 설정한 260 receiverChannel 개수 <= 현재 웹에 접속된 260 receiver 개수, 이것을 메인 핸들러에 보낸다


    //블루투스 연결 상태 불리언들
    private boolean btConnected = false; //bt 연결되어있는가?
    private boolean onceBtConnected = false; //이 서비스에서 bt가 최초로 한번 연결된적 있는가?
    //왜 이것을 따지냐면 최초 안내를 구분하기 위해서도 있고, bt가 최초 연결 되어야 비로소 웹의 연결을 허가해 주기 때문이다

    private BluetoothDevice koda;  //메인에서 intent로 건네받는 kd 기기 주소이다.
    public BT_MainThread sendReceive = null;
    //예전 버전에서 clientClass의 소켓이 serverClass의 소켓과 만나 만들어진 유효한 socket을 통해 생성하는 bt mainThread이다. 이 스레드가 블루투스 송 수신을 담당한다
    public BT_ClientThread clientClass;  // 해당 kd 기기주소를 대변하는 클래스로, accept 대기하고 있는 서버에 연결 시도한다. 연결 실패시 초기화 필요하다. (이 때문에 싱글톤 곤란했다)
  //  public BT_ServerThread serverClass;  // 최초로 한번 run 하고 그 다음에 초기화되거나 하지 않는다. 해당 휴대폰의 블루투스 서버를 오픈하여 accept 대기한다

    //웹 관련 값들.
    private int role=1;
    private final int SENDER=1;
    private final int RECEIVER=2;
    private int serverNum=0;
    String url = "http://13.237.36.33:3000"; //http://13.237.36.33:3000";// http://13.237.36.33:3000"; //"http://13.237.36.33:3000/ // "https://kodaimpulse.herokuapp.com/";
    private Intent intent;
    public WebThread webThread = null;

    //'준비'에 의한 설정값들
    private int receiverChannel; // 220에서 260의 개수 설정. 스트링으로 받아 이쪽에서 인트로 변환해 사용한다
    private String deviceNumber; //260에서 기기번호 설정. 스트링으로 쓰인다.
    //private  BroadcastReceiver btBroadcastReceiver = null;

    //사운드 관련 값들. 딩동 == yes, 땡 == no.
    SoundPool soundpool;
    private int dingdong; //얘네들이 진짜 사운드풀 변수이고
    private int ddang;

    final int DINGDONG = 1; //이것은 노티피케이션시 넘겨주는 플래그이다
    final int DDANG = 2;
    final int NONE=0;
    public BroadcastReceiver btReceiver=null;

    TextToSpeech tts;
    private boolean isSpeak = true;
    final int VERYFAST=0;
    final int FAST=1;
    final int PRECISE=2;
    private final int SILENCE = 3;

    private int voiceSpeed = FAST;// true 면 speakImpulse_long을 호출하고, false면 _fast를 호출한다

    //bt 클라이언트스레드를 오직 하나씩만 생성하도록 하기 위한 플래그이다. bt Client가 생성될 때 true이며, bt 연결 결과(connected or failed)를 받을 때 false이다.
    // 260이 켜고 끄기가 반복되는 상황에서, ACL_Disconnect가 여러번 호출되어 클라이언트 스레드가 여러개가 생겨버리는 상황이 발생할 수 있다. 이를 막는 플래그이다.
    private boolean isBTclientRunning = false;




    @Override
    public void onCreate(){


        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) { //P버전에서는 해당 서비스가 반드시 FOREGROUND이어야 할 필요가 있다. 그 외에는 아님.
            Log.d("gd","pass oncreate notification");
            String NOTIFICATION_CHANNEL_ID = "com.example.kdcommunicator_serviceversion";
            String channelName = "My Background Service";
            NotificationChannel chan = new NotificationChannel(NOTIFICATION_CHANNEL_ID, channelName, NotificationManager.IMPORTANCE_NONE);
            chan.setLightColor(Color.BLUE);
            chan.setLockscreenVisibility(Notification.VISIBILITY_PRIVATE);
            NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
            assert manager != null;
            manager.createNotificationChannel(chan);

            NotificationCompat.Builder notificationBuilder = new NotificationCompat.Builder(this, NOTIFICATION_CHANNEL_ID);
            Notification notification = notificationBuilder.setOngoing(true)
                    .setSmallIcon(R.drawable.ic_android_ex)
                    .setContentTitle("App is running in background")
                    .setPriority(NotificationManager.IMPORTANCE_MIN)
                    .setCategory(Notification.CATEGORY_SERVICE)
                    .build();
            startForeground(1, notification);
        }


        super.onCreate();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId){
        this.intent = intent;
        getIntent(intent); //메인에서 넘겨주는 인텐트 값을 받아온다. koda 장치, 역할, 서버 ...

        if(btReceiver == null)
            create_btReceiver(); //인텐트 필터를 등록한다. IntentFilter : ACL_CONNECTED(잘 안씀) , ACL_DISCONNECTED(BT 연결 끊어졌을 때)

        adapt_BroadCastReceiver(); //그 인텐트필터에 반응하는 리시버 인스턴스를 생성하고 broadcastReceiver로서 등록한다.

        init_wakeLock(); // 앞으로 10시간 동안, 이 서비스는 화면 잠금 상태에서도 돌아간다.
        init_Service(); // 고고


        bluetooth_Pairing(); //블루투스 연결 최초 개시한다. BT Client 를 생성하고 페어링 시도한다.



        return START_NOT_STICKY; //일단 지금은 서비스가 꺼지면 꺼지는걸로 했다.
    }


    public void init_wakeLock(){

        PowerManager powerManager = (PowerManager) getSystemService(POWER_SERVICE);
        PowerManager.WakeLock wakeLock = powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK  |
                PowerManager.ACQUIRE_CAUSES_WAKEUP |
                PowerManager.ON_AFTER_RELEASE, "My:Tag");

        wakeLock.acquire(600*60*1000L /*10 시간동안의 wakeLock 가동. 이 숫자를 0으로 두지 말 것*/);

    }




    public void getIntent(Intent intent){

        koda = intent.getExtras().getParcelable("KDdevice");
        role = intent.getIntExtra("role",SENDER);
        voiceSpeed = intent.getIntExtra("voiceSpeed",FAST);
        if(voiceSpeed == SILENCE){
            dontSpeak();
        }
        deviceNumber = intent.getExtras().getString("deviceNumber","01");

        String temp = intent.getExtras().getString("receiverChannel","1");
        receiverChannel = Integer.parseInt(temp);

        serverNum = intent.getIntExtra("serverNum",2);

        if(serverNum==1){
            url = "http://13.237.36.33:3000";
            //url =  "https://kodaimpulse.herokuapp.com/";
        }
        else if(serverNum==2){
            url = "https://kodaimpulsetest.herokuapp.com/";
        }


        soundpool = new SoundPool(5, AudioManager.STREAM_MUSIC,0);
        ddang = soundpool.load(getApplicationContext(),R.raw.ddang,1);
        dingdong = soundpool.load(getApplicationContext(),R.raw.dingdong,2);


        tts = new TextToSpeech(this, new TextToSpeech.OnInitListener() {
            @Override
            public void onInit(int status) {
                if (status != ERROR) {
                    tts.setLanguage(Locale.KOREAN);
                    Log.d("gd", "tts set");
                }
            }
        }); //일단 만들고 셧다운을 시킴. 만드는것은 필수로 한다
/*
        isSpeak = intent.getBooleanExtra("isSpeak",true);

        if(isSpeak == false) {
            tts.shutdown();
        }*/
    }


    public void init_Service(){



        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {

            String NOTIFICATION_CHANNEL_ID = "com.example.kdcommunicator_serviceversion";
            String channelName = "My Background Service";
            NotificationChannel chan = new NotificationChannel(
                    NOTIFICATION_CHANNEL_ID,
                    channelName,
                    NotificationManager.IMPORTANCE_HIGH
            );
            chan.setLockscreenVisibility(Notification.VISIBILITY_PRIVATE);
            //Context.NOTIFICATION_SERVICE);
            ((NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE))
                    .createNotificationChannel(chan);

            NotificationCompat.Builder notificationBuilder = new NotificationCompat.Builder(this, NOTIFICATION_CHANNEL_ID);

            Notification notification =  notificationBuilder.setOngoing(true)
                    .setContentTitle("KD통신 실행중")
                    .setContentText("KD통신 실행 개시")
                    .setSmallIcon(R.drawable.ic_android_ex)
                    .setCategory(Notification.CATEGORY_SERVICE)
                    .setPriority(NotificationManager.IMPORTANCE_HIGH)
                    .build();
            startForeground(1, notification);
        }
        else {
            Intent notificationIntent = new Intent(this, MainActivity.class);
            PendingIntent pendingIntent =
                    PendingIntent.getActivity(this, 0, notificationIntent, 0);

            Log.d("gd", "entering");


            Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                    .setContentTitle("뉴KD통신 실행중")
                    .setContentText("뉴KD통신 실행 개시!")
                    .setSmallIcon(R.drawable.ic_android_ex)
                    .setContentIntent(pendingIntent)
                    .build();

            startForeground(1, notification);

        }
    }


    //최초의 페어링 시도이자 트리거
    public void bluetooth_Pairing(){
        bluetoothAdapter = BluetoothAdapter.getDefaultAdapter();

        Log.d("gd","start Pairing !!");

        sendBtConnectingStatus(STATE_CONNECTING);

        isBTclientRunning = true;
        clientClass = new BT_ClientThread(uuid,this.bluetoothAdapter,koda,this.handler);
        clientClass.start();


    }


    public void adapt_BroadCastReceiver(){

        IntentFilter filter = new IntentFilter();
        filter.addAction(BluetoothDevice.ACTION_ACL_CONNECTED);
        filter.addAction(BluetoothDevice.ACTION_ACL_DISCONNECTED);


        registerReceiver(btReceiver, filter);
        Log.d("gd","broadcast registed" );
    }


    @Override
    public void onDestroy(){


        /*
        블루투스 메인스레드
        웹 스레드
        에게 인터럽트를 먹여 종료시킨다

        브로드케스트 리시버를 Unregist하고
        tts를 shutdown 한다.
         */
        sendNotification("정상 종료되었습니다 ",NONE);


        if(sendReceive !=null) {
            if(sendReceive.isAlive()) {
                Log.d("gd","sendreceive is alive..so kill it");
                sendReceive.interrupt();
                sendReceive.close_stream();
            }
        }


        if(webThread != null) {
            webThread.stopRunning(); //인터럽트를 걸어서 스레드를 종료시킨다.
            webThread.getSocket().disconnect();
        }
        stopService(intent);

        if(btReceiver != null) {
            try { //브로드캐스트 리시버를 해제한다.
                Log.d("gd","unregistering bt broadcast receiver, handler");
                unregisterReceiver(btReceiver);
                btReceiver = null;
                mainHandler = null;
                handler = null;

            }catch (IllegalArgumentException e){
                Log.d("gd","illegal unregistering bt receiver catch");
                e.printStackTrace();
                btReceiver=null;
            }
        }
/*
        if(tts != null) {
            tts.stop();
            tts.shutdown();
            tts = null;
        }*/
        super.onDestroy();
    }

    //////////////////////////////////////////////////////////////////// BroadCast Receiver for BT reconnection /////////////////////////////////////////////////////////////////////////////////////////

    public void create_btReceiver() {

        btReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                final String action = intent.getAction();
                Log.d("gd", "get broadcast action :" + action);
                switch (action) {

                    case BluetoothDevice.ACTION_ACL_CONNECTED:
                        break;

                    case BluetoothDevice.ACTION_ACL_DISCONNECTED: //블루투스 연결 해제를 감지했을 때 (연결이 기본적으로 성공했다가 - 끊어지는 경우이다) :
                            btConnected = false;
                            try {
                                sendReceive.interrupt(); // 익셉션이 일어날 수 있으므로, sendReceive 중지
                            }catch (Exception e){
                                e.printStackTrace();
                            }

                            Log.d("gd", "finding koda device. Try connecting every 10 sec..");
                            sendBtConnectingStatus(STATE_CONNECTION_FAILED);
                            tts.speak("기기와의 연결이 끊어졌습니다. 지금부터 자동 연결 시도합니다",TextToSpeech.QUEUE_ADD,null);


                            new Handler().postDelayed(new Runnable() {
                                @Override
                                public void run() {
                                    sendBtConnectingStatus(STATE_CONNECTING);
                                    if(isBTclientRunning == false && btConnected == false) { //도중에 btConnected가 true가 될 수도 있어
                                        //이곳은 본래 강력한 mutex 처리가 필요하나, 일단은 boolean으로 처리하였다. 논리적으로는 clientClass는 단 하나만 존재할 필요가 있다
                                        isBTclientRunning = true;
                                        clientClass = new BT_ClientThread(uuid, bluetoothAdapter, koda, handler);
                                        clientClass.start();
                                    }
                                    else{
                                        Log.d("gd","it blocked by isBtClientRunning !");
                                    }

                                }
                            }, 10000);// 10초 마다 연결 시도
                            //이후의 재접속 루프는 connection_failed 핸들러에서 처리하고, 최초의 재접속만 여기서 처리하는것이다.



                        break;

                }


            }
        };
    }

    ////////////////////////////////// ///////////////////////////////////// Handler ///////////////////////////////// ///////////////////////////////////////



    //스레드 - 서비스 간 통신을 위해 핸들러가 필요
    //이 핸들러는 블루투스 수신시 / 웹 메세지 수신시의  콜백으로서 작동한다
    Handler handler = new Handler(new Handler.Callback() {

        @Override
        public boolean handleMessage(Message msg) {
            Message main_msg = new Message();
            switch (msg.what) {
                //connecting 플래그는 현재 사용하고 있지 않음
                /*
                case STATE_CONNECTING:
                    Log.d("gd","now connecting");
                    break;

                 */

                case STATE_CONNECTED:

                    isBTclientRunning = false; //만약 필요하다면 (재접속을 위해서) 클라이언트는 이제 다시 새로이 생성될 수 있다
                    btConnected = true; //당연히 bt connected 된다

                    if(onceBtConnected == false){ //이것이 최초 연결인 경우이다. 최초로 블루투스 연결에 성공했다면, 웹 연결을 개시한다
                            if(webThread == null) {

                                Log.d("gd", "webthread begin by" + url );
                                webThread = new WebThread(url, 2 , handler, receiverChannel, deviceNumber);
                                webThread.start(); //web connect & add listener

                            }
                            onceBtConnected = true;
                    }

                    sendReceive = new BT_MainThread( clientClass.getBTsocket() , handler); //클라이언트가 서버와의 connect 성공으로 리턴받은 구체적인 socket값을 메인스레드에 넣어준다. 이로서 메인스레드는 본격적은 블루투스 송수신이 가능해진다
                    sendReceive.start(); // 블루투스 수신 대기 및 인터럽트되면 깔끔하게 종료된다
                    sendBtConnectingStatus(STATE_CONNECTED);

                    sendNotification("블루투스 연결 완료!",DINGDONG);
                    tts.speak("연결되었습니다.",TextToSpeech.QUEUE_ADD,null);
                    if(role == RECEIVER) {
                        webThread.getSocket().emit("send_enter", deviceNumber);
                    }

                    break;

                case STATE_CONNECTION_FAILED: // 이것은 ACL_DISCONNECTION과는 다른 케이스이다 (acl_disconnection: 연결 되었다가 끊어짐 //connection_failed: BT클라이언트 스레드가 연결 시도했으나 실패)
                    //보통 브로드캐스트 리시버 ACL_DISCONNECTED 가 재연결 트리거를 걸고, 그 이후로의 블루투스 연결 실패는 여기서 처리한다
                    //다시 말해 ACL_DISCONNECTED가 연결 끊김을 인지하고 다시 BT 연결 시도하고, 이쪽에서 될 때까지 반복하는 역할을 한다
                    isBTclientRunning = false; //connect 성공과 마찬가지로 현재 존재하고있던 클라이언트스레드가 아무튼 run을 다 했으므로 새로운 run을 위한 개체를 생성하는 것을 허용한다
                    sendBtConnectingStatus(STATE_CONNECTION_FAILED);

                       sendNotification("재 연결중!",NONE);
                       new Handler().postDelayed(new Runnable() {
                           @Override
                           public void run() {
                               sendBtConnectingStatus(STATE_CONNECTING);
                               if(isBTclientRunning == false && btConnected == false) { //이렇게 플래그를 달아주면, 하나의 bt클라이언트가 연결 결과를 내기 전까지는 다른 bt클라이언트가 생성될 가능성이 현저히 적다.

                                   isBTclientRunning = true;
                                   clientClass = new BT_ClientThread(uuid, bluetoothAdapter, koda, handler);
                                   clientClass.start(); //얘가 연결시도를 끝내면 핸들러에게 failed 혹은 connected를 보내고, 핸들러는 그것을 수신하자 마자 isBTclientRunning을 false로 하는 것으로 또다시 clientThread를 생성할 권한을 연다.
                               }
                               else{
                                   Log.d("gd","making client is blocked by flag - isBTclientRunning");
                               }

                           }
                       }, 10000);// 10초 정도 딜레이를 준 후 시작


                    break; //////////////////////////// failed //////////////////////////////////////

                case STATE_BT_MESSAGE_RECEIVED:
                    //이쪽은 블루투스 메세지에 대한 핸들링이다. 블루투스 메세지를 받으면, 그것을 서버에 올린다

                    String btMessage = msg.obj.toString();
                    Log.d("gd","btMessage handled, message: " + btMessage);

                    webThread.btMessage_Callback(btMessage); //bt메세지를 읽어서 그것을 서버에 올려주는 역할을 하는 메소드이다


                    break; // BT_MESSAGE RECEIVED에 대한 케이스 브레이크


                case STATE_WEB_MESSAGE_RECEIVED:
                        //WEB 메세지에 대한 처리. 기본적으로 그것을 블루투스로 전달하는데, 여러가지 인터페이스도 추가로 제공한다(tts로다가)

                    String webMessage = msg.obj.toString();
                    Log.d("gd", "handler received from web: " + webMessage);

                    if(webMessage.contains("sigready")){
                        sendReceive.write("F".getBytes());
                    }

                    else if(webMessage.contains("linkauto")){
                        sendReceive.write("P".getBytes());
                    }

                    else if(webMessage.equals("auto")){
                        sendReceive.write("A".getBytes());
                    }



                    else if(webMessage.contains("set_level")){
                        int level = msg.arg1;
                        switch (level){
                            case 1:
                                sendReceive.write("e".getBytes());
                                break;
                            case 2:
                                sendReceive.write("f".getBytes());
                                break;
                            case 3:
                                sendReceive.write("g".getBytes());
                                break;
                            case 4:
                                sendReceive.write("h".getBytes());
                                break;
                            case 5:
                                sendReceive.write("i".getBytes());
                                break;
                        }

                    }

                    else if(webMessage.contains("set_mode")){
                        int mode = msg.arg1;
                        switch (mode){
                            case 1:
                                sendReceive.write("a".getBytes());
                                break;
                            case 2:
                                sendReceive.write("b".getBytes());
                                break;
                            case 3:
                                sendReceive.write("c".getBytes());
                                break;
                        }

                    }
                    else { // "COMMAND"
                        String cmd = (String) msg.obj;
                             Log.d("gd","cmd is " + cmd + "pass raw cmd to KD500 BT ");
                            sendReceive.write(cmd.getBytes());

                    }


                        break;
            }
            return true;
        }
    }); //핸들러




    //"YTN"이 서버에서 왔을 때 , 이것은 220 입장에서 결과값을 받았다는 것이고, 그것을 음성 안내하는 메소드들이다
    //음성안내 - veryFast 버전이다
    public void speakImpulse_veryfast(String impulse){

        //Y T N x P y I 0 z , 회선x, 상y, 기기번호z
        //0 1 2 3 4 5 6 7 8

        if(impulse.contains("@") == false) {
            char lineNumber = impulse.charAt(3);
            char phase = impulse.charAt(5);
            String devNum = impulse.substring(7, impulse.length());
            int devNumber = Integer.parseInt(devNum);

            //tts.speak("결과 ", TextToSpeech.QUEUE_FLUSH, null);

            tts.speak(phase + "!", TextToSpeech.QUEUE_ADD, null);
            tts.speak(lineNumber + "!", TextToSpeech.QUEUE_ADD, null);
           // if(receiverChannel >1) {
                tts.speak("변압기" + devNumber + "번", TextToSpeech.QUEUE_ADD, null);
             //   }
        }

        else{
            tts.speak("이륙공에서 측정을 실패했습니다 ", TextToSpeech.QUEUE_ADD, null);
        }
    }

    //음성안내- Fast 버전이다
    public void speakImpulse_fast(String impulse){

        //Y T N x P y I 0 z , 회선x, 상y, 기기번호z
        //0 1 2 3 4 5 6 7 8

        if(impulse.contains("@") == false) {
            char lineNumber = impulse.charAt(3);
            char phase = impulse.charAt(5);
            String devNum = impulse.substring(7, impulse.length());
            int devNumber = Integer.parseInt(devNum);

            //tts.speak("결과 ", TextToSpeech.QUEUE_FLUSH, null);
            tts.speak("회선 " + lineNumber + " 번!", TextToSpeech.QUEUE_ADD, null);
            tts.speak("상은! " + phase + " 상", TextToSpeech.QUEUE_ADD, null);
           // if(receiverChannel >1)
             tts.speak("변압기! " + devNumber + "번입니다", TextToSpeech.QUEUE_ADD, null);
        }

        else{
            tts.speak("이륙공에서 측정을 실패했습니다 ", TextToSpeech.QUEUE_ADD, null);
        }
    }


    //음성안내 - 상세 버전이다
    public void speakImpulse_precise(String impulse){


        //Y T N x P y I 0 z , 회선x, 상y, 기기번호z
        //0 1 2 3 4 5 6 7 8

        if(impulse.contains("@") == false) {
            char lineNumber = impulse.charAt(3);
            char phase = impulse.charAt(5);
            String devNum = impulse.substring(7, impulse.length());
            int devNumber = Integer.parseInt(devNum);

            //tts.speak("결과 ", TextToSpeech.QUEUE_ADD, null);
            tts.speak("회선번호는! " + lineNumber + " 번", TextToSpeech.QUEUE_ADD, null);
            tts.speak("상은! " + phase + " 상 ", TextToSpeech.QUEUE_ADD, null);
            tts.speak("변압기 번호는! " + devNumber + "번입니다", TextToSpeech.QUEUE_ADD, null);
        }

        else{
            tts.speak("이륙공에서 측정을 실패했습니다 ", TextToSpeech.QUEUE_ADD, null);
        }
    }


    //안드로이드 Notification을 하는 메소드이다. String 파라메터가 표기할 출력 문장이고, 사운드타입 딩동 , 땡, 0(무음) 이다.
    private void sendNotification(String messageBody,int soundType) {

        NotificationCompat.Builder notificationBuilder;


        Intent intent = new Intent(this, MainActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0 /* Request code */, intent, PendingIntent.FLAG_ONE_SHOT);

        String channelId = "fcm_default_channel";//getString(R.string.default_notification_channel_id);
        Uri defaultSoundUri= RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM);
       // Uri customSound = Uri.parse("android.resource://"+  this.getPackageName() + "/" + R.raw.one);
        //Uri customSound = Uri.parse(Environment.getExternalStorageDirectory().getPath()+ )
        // Log.d("gd","uri : " + this.getPackageName()+"/"+R.raw.dingdong);

        notificationBuilder =
                new NotificationCompat.Builder(this, channelId)
                        .setSmallIcon(R.mipmap.ic_launcher)//drawable.splash)
                        .setContentTitle("KD통신")
                        .setContentText(messageBody)
                        .setAutoCancel(true)
                       // .setSound(customSound)
                        .setContentIntent(pendingIntent);

        NotificationManager notificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);



        // Since android Oreo notification channel is needed.
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {


            notificationBuilder.setDefaults(Notification.DEFAULT_VIBRATE);

            AudioAttributes attrs = new AudioAttributes.Builder()
            .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION)
            .setUsage(AudioAttributes.USAGE_NOTIFICATION)
            .build();

            NotificationChannel channel = new NotificationChannel(channelId,"Channel human readable title", NotificationManager.IMPORTANCE_HIGH);
            //channel.setSound(customSound, attrs );

            notificationManager.createNotificationChannel(channel);

        }

            notificationManager.notify(1 /* ID of notification */, notificationBuilder.build());

        if(soundType == DINGDONG) {
            soundpool.play(dingdong, 1f, 1f, 0, 0, 1f);
        }
        else if(soundType == DDANG){

            soundpool.play(ddang, 1f, 1f, 0, 0, 1f);

        }
        // none = no sound


    }


    //////////////////////////////// developing bind service ///////////////////

    private final IBinder csIBinder = new LocalBinder();
    public Handler mainHandler;

    public class LocalBinder extends Binder{

        public CommunicationService getInstance(){
            return CommunicationService.this;
            /*클래스가 중첩되는 경우( inner  / outer )에 Class. this는 가장 안쪽 scope의 class객체를 가리킵니다.
            지금같은경우는 그냥 communicationService의 객체 그 자체임*/
        }

    }


    public void dontSpeak(){
        /*
        Log.d("gd","dont speak");
        isSpeak = false;
        tts.shutdown();
        */
    }

    public void speak(){
        Log.d("gd","ok speak");
        isSpeak = true;
        tts = new TextToSpeech(this, new TextToSpeech.OnInitListener() {
            @Override
            public void onInit(int status) {
                if (status != ERROR) {
                    tts.setLanguage(Locale.KOREAN);
                    Log.d("gd", "tts set");
                }
            }
        }); //일단 만들고 셧다운을 시킴. 만드는것은 필수로 한다


    }


    public void sendBtConnectingStatus(int what){

        Message main_msg = new Message();
        main_msg.obtain();
        main_msg.what = what;
        try {
            mainHandler.sendMessage(main_msg);
        }
        catch (NullPointerException n){
            n.printStackTrace();
        }
    }

    public void setDeviceNumber_duringRunning(String deviceNumber){
        this.deviceNumber = deviceNumber;
        webThread.getSocket().emit("join",role,deviceNumber); //role == receiver

    }

    public void setVoiceSpeed(int spd){
        this.voiceSpeed = spd;
        if(voiceSpeed == SILENCE){
            dontSpeak();
        }

        if(isSpeak == false && voiceSpeed != SILENCE){
            isSpeak = true;
            speak();
        }
    }

    @Override
    public IBinder onBind(Intent intent){
        return csIBinder;
    }

    public void getMainHandler(Handler m_handler){
        this.mainHandler =  m_handler;
    }




}
