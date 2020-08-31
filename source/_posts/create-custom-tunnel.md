---
title: Shadowsocks 实现分析以及如何实现一个代理协议
date: 2020-08-27 16:07:43
tags:
  - 翻墙
  - 代理
  - Python
categories:
  - 翻墙
---



从使用 [GoAgent](https://www.wikiwand.com/zh/GoAgent) 翻墙开始，到后来用理稳定的 [Shadowsocks](http://shadowsocks.org/en/index.html)，SS 不太稳定后又换到 [V2Ray](https://v2ray.com/)（主要是 vmess 协议），一直在 「用」而不了解其实现，迫于现在翻墙有越来越难的趋势，还是了解一下比较好。

### 从 Shadowsocks 开始

首先看看 SS 是怎么实现的，先找了一个早期的版本，要求是功能较为完善，实现简单，目前的版本已经较为复杂，各平台的兼容代码也较多，最终选了一个 2014 年的版本，1.3.7，链接放这里 https://github.com/shadowsocks/shadowsocks/tree/9d3e2d717753ba9489fcd553cd444449fffb13ac

SS 的介绍中提到：

    Shadowsocks is a secure split proxy loosely based on SOCKS5.

SOCKS5 协议很常见，软件和系统的设置里经常看得到这个配置项，SOCKS5 的工作方式应该是软件或者系统里实现了将流量接管的功能，并在配置了 SOCKS5 代理的时候，将流量转发到 SOCKS5 的 server 端，server 收到流量代理请求后再把流量转发出去。

那么来看一下 SOCKS5 是怎么实现的？SOCKS5 的定义在 [https://tools.ietf.org/html/rfc1928](https://tools.ietf.org/html/rfc1928), 大致看了一下，想着这么通用的东西应该有人实现过了，就想找实现好的。

由于我比较熟悉 Python，所以就找了一份 Python 的实现。

https://github.com/rushter/socks5/blob/master/server.py

代码如下：

```python
import logging
import select
import socket
import struct
from socketserver import ThreadingMixIn, TCPServer, StreamRequestHandler

logging.basicConfig(level=logging.DEBUG)
SOCKS_VERSION = 5


class ThreadingTCPServer(ThreadingMixIn, TCPServer):
    pass


class SocksProxy(StreamRequestHandler):
    username = 'username'
    password = 'password'

    def handle(self):
        logging.info('Accepting connection from %s:%s' % self.client_address)

        # greeting header
        # read and unpack 2 bytes from a client
        header = self.connection.recv(2)
        version, nmethods = struct.unpack("!BB", header)

        # socks 5
        assert version == SOCKS_VERSION
        assert nmethods > 0

        # get available methods
        methods = self.get_available_methods(nmethods)

        # accept only USERNAME/PASSWORD auth
        if 2 not in set(methods):
            # close connection
            self.server.close_request(self.request)
            return

        # send welcome message
        self.connection.sendall(struct.pack("!BB", SOCKS_VERSION, 2))

        if not self.verify_credentials():
            return

        # request
        version, cmd, _, address_type = struct.unpack("!BBBB", self.connection.recv(4))
        assert version == SOCKS_VERSION

        if address_type == 1:  # IPv4
            address = socket.inet_ntoa(self.connection.recv(4))
        elif address_type == 3:  # Domain name
            domain_length = ord(self.connection.recv(1)[0])
            address = self.connection.recv(domain_length)
            # resolve the domain name
            address = socket.gethostbyname(address)
        port = struct.unpack('!H', self.connection.recv(2))[0]

        # reply
        try:
            if cmd == 1:  # CONNECT
                remote = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                remote.connect((address, port))
                bind_address = remote.getsockname()
                logging.info('Connected to %s %s' % (address, port))
            else:
                self.server.close_request(self.request)

            addr = struct.unpack("!I", socket.inet_aton(bind_address[0]))[0]
            port = bind_address[1]
            reply = struct.pack("!BBBBIH", SOCKS_VERSION, 0, 0, 1,
                                addr, port)

        except Exception as err:
            logging.error(err)
            # return connection refused error
            reply = self.generate_failed_reply(address_type, 5)

        self.connection.sendall(reply)

        # establish data exchange
        if reply[1] == 0 and cmd == 1:
            self.exchange_loop(self.connection, remote)

        self.server.close_request(self.request)

    def get_available_methods(self, n):
        methods = []
        for i in range(n):
            methods.append(ord(self.connection.recv(1)))
        return methods

    def verify_credentials(self):
        version = ord(self.connection.recv(1))
        assert version == 1

        username_len = ord(self.connection.recv(1))
        username = self.connection.recv(username_len).decode('utf-8')

        password_len = ord(self.connection.recv(1))
        password = self.connection.recv(password_len).decode('utf-8')

        if username == self.username and password == self.password:
            # success, status = 0
            response = struct.pack("!BB", version, 0)
            self.connection.sendall(response)
            return True

        # failure, status != 0
        response = struct.pack("!BB", version, 0xFF)
        self.connection.sendall(response)
        self.server.close_request(self.request)
        return False

    def generate_failed_reply(self, address_type, error_number):
        return struct.pack("!BBBBIH", SOCKS_VERSION, error_number, 0, address_type, 0, 0)

    def exchange_loop(self, client, remote):

        while True:

            # wait until client or remote is available for read
            r, w, e = select.select([client, remote], [], [])

            if client in r:
                data = client.recv(4096)
                if remote.send(data) <= 0:
                    break

            if remote in r:
                data = remote.recv(4096)
                if client.send(data) <= 0:
                    break


if __name__ == '__main__':
    with ThreadingTCPServer(('127.0.0.1', 9011), SocksProxy) as server:
        server.serve_forever()

```

可以看到这里代码量不大，百行左右，逻辑也是比较简单的，SOCKS5 的运行时序图如下：

![](https://i.imgur.com/aR3qn4C.png)

server 端接到代理请求并获得要目标地址后，对目标地址发起 TCP连接，建立连接成功后持续交换数据，从客户端得到数据传送给远端，远端响应后得到响应数据传给客户端，和「中间人攻击」类似。其中对目标地址发起 TCP连接的代码在第 61 行，这里使用了 Python 的 socket 库。

好了，SOCKS5 我们弄明白了，那和 SS 啥关系，这个时候我们再去看 SS 的代码，就会发现，两者的实现基本上是一样的，SS 本地也是运行了一个 SOKCS5 server，主要区别就是

1. SS 的 SOCKS5 server 没有实现鉴权，因为运行在本地，可以视为一个可信网络，一般不用鉴权
2. SS 接到客户端的代理请求后，并没有直接去连接目标服务器，而是创建了一个到一个「代理服务器」的连接 https://github.com/shadowsocks/shadowsocks/blob/9d3e2d717753ba9489fcd553cd444449fffb13ac/shadowsocks/local.py#L156 ，这里的服务器端口就是我们经常需要写在配置文件里的服务器端口了。
3. 多了一个 SS 的服务端，客户端是请求不在 SOCKS5 server 接到代理请求时发送了，总得有个地方发送吧，SS 把这部分工作移到了一个 SS server 的服务端进程来做。
4. SS 在传输数据给代理服务器的时候，对数据进行了加密，也就是 157 行调用的这个方法 https://github.com/shadowsocks/shadowsocks/blob/9d3e2d717753ba9489fcd553cd444449fffb13ac/shadowsocks/local.py#L157 ，这里的加密应用的就是配置里的密码套件选项了，如最开始经常用的 aes-128-cfb，现在的 aes-128-gcm 等。

SS server 的实现和 local 很像，它从 local 获得要连接的目标地址和要传输的数据，对目标服务器发起连接并把数据传给对方，等对方的响应数据到了再把响应数据传给 local，这样整个数据传输链条就通了。

SS 的时序图如下：

![](https://i.imgur.com/Tc8dUsj.png)


这里我们就基本清楚 SS 是怎么工作了。

到这里，也解答了一个我之前理解的不太对的一个概念，网络请求是怎么发出去的？

根据 OSI 7层参考模型和 TCP/IP 4 层模型

![https://i.imgur.com/XJulVv3.png](https://i.imgur.com/XJulVv3.png)

我们通常的 HTTP 请求工作在应用层，发起 HTTP 请求时先发起一个 TCP连接，再将 HTTP 的数据包传给服务端，伪代码如下：

```
def prepare_HTTP_data():
    # do something

    # 这里的数据是 bytes
    return data

remote = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
remote.connect((address, port))
data = prepare_HTTP_data()
remote.send(data)
```

这里也体现了 TCP 作为「传输层」协议的责任，只管把数据传输到目标服务器，并不关心传的是什么，而 HTTP 的实现都由自己这一层完成，和 TCP 无关，TCP 这是层是「可插拔」的，这里通过 TCP 传输，理论上也可以通过其他协议传输，通常 HTTP 要求依赖的传输协议是「可靠」的，而可能快要定版的 HTTP3 不再使用 TCP 传输，也是基于 UDP 再实现了一个可靠的传输协议，用以替换 HTTP 依赖的 TCP。


### 怎么自己实现一个代理协议？

1. 实现一个 SOCKS5 server，用以接管流量
2. 定义一个协议，将接管到的流量数据传给代理服务器
3. 选择一个传输层协议用来传输这些数据
4. 实现一个代理服务器，用来接受流量数据并转发出去

这应该也是当初 clowwindy 的思路吧 [https://web.archive.org/web/20140930095114/http://www.v2ex.com/t/32777](https://web.archive.org/web/20140930095114/http://www.v2ex.com/t/32777)

本着动手造轮子更好理解的原则，准备自己实现一个类似的方案。

#### 协议

地址沿用 SOCKS5 的定义 https://tools.ietf.org/html/rfc1928#section-5

地址后跟着要传输的数据，也就是

    [target][payload]

传输协议使用 SSL，现成的传输协议，经过了各种项目的考验，实现也现成，不用造轮子，当然缺点就是不够「轻量」，不过没有太大所谓。最近 v2ray 社区在开发一个新的协议，VLESS，其要求传输层用像 TLS 类的「可靠信道」，且经过了激烈的争论 https://github.com/v2ray/v2ray-core/issues/2636

我实现了两个版本，一个是多线程版本，一个是 aio 版本，代码放到这里吧。

多线程 local

```python
import struct

import socket

import select

from socketserver import ThreadingMixIn, TCPServer, StreamRequestHandler

import ssl

SOCKS_VERSION = 5

class ThreadingTCPServer(ThreadingMixIn, TCPServer):
    pass

class SOCKSProxy(StreamRequestHandler):
    def handle(self):
        header = self.connection.recv(2)
        version, nmethods = struct.unpack('!BB', header)
        print(version, nmethods)
        self.connection.sendall(struct.pack("!BB", SOCKS_VERSION, 0))

        methods = []
        for i in range(nmethods):
            methods.append(ord(self.connection.recv(1)))

        print(methods)

        version, cmd, _, addr_type = struct.unpack('!BBBB', self.connection.recv(4))
        print(version, cmd, addr_type)

        domain_length = 0
        if addr_type == 1:
            address_bytes = self.connection.recv(4)
            address = socket.inet_ntoa(address_bytes)
        elif addr_type == 3:
            domain_length = self.connection.recv(1)[0]
            address = self.connection.recv(domain_length)
            address_bytes = address

        port = struct.unpack('!H', self.connection.recv(2))[0]
        print(address, address_bytes, port)

        addr_pack_fro_remote = struct.pack(f'!BB{len(address_bytes)}sH', addr_type, domain_length, address_bytes, port)
        print(struct.unpack(f'!BB{len(address_bytes)}sH', addr_pack_fro_remote))


        if cmd == 1:
            remote = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            remote = ssl.wrap_socket(remote,
                           ssl_version=ssl.PROTOCOL_TLSv1_2)
            remote.connect(('ahhh-prod.imciel.com', 9012))
            bind_address = remote.getsockname()
        else:
            self.server.close_request(self.request)
        
        addr = struct.unpack('!I', socket.inet_aton(bind_address[0]))[0]
        port = bind_address[1]
        reply = struct.pack('!BBBBIH', SOCKS_VERSION, 0, 0, 1, addr, port)
        print(reply)
        self.connection.sendall(reply)


        addr_send = False

        while True:
            try:
                r, w, e = select.select([self.connection, remote], [], [])

                if self.connection in r:
                    if not addr_send:
                        remote.send(addr_pack_fro_remote)
                        addr_send = True

                    data = self.connection.recv(4096)
                    if remote.send(data) <= 0:
                        break

                if remote in r:
                    data = remote.recv(4096)
                    if self.connection.send(data) <= 0:
                        break
            except Exception as e:
                print(e)
                break



if __name__ == "__main__":
    with ThreadingTCPServer(('localhost', 9011), SOCKSProxy) as server:
        server.serve_forever()
```

多线程 server

```python

import struct

import socket

import select

from socketserver import ThreadingMixIn, TCPServer, StreamRequestHandler

import ssl

SOCKS_VERSION = 5

class TLSServer(TCPServer):
    def __init__(self,
                 server_address,
                 RequestHandlerClass,
                 certfile,
                 keyfile,
                 ssl_version=ssl.PROTOCOL_TLSv1_2,
                 bind_and_activate=True):
        TCPServer.__init__(self, server_address, RequestHandlerClass, bind_and_activate)
        self.certfile = certfile
        self.keyfile = keyfile
        self.ssl_version = ssl_version

    def get_request(self):
        newsocket, fromaddr = self.socket.accept()

        debug = True
        if debug:
            return newsocket, fromaddr
        connstream = ssl.wrap_socket(newsocket,
                                 server_side=True,
                                 certfile = self.certfile,
                                 keyfile = self.keyfile,
                                 ssl_version = self.ssl_version)
        return connstream, fromaddr

class ThreadingTCPServer(ThreadingMixIn, TLSServer):
    pass

class SOCKSProxy(StreamRequestHandler):
    def handle(self):
        header = self.connection.recv(2)
        addr_type, domain_length = struct.unpack('!BB', header)
        print(header)

        if addr_type == 1:
            address_bytes = self.connection.recv(4)
            address = socket.inet_ntoa(address_bytes)
        elif addr_type == 3:
            address = self.connection.recv(domain_length)
            address = socket.gethostbyname(address)


        port = struct.unpack('!H', self.connection.recv(2))[0]
        print(addr_type, address, port)

        remote = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        remote.connect((address, port))
        bind_address = remote.getsockname()
        
        addr = struct.unpack('!I', socket.inet_aton(bind_address[0]))[0]
        port = bind_address[1]
        # reply = struct.pack('!BBBBIH', SOCKS_VERSION, 0, 0, addr_type, addr, port)
        # print(reply)
        # self.connection.sendall(reply)


        while True:
            r, w, e = select.select([self.connection, remote], [], [])

            if self.connection in r:
                data = self.connection.recv(4096)
                if remote.send(data) <= 0:
                    break

            if remote in r:
                data = remote.recv(4096)
                if self.connection.send(data) <= 0:
                    break

        self.connection.close()
        remote.close()



if __name__ == "__main__":
    with ThreadingTCPServer(('localhost', 9012), SOCKSProxy, 'fullchain.pem', 'privkey.pem') as server:
    # with ThreadingTCPServer(('localhost', 9012), SOCKSProxy, 'selfsigned.cert', 'selfsigned.key') as server:
        server.serve_forever()

    # with ThreadingTCPServer(('localhost', 9011), SOCKSProxy) as server:
    #     server.serve_forever()
```

aio local

```python
import asyncio
import logging
from aio_protocol import LocalTunnelProtocol
import uvloop

logging.basicConfig(level=logging.DEBUG)

async def run_local():
        

    try:
        # server_address = 'ahhh-prod.imciel.com'
        server_address = '127.0.0.1'
        server_port = 9012

        loop = asyncio.get_running_loop()
        # Start the server and serve forever
        server = await loop.create_server(
            lambda: LocalTunnelProtocol(server_address, server_port),
            # lambda: BaseTunnelProtocol(),
            '0.0.0.0', 1081 
        )
        async with server:
            await server.serve_forever()
    except Exception as e:
        print(e)


if __name__ == '__main__':
    uvloop.install()
    asyncio.run(run_local())
```


aio server 

```python
import asyncio
from aio_protocol import ServerProtocol
import logging
import uvloop

logging.basicConfig(level=logging.DEBUG)

async def run_server():
    try:
        loop = asyncio.get_running_loop()
        # Start the server and serve forever
        server = await loop.create_server(
            lambda: ServerProtocol(),
            # lambda: BaseTunnelProtocol(),
            '0.0.0.0', 9012 
        )
        async with server:
            await server.serve_forever()
    except Exception as e:
        print(e)


if __name__ == '__main__':
    uvloop.install()
    asyncio.run(run_server())
```

其中 aio 版本有一个公共模块，是协议的定义和实现

aio_protocol.py

```python

import asyncio
import struct
import logging
import socket
import enum
import ssl

class Command(enum.IntEnum):
    """
    SOCKS5 request command type
    """
    CONNECT = 0x01
    BIND = 0x02
    UDP_ASSOCIATE = 0x03


class AddressType(enum.IntEnum):
    """
    SOCKS5 address type
    """
    IPV4 = 0x01
    DOMAINE_NAME = 0x03
    IPV6 = 0x04

class BaseTunnelProtocol(asyncio.StreamReaderProtocol):

    def __init__(self):
        loop = asyncio.get_running_loop()
        reader = asyncio.StreamReader(loop=loop)
        super().__init__(reader, loop=loop)

        self.reader = reader
        self.loop = loop
        self.transport = None
        self._shutdown = asyncio.Event()
        self.logger = logging.getLogger('aiotunnel.protocol.BaseTunnelProtocol')

    def reader(self):
        return self.reader

    async def read(self, fmt):
        """
        Unpacks data from the input stream

        :param fmt: Data format
        :return: The unpacked data
        """
        return struct.unpack(fmt, await self.reader.read(struct.calcsize(fmt)))

    def write(self, fmt, *argv):
        """
        Packs data to the writer

        :param fmt: Pack format
        :param argv: Pack content
        """
        self.transport.write(struct.pack(fmt, *argv))


    def connection_made(self, transport):
        self.transport = transport
        self.transport.set_write_buffer_limits(0)
        sock = transport.get_extra_info('socket')
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
        super().connection_made(transport)

    def connection_lost(self, exc):
        self.logger.debug('The server closed the connection')
        self.transport.close()
        super().connection_lost(exc)

    def eof_received(self):
        self.logger.debug('No more data to receive')
        self.transport.close()
        super().eof_received()

    def close(self):
        self._shutdown.set()
        self.transport.close()
        # super().close()
    # async def wait_closed(self):
    #     await self.transport.wait_closed()


class LocalOpenRemoteProtocol(BaseTunnelProtocol):

    def __init__(self, transport, loop):
        super().__init__()
        self.client_transport = transport
        self.logger = logging.getLogger('aiotunnel.protocol.LocalOpenRemoteProtocol')

    def write(self, data):
        self.transport.write(data)

    async def drain(self):
        await self.transport.drain()

    def connection_made(self, transport):
        super().connection_made(transport)
        self.loop.create_task(self.async_write_data())

    async def async_write_data(self):
        while not self._shutdown.is_set():
            data = await self.reader.read(4096)
            if not data:
                break
            self.client_transport.write(data)

class ServerProtocol(BaseTunnelProtocol):

    def __init__(self, on_conn_lost=None):
        super().__init__()
        self.on_conn_lost = on_conn_lost
        self.logger = logging.getLogger('aiotunnel.protocol.ServerProtocol')

    def connection_made(self, transport):
        super().connection_made(transport)
        self.loop.create_task(self.async_write_data())

    def data_received(self, data):
        super().data_received(data)

    async def async_write_data(self):
        while not self._shutdown.is_set():
            addr_type, domain_length = await self.read('!BB')

            if addr_type == 1:
                address_bytes = await self.reader.read(4)
                address = socket.inet_ntoa(address_bytes)
            elif addr_type == 3:
                address = await self.reader.read(domain_length)
                address = socket.gethostbyname(address)


            port_data = await self.read('!H')
            port = port_data[0]

            ssl_ctx = ssl.create_default_context()

            debug = True
            if debug:
                ssl_ctx = None

            self.logger.debug(f'Open connection {address}:{port}')

            protocol = LocalOpenRemoteProtocol(self.transport, self.loop)
            remote_reader, remote_writer = await self.loop.create_connection(
                lambda: protocol, address, port)

            writer = asyncio.StreamWriter(remote_reader, protocol, protocol.reader, self.loop)
            self.logger.debug(f'Opened {address}:{port}')

            # Get our own details
            server_socket = remote_reader.get_extra_info("socket")
            bind_address, bind_port = server_socket.getsockname()[:2]
            bind_address = socket.inet_pton(server_socket.family, bind_address)
            if server_socket.family == socket.AF_INET:
                address_type = AddressType.IPV4
            elif server_socket.family == socket.AF_INET6:
                address_type = AddressType.IPV6

            data = await self.reader.read(4096)
            while data:
                remote_writer.write(data)
                # await remote_writer.drain()
                data = await self.reader.read(4096)

            writer.close()
            await writer.wait_closed()
            self.close()


class LocalTunnelProtocol(BaseTunnelProtocol):

    def __init__(self, server_hostname, server_port, on_conn_lost=None):
        super().__init__()
        self.server_hostname = server_hostname
        self.server_port = server_port
        self.write_queue = asyncio.Queue()
        self.on_conn_lost = on_conn_lost
        self.logger = logging.getLogger('aiotunnel.protocol.LocalTunnelProtocol')

    def connection_made(self, transport):
        super().connection_made(transport)
        self.loop.create_task(self.async_write_data())

    async def async_write_data(self):
        while not self._shutdown.is_set():
            try:
                version, nb_methods = await self.read('!BB')
            except struct.error:
                self.write('!BB', 0x05, 0x01)
                self.close()
                return

            if version != 0x05:
                self.write('!BB', 0x05, 0x01)
                self.close()
                return

            # Ignore the methods
            await self.reader.read(nb_methods)

            # Sends the server "selected" method: no authentication
            self.write('!BB', version, 0x00)

            # Read the header of the request
            version, cmd, _, address_type = await self.read('!BBBB')
            domain_length = 0
            if cmd != 0x01:
                write_reply(ReplyCode.COMMAND_NOT_SUPPORTED)
                self.close()
                return

            if address_type == AddressType.IPV4:
                # IPv4 connection
                raw_address = await self.reader.read(4)
                address = socket.inet_ntop(socket.AF_INET, raw_address)
            elif address_type == AddressType.IPV6:
                # IPv6 connection
                raw_address = await self.reader.read(16)
                address = socket.inet_ntop(socket.AF_INET6, raw_address)
            elif address_type == AddressType.DOMAINE_NAME:
                # DNS resolution
                domain_length = (await self.read('!B'))[0]
                hostname = (await self.read("!{}s".format(domain_length)))[0]
                # address = socket.gethostbyname(hostname)
                address = hostname
            else:
                write_reply(ReplyCode.ADDRESS_NOT_SUPPORTED)
                self.close()
                raise IOError("Unhandled address type: {:x}".format(address_type))

            port = (await self.read('!H'))[0]

            if type(address) == str:
                address = address.encode('utf-8')
            addr_pack_fro_remote = struct.pack(f'!BB{domain_length}sH', address_type, domain_length, address, port)

            # transport, _ = await self.loop.create_connection(
            #     lambda: LocalOpenRemoteProtocol, host, port)
            ssl_ctx = ssl.create_default_context()
            debug = True
            if debug:
                ssl_ctx = None

            self.logger.debug(f'Open connection {address}:{port}')

            protocol = LocalOpenRemoteProtocol(self.transport, self.loop)
            remote_reader, remote_writer = await self.loop.create_connection(
                lambda: protocol, self.server_hostname, self.server_port, ssl=ssl_ctx)

            writer = asyncio.StreamWriter(remote_reader, protocol, protocol.reader, self.loop)
            self.logger.debug(f'Opened {address}:{port}')

            # Get our own details
            server_socket = remote_reader.get_extra_info("socket")
            bind_address, bind_port = server_socket.getsockname()[:2]
            bind_address = socket.inet_pton(server_socket.family, bind_address)
            if server_socket.family == socket.AF_INET:
                address_type = AddressType.IPV4
            elif server_socket.family == socket.AF_INET6:
                address_type = AddressType.IPV6

            self.write('!BBBB', 0x05, 0x00, 0x00, address_type)
            self.transport.write(bind_address)
            self.write('!H', port)

            remote_writer.write(addr_pack_fro_remote)
            data = await self.reader.read(4096)
            while data:
                remote_writer.write(data)
                data = await self.reader.read(4096)

            writer.close()
            await writer.wait_closed()
```

### 引用

- [Writing a simple SOCKS server in Python](https://rushter.com/blog/python-socks-server/)
- [Socks5 Proxy & Shadowsocks](https://xulog.com/2018/05/29/Socks5Proxy-Shadowsocks.html)

### 致谢

- [aiotunnel](https://github.com/codepr/aiotunnel)