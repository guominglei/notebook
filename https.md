# HTTPS
## 链接建立过程

    * 1. client hello
        客户端向服务器发送 client hello消息。
        信息包括 random1, 客户端支持的加密版本和SSL版本信息。
    * 2. server hello
        服务器向client发送 server hello. 
        信息包括 random2, 从客户端支持的加密版本。选择一个。
    * 3. certificate
        服务器把服务器公钥证书文件发送给客户端。客户端据此获取到公钥
    * 4. server hello Done
        服务器通知客户端。server hello 阶段结束。
    * 5. client key exchange
        客户端 生成 random3， 并根据服务器公钥对random3进行加密。发送出去。
        服务器收到信息后，会用秘钥把信息解析出来。
        这时候，客户端和服务器就都知道 random1 random2 random3 的内容了。
        以后通信，就用这三个random 根据协商好的加密算法。生成一个对称秘钥。
        正常通信后，使用这个秘钥进行数据加密
    * 6. change cipher spec
        客户端通知服务器正常通信的秘钥确立。以后用秘钥加密解密。 
    * 7. Encrypted handshake message (client)
        客户端使用对称秘钥加密一个条数据。
        服务器收到后，用秘钥解析出来。确定对方加密，解密过程ok
    * 8. change cipher spec
        服务器向客户声明，以后用秘钥通信。
    * 9. Encrpted Handshake message (client)
        服务器向客户端发送一条用秘钥加密的数据。客户端解析出来后。
        确定对方加密，解密过程ok
    * 10. 通道建立成功。开始正常通信。

