package com.example.kdcommunicator_impulseversion;

import android.os.Handler;
import android.os.Message;
import android.util.Log;

import com.github.nkzawa.emitter.Emitter;
import com.github.nkzawa.socketio.client.IO;
import com.github.nkzawa.socketio.client.Socket;

import org.json.JSONException;
import org.json.JSONObject;

import java.net.URISyntaxException;

public class WebThread  extends  Thread{

    public Socket socket;
    public Handler handler;
    private int receiverChannel; //유효하다
    private String deviceNumber;


    final private int SENDER=1;
    final private int RECEIVER=1;
    private int role=SENDER; //유효하다

    private boolean correspond = true;
    

    private final int STATE_WEB_MESSAGE_RECEIVED = 5; //CommunicationService에게 핸들러로 보내는 플래그값


    //웹스레드의 생성은 블루투스의 최초 연결이 성공했을 때 진행된다.
    //즉 유효한 KD 기기가 없는 상태에서 연결을 개시했을 때에는 서버와의 연결을 유보하는 것이다.
    public  WebThread(String get_url, int input_role, android.os.Handler handler,int receiverChannel, String deviceNumber){

        Log.d("gd","webThread is created!!");
        this.role = input_role; //메인 -> 서비스 -> 스레드로 이어지는 설정값들을 이어받는다. 역할(220/260)
        this.receiverChannel = receiverChannel; //260 개수
        this.deviceNumber = deviceNumber; //260 장치번호
        try {
            socket = IO.socket(get_url); //URL 은 웹 도메인 서버 주소이다
            socket.connect();
            //socket.open();
            Log.d("gd","join complete by : " + get_url);
            socket.emit("join",2,"01"); //220인 경우 DeviceNumber는 그냥 01이다.

        } catch (URISyntaxException e) {
            Log.d("gd", "web server enter failed");
            e.printStackTrace();
        }

        webListening(); //리스너를 regist한다.
        this.handler = handler; //communicationService의 핸들러를 등록하는 것으로, 웹 메시지를 받을 때 이를 서비스에 넘겨주는 역할을 한다.
        Log.d("gd","web thread created");


    }


    public void stopRunning(){
        Log.d("gd","stop webThread running ");

        this.interrupt(); //인터럽트를 맞으면 메인의 슬립이 풀려나고 익셉션을 뿌리는데 거기에 return을 시키는 것으로 메인의 루프를 종료시킨다.
        //서비스의 onDestroy 에서 호출된다. 그냥 webThread.interrupt()로 하면 되지만 로그메세지라던가 또 다른 처리가 있을때를 대비하여 별도의 메소드화 했다...
    }


    @Override
    public void run(){
        Log.d("gd","webThread run start");

        while(true){

                if( isInterrupted()   ){ //슬립이 아닌 일반 실행 도중에 인터럽트를 받았을 때를 대비하여
                    Log.d("gd","webThread interrupt received, it will die.");
                    break;
                }



                try {
                    Thread.sleep(60000); //1분마다
                }
                catch (InterruptedException e){ //스레드 슬립 도중 interrupt를 받으면 슬립에서 강제로 일어나며 InterruptedException이 발생한다
                    //이 익셉션의 핸들러에서 break 해줌으로서 스레드는 무한 핑-퐁 루프에서 탈출하게 된다.
                    Log.d("gd","webthread sleep break");
                    e.printStackTrace();
                    break;
                     //it means getting interrupt from sleep !
                }
        }

        Log.d("gd","return the webThread");
        this.socket.off(); // 등록된 리스너를 초기화한다
        this.socket.disconnect();


        return;
    }




    public void webListening(){
        Log.d("gd", "web listener registed. ");

        socket.on("setOption", new Emitter.Listener() {
            //센더220 리시버260 모두에게 전송될 수 있는 내용이다.
            //disconnect를 서버에서 감지 했을때, 접속된 모든 유저에게 (tryRepair,역할)을 뿌린다.
            //본인의 현재 역할이 tryRepair의 역할과 같다면, join을 시도한다.

            @Override
            public void call(Object... args) {

                JSONObject data = (JSONObject) args[0];

                try{
                    String command = data.getString("cmd"); //0 또는 1(sender) 또는 2(receiver)를 받는다.
                    Log.d("gd","got 'setoption event' from server, cmd : " + command);
                    //command를 bt로 넘겨주기만 하면 된다.
                    Message msg = Message.obtain();
                    msg.what = STATE_WEB_MESSAGE_RECEIVED;
                    msg.obj = command;

                    sendToServiceHandler(msg);

                }catch (JSONException e){
                    e.printStackTrace();
                }
            }
        });


        socket.on("tryRepair", new Emitter.Listener() {
            //센더220 리시버260 모두에게 전송될 수 있는 내용이다.
            //disconnect를 서버에서 감지 했을때, 접속된 모든 유저에게 (tryRepair,역할)을 뿌린다.
            //본인의 현재 역할이 tryRepair의 역할과 같다면, join을 시도한다.

           @Override
            public void call(Object... args) {

                JSONObject data = (JSONObject) args[0];
                try{
                    int missed_role = data.getInt("role"); //0 또는 1(sender) 또는 2(receiver)를 받는다.

                    if(role == missed_role || missed_role == 3 ){ //나 자신의 role이 disconnected 된 missed Role과 같다면 일단 join을 시도한다
                        Log.d("gd","try re-pair by role : " + role );
                        socket.emit("join",role,deviceNumber); //중복된 join인 경우 서버에서 알아서 걸러줄 것이고, 유효한 join인경우 재접속이 된다.
                    }

                }catch (JSONException e){
                    e.printStackTrace();
                }
            }
        });



        socket.on("sigready", new Emitter.Listener() {
            //센더220 휴대폰에게만 전송되는 내용이다
            //260 모바일이 ACL_Disconnect 되면 YTE를 서버에 올리고 220이 이것을 받아 블루투스로 전달하게 된다
            //'N'번 260의 블루투스 연결이 끊어졌을음 알린다.

            @Override
            public void call(Object... args) {
                JSONObject data = (JSONObject) args[0];
                try{

                    Log.d("gd","get sig ready from web server");
                    Message msg = Message.obtain();
                    msg.what = STATE_WEB_MESSAGE_RECEIVED;
                    msg.obj = "sigready";
                    //handler.sendMessage(msg);
                    sendToServiceHandler(msg);

                }catch (Exception e){
                    e.printStackTrace();
                }

            }
        });

        socket.on("set_level", new Emitter.Listener() {
            //센더220 휴대폰에게만 전송되는 내용이다
            //260 모바일이 ACL_Disconnect 되면 YTE를 서버에 올리고 220이 이것을 받아 블루투스로 전달하게 된다
            //'N'번 260의 블루투스 연결이 끊어졌을음 알린다.

            @Override
            public void call(Object... args) {
                JSONObject data = (JSONObject) args[0];
                try{
                    String lv;
                    lv = data.getString("level");
                    int level = Integer.parseInt(lv);

                    Log.d("gd","get 'set level' from web server. level : " + level);
                    Message msg = Message.obtain();
                    msg.what = STATE_WEB_MESSAGE_RECEIVED;
                    msg.arg1 = level;
                    msg.obj = "set_level";
                    //handler.sendMessage(msg);
                    sendToServiceHandler(msg);

                }catch (Exception e){
                    e.printStackTrace();
                }

            }
        });

        socket.on("set_mode", new Emitter.Listener() {
            //센더220 휴대폰에게만 전송되는 내용이다
            //260 모바일이 ACL_Disconnect 되면 YTE를 서버에 올리고 220이 이것을 받아 블루투스로 전달하게 된다
            //'N'번 260의 블루투스 연결이 끊어졌을음 알린다.

            @Override
            public void call(Object... args) {
                JSONObject data = (JSONObject) args[0];
                try{
                    String md;
                    md = data.getString("mode");
                    int mode = Integer.parseInt(md);

                    Log.d("gd","get 'set mode' from web server. mode : " + mode);
                    Message msg = Message.obtain();
                    msg.what = STATE_WEB_MESSAGE_RECEIVED;
                    msg.arg1 = mode;
                    msg.obj = "set_mode";
                    //handler.sendMessage(msg);
                    sendToServiceHandler(msg);

                }catch (Exception e){
                    e.printStackTrace();
                }

            }
        });


        socket.on("auto", new Emitter.Listener() {
            //센더220 휴대폰에게만 전송되는 내용이다
            //260 모바일이 ACL_Disconnect 되면 YTE를 서버에 올리고 220이 이것을 받아 블루투스로 전달하게 된다
            //'N'번 260의 블루투스 연결이 끊어졌을음 알린다.

            @Override
            public void call(Object... args) {
                Log.d("gd","receive auto");
                JSONObject data = (JSONObject) args[0];
                try{

                    Message msg = Message.obtain();
                    msg.what = STATE_WEB_MESSAGE_RECEIVED;
                    msg.obj = "auto";
                    //handler.sendMessage(msg);
                    sendToServiceHandler(msg);

                }catch (Exception e){
                    e.printStackTrace();
                }

            }
        });

        socket.on("linkauto", new Emitter.Listener() {
            //센더220 휴대폰에게만 전송되는 내용이다
            //260 모바일이 ACL_Disconnect 되면 YTE를 서버에 올리고 220이 이것을 받아 블루투스로 전달하게 된다
            //'N'번 260의 블루투스 연결이 끊어졌을음 알린다.

            @Override
            public void call(Object... args) {
                Log.d("gd","receive auto");
                JSONObject data = (JSONObject) args[0];
                try{

                    Message msg = Message.obtain();
                    msg.what = STATE_WEB_MESSAGE_RECEIVED;
                    msg.obj = "linkauto";
                    //handler.sendMessage(msg);
                    sendToServiceHandler(msg);

                }catch (Exception e){
                    e.printStackTrace();
                }

            }
        });


    } //onListening


    public void btMessage_Callback(String btMessage){

        Log.d("tmp_test","bt message on webThread : "+btMessage);

        if(btMessage.contains("TMP")){
            //TMP000
            int tempData =0;
            Log.d("tmp_test","receive tmp");

                String parseTemp = btMessage.substring(3);//3,4,5
                Log.d("tmp_test", "temperature is " + parseTemp);
                tempData = Integer.parseInt(parseTemp);
                this.socket.emit("temp", tempData);


        }
        else if(btMessage.equals("ACK")) {
            Log.d("gd","bt receive ack -> pass it to server");
            //220에서 SC를 받으면 서버에 올린다. 서버는 이를 모든 260(정확히는 220을 제외한 모든 유저)에게 보낸다
            Log.d("gd","send ack !");
            this.socket.emit("ack");


        }else if(btMessage.contains("GAS")){
            //GAS000F
            int gasData =0;
            Log.d("gas_test","receive gas");

            String parseTemp = btMessage.substring(3);//3,4,5
            Log.d("gas_test", "gas is " + parseTemp);
            gasData = Integer.parseInt(parseTemp);
            this.socket.emit("gas", gasData);

        }else if(btMessage.contains("CURR")){
            //CURR0F
            int currentPulse =0;
            Log.d("curr_test","receive  currPuse");

            String parseTemp = btMessage.substring(4);//3,4,5
            Log.d("curr_test", "curr is " + parseTemp);
            currentPulse = Integer.parseInt(parseTemp);
            this.socket.emit("currentPulse", currentPulse);

        }else if(btMessage.contains("HMD")){
            //HMD00F
            int hmPercent =0;
            Log.d("hmd_test","receive  hmd");

            String parseTemp = btMessage.substring(3);//3,4,5
            Log.d("hmd_test", "hmd is " + parseTemp);
            hmPercent = Integer.parseInt(parseTemp);
            this.socket.emit("humid", hmPercent);

        }else if(btMessage.contains("ENT")){
            //ENT0F
            int currentPulse =0;
            Log.d("curr_test","receive  currPuse");

            String parseTemp = btMessage.substring(3);//3,4,5
            Log.d("curr_test", "curr is " + parseTemp);
            currentPulse = Integer.parseInt(parseTemp);
            this.socket.emit("entrance", currentPulse);

        }

        else if(btMessage.equals("ON")) {
            Log.d("gd","bt receive ack -> pass it to server");
             Log.d("gd","send auto on !");
            this.socket.emit("auto_on");


        }


        else if(btMessage.equals("Of")) {
            Log.d("gd","bt receive ack -> pass it to server");
            Log.d("gd","send auto off !");
            this.socket.emit("auto_off");

        }

        else if(btMessage.equals("OK")) {
            Log.d("gd","send OK !");
            this.socket.emit("send_ok");

        }

        else if(btMessage.contains("X")){
            this.socket.emit("setOption",btMessage.toString());
            Log.d("gd","transmit by setoption");
        }



    }



    public Socket getSocket(){

        return socket;
    }


    //핸들러 보낼 때 널 포인트 익셉션 처리를 해주기 위해서이다.
    public void sendToServiceHandler(Message msg){

       try{ handler.sendMessage(msg); }
       catch (NullPointerException n){
           Log.d("gd","null error at servicehandler sending. here is webThread");
           n.printStackTrace();
       }

    }

}
