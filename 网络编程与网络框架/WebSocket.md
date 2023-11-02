# WebSocket

WebSocket是一种在单个TCP连接上进行全双工通讯的协议，它使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在WebSocket API中，浏览器和服务器只需要完成一次握 手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

## 意义

为什么需要WebSocket，因为 HTTP 协议有一个缺陷：通信只能由客户端发起。而WebSocket可以实现双向通信。一般来说WebSocket是用来实现**双工通信的长连接**的。HTTP想要达到 这种效果，一般会通过**轮询或者long poll**来实现，这样比较占用资源且非常被动

## 请求响应

客户端请求

```
GET / HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Host: example.com
Origin: http://example.com
Sec-WebSocket-Key: sN9cRrP/n9NdMgdcy2VJFQ==
Sec-WebSocket-Version: 13
```

服务器响应

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: fFBooB7FAkLlXgRSz0BT3v4hq5s=
Sec-WebSocket-Location: ws://example.com/
```



###### 基于Okhttp实现WebSocket

```
var mClient: OkHttpClient = OkHttpClient()
val request = Request.Builder()
            .url("ws://www.jotalk.tech:81")
            .header("Upgrade", "websocket")
            .header("Connection", "Upgrade")
            .header("Sec-WebSocket-Key", "http://www.jotalk.tech")
            .header("Sec-WebSocket-Version", "13")
            .build()
            
mClient.newWebSocket(request,object:WebSocketListener(){

            override fun onOpen(webSocket: WebSocket?, response: Response?) {
                super.onOpen(webSocket, response)
                //成功连接
            }

            override fun onFailure(webSocket: WebSocket?, t: Throwable?, response: Response?) {
                super.onFailure(webSocket, t, response)
            }

            override fun onClosing(webSocket: WebSocket?, code: Int, reason: String?) {
                super.onClosing(webSocket, code, reason)
            }

            override fun onMessage(webSocket: WebSocket?, text: String?) {
                super.onMessage(webSocket, text)
            }

            override fun onMessage(webSocket: WebSocket?, bytes: ByteString?) {
                super.onMessage(webSocket, bytes)
            }

            override fun onClosed(webSocket: WebSocket?, code: Int, reason: String?) {
                super.onClosed(webSocket, code, reason)
                //发生任何错误都会回调改方法
            }
        })

mClient.dispatcher().executorService().shutdown()
```





# SocketIO

WebSocket是HTML5的一种新通信协议，它实现了浏览器与服务器之间的双向通讯。而Socket.IO是一个完全由JavaScript实现、基于Node.js、支持WebSocket的协议用于实时通信、跨平台的开源框架

官方提供的android版本的socket-IO框架

[socket.io-client-java](https://github.com/socketio/socket.io-client-java)



## 用法

###### 依赖

```
    implementation('io.socket:socket.io-client:0.8.3') {
        exclude group: 'org.json', module: 'json'
    }
```

###### 初始化

```
    {
        try {
            mSocket = IO.socket(SOCKETIO_URL);
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    }
```

###### 连接

```
mSocket.connect();
mSocket.disconnect();
```

###### 监听

我们可以通过 mSocket.on(“KEY”, LISTENER) 方法来监听服务器返回的事件。其中 “KEY” 为事件的键， LISTENER 为监听器，监听器的类型为 Emitter.Listener() 

```
mSocket.on(Socket.EVENT_CONNECT, new Emitter.Listener() {
            @Override
            public void call(Object... args) {
                 //call方法是在子线程中执行的，需要切换到主线程
                getActivity().runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                       
                    }
                });
            }
        });
```

###### 发送

```
mSocket.emit("KEY", mString);
```

默认以polling轮询的方式，可以改成websocket

只需要配置

```
 val opts = IO.Options();
 opts.transports = arrayOf(WebSocket.NAME)
 mSocket = IO.socket("http://47.111.6.162:3000/",opts)
```

