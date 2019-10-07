---
title: NIO之SocketChannel小实践-聊天程序mini版
date: 2019-09-20 18:41:28
tags:
- nio
- socket
categoreis: nio
---

想起之前研究java的nio时候，发现自己大学时用的socket在nio上有一个实现：**SocketChannel**，所以很感兴趣的一边学一边尝试，写了个很简单的几百行小程序，模仿现实中的（一server对多client）聊天，话不多说直接上代码和实现效果,可以看到，**得力于nio，server程序一个线程就可以同时处理多个client的socket连接**

- Server.java

  ```java
  import java.io.IOException;
  import java.net.InetSocketAddress;
  import java.nio.ByteBuffer;
  import java.nio.channels.SelectionKey;
  import java.nio.channels.Selector;
  import java.nio.channels.ServerSocketChannel;
  import java.nio.channels.SocketChannel;
  import java.time.LocalDateTime;
  import java.time.format.DateTimeFormatter;
  import java.util.Iterator;
  import java.util.Scanner;
  
  public class Server implements Runnable{
  	private Selector selector;
      private ByteBuffer buffer = ByteBuffer.allocate(1024); //?
      
      public Server(int port){
          try {
              selector = Selector.open();
              ServerSocketChannel ssc = ServerSocketChannel.open();
              //设置服务器为非阻塞方式
              ssc.configureBlocking(false);
              ssc.bind(new InetSocketAddress(port));
              //把服务器通道注册到多路复用选择器上，并监听阻塞状态
              ssc.register(selector,SelectionKey.OP_ACCEPT);
              System.out.println("Server start whit port : "+port);
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
      
      public void run(){
          while(true){
              try {
                  //会这这里处理事件，也是阻塞的，事件包括客户端连接，客户端发送数据到来,以及客户端断开连接等等
                  //若没有事件发生，也会阻塞
                  selector.select();
                  //System.out.println("阻塞在这");
                  //返回所有已经注册到多路复用选择器的通道的SelectionKey
                  Iterator<SelectionKey> keys = selector.selectedKeys().iterator();
                  //遍历keys
                  while(keys.hasNext()){
                      SelectionKey key = keys.next();
                      //下一个key，就像数组访问的i++
                      keys.remove();
                      if(key.isValid()){           //判断key是否有效
                          if(key.isAcceptable()){  //请求连接事件
                              accept(key);         //处理新客户的连接
                          }
                          if(key.isReadable()){    //有数据到来
                              read(key);
                          }
                      }
                  }
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
      }
      
      /**
       * 处理客户端连接
       * 服务器为每个客户端生成一个Channel
       * Channel与客户端对接
       * Channel绑定到Selector上
       * **/
      private void accept(SelectionKey key){
          try {
              //获取之前注册的SocketChannel通道
              ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
              //执行阻塞方法，Channel和客户端对接
              SocketChannel sc = ssc.accept();
              //设置模式为非阻塞
              sc.configureBlocking(false);
              System.out.println("client connect,id is->"+sc.socket().getRemoteSocketAddress());
              //注册监听读key
              sc.register(selector,SelectionKey.OP_READ);
          }catch(Exception e){
              e.printStackTrace();
          }
      }
      
      private void read(SelectionKey key){
  
          try {
              //清空缓冲区的旧数据
              buffer.clear();
              SocketChannel sc = (SocketChannel) key.channel();
              int count = sc.read(buffer);
              if(count == -1){
                  key.channel().close();
                  key.cancel();
                  return;
              }
              //读取到了数据，将buffer的position复位到0
              buffer.flip();
              byte[] bytes = new byte[buffer.remaining()];
              buffer.get(bytes);
              String body = new String(bytes).trim();
              System.out.println(sc.socket().getRemoteSocketAddress()+" client:"+body);
              
              ByteBuffer outBuffer=ByteBuffer.allocate(1024);
              //阻塞 模拟等待用户输入信息
              outBuffer.put(writeSomething());
              outBuffer.flip();
              //继续注册监听读key
              sc.register(selector,SelectionKey.OP_READ);
              //向client写入信息
              sc.write(outBuffer);
              outBuffer.clear();
  
              
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
      
      public static void main( String[] args )
      {
      	//启动线程开始监听
          new Thread(new Server(8379)).start();
      }
      
      //拼写输入内容
      public static byte[] writeSomething() throws IOException{
  		DateTimeFormatter format= DateTimeFormatter.ofPattern("yyyy-mm-dd HH:mm:ss");
  		String time=format.format(LocalDateTime.now())+"-> ";
  		System.out.print("请输入：");
  		Scanner sc=new Scanner(System.in);
  		String str=sc.nextLine();
          return (time+str).getBytes();
  	}
  	
  	 
  }
  ```

- Client.java

  ```java
  import java.io.IOException;
  import java.net.InetSocketAddress;
  import java.nio.ByteBuffer;
  import java.nio.channels.SelectionKey;
  import java.nio.channels.Selector;
  import java.nio.channels.SocketChannel;
  import java.time.LocalDateTime;
  import java.time.format.DateTimeFormatter;
  import java.util.Iterator;
  import java.util.Scanner;
  
  public class Client {
  	private static final String host = "127.0.0.1";
      private static final int port = 8379;
      private Selector selector;
  	public static void main(String[] args){
          InetSocketAddress address = new InetSocketAddress(host,port);
          ByteBuffer writeBytes=ByteBuffer.allocate(1024);
          Client client=new Client();
          SocketChannel sc =null;
          try{
              sc = SocketChannel.open();
              sc.configureBlocking(false);
              //第一次向selector注册链接事件
              client.selector = Selector.open();
              sc.register(client.selector, SelectionKey.OP_CONNECT);
              //链接服务端，触发OP_CONNECT事件
              sc.connect(address);
              while(true){
              	//监听所有key，没有key时间完成时会阻塞
              	int events = client.selector.select();
              	if(events>0){
              		Iterator<SelectionKey> selectionKeys = client.selector.selectedKeys().iterator();
              			while(selectionKeys.hasNext()){
              				 SelectionKey selectionKey = selectionKeys.next();
                               selectionKeys.remove();
                               SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                               //连接事件
                               if (selectionKey.isConnectable()) {
                              	 //获取key的socketChannel
                                   if (socketChannel.isConnectionPending()) {
                                  	 //如果已经连接成功，则将状态改为链接完成，开始通讯
                                       socketChannel.finishConnect();
                                   }
                                   System.out.println("server connect success");
                                   //向selector注册read事件，当server write完毕触发
  //                                 socketChannel.register(client.selector, SelectionKey.OP_READ);
  //                                 
  //                                 writeBytes.put(writeSomething());
  //                                 writeBytes.flip();
  //                                 //向server写入信息
  //                                 socketChannel.write(writeBytes);
  //                                 writeBytes.clear();
                                   //socketChannel.close();
                               } else if (selectionKey.isReadable()) {
                                   ByteBuffer buffer = ByteBuffer.allocate(1024);
                                   socketChannel.read(buffer);
                                   buffer.flip();
                                   System.out.println("server："+new String(buffer.array()).trim());
                                   
                                   
                                   //休眠一秒继续发送消息
                                   Thread.sleep(1000);
                                   
                                   
                                   //sc = SocketChannel.open();
                                   //继续向selector注册read事件
  //                                 socketChannel.register(client.selector, SelectionKey.OP_READ);
  //                                 writeBytes.put(writeSomething());
  //                                 writeBytes.flip();
  //                                 //向server写入信息
  //                                 socketChannel.write(writeBytes);
  //                                 writeBytes.clear();
                                   //sc.connect(address);
                               }
                               
                               //继续向selector注册read事件
                               socketChannel.register(client.selector, SelectionKey.OP_READ);
                               byte[] ws=writeSomething();
                               writeBytes.put(ws);
                               writeBytes.flip();
                               //向server写入信息
                               socketChannel.write(writeBytes);
                               writeBytes.clear();
              			}
              	}
              }
          }catch(IOException e){
              e.printStackTrace();
          } catch (InterruptedException e) {
  			// TODO Auto-generated catch block
  			e.printStackTrace();
  		}finally {
              if(sc!=null){
                  try {
                      sc.close();
                  }catch (IOException e){
                      e.printStackTrace();
                  }
              }
          }
      }
  	
  	public static byte[] writeSomething() throws IOException{
  		DateTimeFormatter format= DateTimeFormatter.ofPattern("yyyy-mm-dd HH:mm:ss");
  		String time=format.format(LocalDateTime.now())+"-> ";
  		System.out.print("请输入：");
  		Scanner sc=new Scanner(System.in);
  		String str=sc.nextLine();
          return (time+str).getBytes();
  	}
  }
  
  ```

- 实现效果图

  ![效果图](/intro/0027.png)