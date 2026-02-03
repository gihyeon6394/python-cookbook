# CHAPTER 11 Network and Web Programming

이 챕터는 Python을 네트워크 및 분산 애플리케이션에 사용하는 다양한 주제를 다룬다. 기존 서비스에 접근하는 클라이언트로서의 Python 사용과 서버로서 네트워크 서비스를 구현하는 내용으로 나뉜다.

## 11.1. HTTP 서비스와 클라이언트로 상호작용하기

### 문제

HTTP를 통해 다양한 서비스에 클라이언트로 접근해야 한다. 예를 들어 데이터 다운로드나 REST 기반 API와의 상호작용이 필요하다.

### 해결방법

**기본 urllib 사용:**

```python
from urllib import request, parse

# GET 요청
url = 'http://httpbin.org/get'
parms = {'name1': 'value1', 'name2': 'value2'}
querystring = parse.urlencode(parms)
u = request.urlopen(url + '?' + querystring)
resp = u.read()

# POST 요청
u = request.urlopen(url, querystring.encode('ascii'))
resp = u.read()

# 커스텀 헤더 추가
headers = {'User-agent': 'none/ofyourbusiness', 'Spam': 'Eggs'}
req = request.Request(url, querystring.encode('ascii'), headers=headers)
u = request.urlopen(req)
resp = u.read()
```

**requests 라이브러리 사용 (권장):**

```python
import requests

# POST 요청
resp = requests.post(url, data=parms, headers=headers)
text = resp.text  # Unicode 디코딩된 텍스트
content = resp.content  # 바이너리 컨텐츠
json_data = resp.json  # JSON 파싱

# HEAD 요청
resp = requests.head('http://www.python.org/index.html')
status = resp.status_code
last_modified = resp.headers['last-modified']

# 기본 인증
resp = requests.get('http://pypi.python.org/pypi?:action=login',
                    auth=('user', 'password'))

# 쿠키 전달
resp1 = requests.get(url)
resp2 = requests.get(url, cookies=resp1.cookies)

# 파일 업로드
files = {'file': ('data.csv', open('data.csv', 'rb'))}
r = requests.post(url, files=files)
```

### 논의

간단한 GET/POST 요청은 urllib로 충분하지만, 더 복잡한 작업은 requests 라이브러리를 사용하는 것이 좋다. requests는 응답 처리가 편리하며(text, content, json 속성),
프록시, 인증, 쿠키 등을 간편하게 처리할 수 있다.

테스트 시에는 httpbin.org 서비스를 사용하면 편리하다. 이 서비스는 요청을 받아 JSON 형태로 정보를 반환한다.

## 11.2. TCP 서버 생성하기

### 문제

TCP 인터넷 프로토콜을 사용하여 클라이언트와 통신하는 서버를 구현해야 한다.

### 해결방법

**socketserver 라이브러리 사용:**

```python
from socketserver import BaseRequestHandler, TCPServer

class EchoHandler(BaseRequestHandler):
    def handle(self):
        print('Got connection from', self.client_address)
        while True:
            msg = self.request.recv(8192)
            if not msg:
                break
            self.request.send(msg)

if __name__ == '__main__':
    serv = TCPServer(('', 20000), EchoHandler)
    serv.serve_forever()
```

**파일 인터페이스를 사용하는 핸들러:**

```python
from socketserver import StreamRequestHandler, TCPServer

class EchoHandler(StreamRequestHandler):
    def handle(self):
        print('Got connection from', self.client_address)
        for line in self.rfile:  # 파일형 객체로 읽기
            self.wfile.write(line)  # 파일형 객체로 쓰기
```

### 논의

**멀티스레드 서버:**

```python
from socketserver import ThreadingTCPServer

serv = ThreadingTCPServer(('', 20000), EchoHandler)
serv.serve_forever()
```

**워커 스레드 풀 사용:**

```python
from threading import Thread

NWORKERS = 16
serv = TCPServer(('', 20000), EchoHandler)
for n in range(NWORKERS):
    t = Thread(target=serv.serve_forever)
    t.daemon = True
    t.start()
serv.serve_forever()
```

**소켓 옵션 설정:**

```python
serv = TCPServer(('', 20000), EchoHandler, bind_and_activate=False)
serv.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, True)
serv.server_bind()
serv.server_activate()
serv.serve_forever()

# 또는 클래스 변수로 설정
TCPServer.allow_reuse_address = True
serv = TCPServer(('', 20000), EchoHandler)
```

**순수 소켓 구현:**

```python
from socket import socket, AF_INET, SOCK_STREAM

def echo_handler(address, client_sock):
    print('Got connection from {}'.format(address))
    while True:
        msg = client_sock.recv(8192)
        if not msg:
            break
        client_sock.sendall(msg)
    client_sock.close()

def echo_server(address, backlog=5):
    sock = socket(AF_INET, SOCK_STREAM)
    sock.bind(address)
    sock.listen(backlog)
    while True:
        client_sock, client_addr = sock.accept()
        echo_handler(client_addr, client_sock)
```

## 11.3. UDP 서버 생성하기

### 문제

UDP 인터넷 프로토콜을 사용하여 클라이언트와 통신하는 서버를 구현해야 한다.

### 해결방법

```python
from socketserver import BaseRequestHandler, UDPServer
import time

class TimeHandler(BaseRequestHandler):
    def handle(self):
        print('Got connection from', self.client_address)
        msg, sock = self.request  # 메시지와 소켓
        resp = time.ctime()
        sock.sendto(resp.encode('ascii'), self.client_address)

if __name__ == '__main__':
    serv = UDPServer(('', 20000), TimeHandler)
    serv.serve_forever()
```

**클라이언트 테스트:**

```python
from socket import socket, AF_INET, SOCK_DGRAM

s = socket(AF_INET, SOCK_DGRAM)
s.sendto(b'', ('localhost', 20000))
print(s.recvfrom(8192))  # (b'Wed Aug 15 20:35:08 2012', ('127.0.0.1', 20000))
```

### 논의

UDP는 연결이 없고 신뢰성이 없다. 메시지 손실 가능성이 있으므로 중요한 애플리케이션에서는 시퀀스 번호, 재시도, 타임아웃 등의 메커니즘이 필요하다. UDP는 실시간 멀티미디어 스트리밍이나 게임처럼 신뢰성보다
속도가 중요한 경우에 주로 사용된다.

**멀티스레드 UDP 서버:**

```python
from socketserver import ThreadingUDPServer

serv = ThreadingUDPServer(('', 20000), TimeHandler)
serv.serve_forever()
```

## 11.4. CIDR 주소에서 IP 주소 범위 생성하기

### 문제

"123.45.67.89/27" 같은 CIDR 네트워크 주소에서 모든 IP 주소 범위를 생성해야 한다.

### 해결방법

```python
import ipaddress

# IPv4
net = ipaddress.ip_network('123.45.67.64/27')
for a in net:
    print(a)  # 123.45.67.64부터 123.45.67.95까지

# IPv6
net6 = ipaddress.ip_network('12:3456:78:90ab:cd:ef01:23:30/125')
for a in net6:
    print(a)

# 배열처럼 인덱싱
print(net.num_addresses)  # 32
print(net[0])  # IPv4Address('123.45.67.64')
print(net[-1])  # IPv4Address('123.45.67.95')

# 멤버십 체크
a = ipaddress.ip_address('123.45.67.69')
print(a in net)  # True

# IP 인터페이스
inet = ipaddress.ip_interface('123.45.67.73/27')
print(inet.network)  # IPv4Network('123.45.67.64/27')
print(inet.ip)  # IPv4Address('123.45.67.73')
```

### 논의

ipaddress 모듈과 socket 라이브러리 간의 상호작용은 제한적이다. IPv4Address 인스턴스를 주소 문자열 대신 직접 사용할 수 없으며, str()로 명시적 변환이 필요하다.

## 11.5. 간단한 REST 기반 인터페이스 생성하기

### 문제

네트워크를 통해 프로그램을 원격으로 제어하거나 상호작용할 수 있는 간단한 REST 기반 인터페이스가 필요하지만, 완전한 웹 프레임워크를 설치하고 싶지 않다.

### 해결방법

**WSGI 기반 디스패처 구현:**

```python
# resty.py
import cgi

def notfound_404(environ, start_response):
    start_response('404 Not Found', [('Content-type', 'text/plain')])
    return [b'Not Found']

class PathDispatcher:
    def __init__(self):
        self.pathmap = {}
    
    def __call__(self, environ, start_response):
        path = environ['PATH_INFO']
        params = cgi.FieldStorage(environ['wsgi.input'], environ=environ)
        method = environ['REQUEST_METHOD'].lower()
        environ['params'] = {key: params.getvalue(key) for key in params}
        handler = self.pathmap.get((method, path), notfound_404)
        return handler(environ, start_response)
    
    def register(self, method, path, function):
        self.pathmap[method.lower(), path] = function
        return function
```

**핸들러 작성:**

```python
import time

def hello_world(environ, start_response):
    start_response('200 OK', [('Content-type', 'text/html')])
    params = environ['params']
    resp = '<html><body><h1>Hello {name}!</h1></body></html>'.format(
        name=params.get('name'))
    yield resp.encode('utf-8')

def localtime(environ, start_response):
    start_response('200 OK', [('Content-type', 'application/xml')])
    resp = '''<?xml version="1.0"?>
<time>
  <year>{t.tm_year}</year>
  <month>{t.tm_mon}</month>
  <day>{t.tm_mday}</day>
</time>'''.format(t=time.localtime())
    yield resp.encode('utf-8')

if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    
    dispatcher = PathDispatcher()
    dispatcher.register('GET', '/hello', hello_world)
    dispatcher.register('GET', '/localtime', localtime)
    
    httpd = make_server('', 8080, dispatcher)
    print('Serving on port 8080...')
    httpd.serve_forever()
```

### 논의

WSGI는 서버와 프레임워크에 중립적인 표준이므로 다양한 서버에서 애플리케이션을 사용할 수 있다. WSGI 애플리케이션은 특정 호출 규약을 따르는 callable로 구현된다:

- environ: CGI 스타일의 딕셔너리
- start_response: 응답을 시작하는 함수
- 반환값: 바이트 문자열의 시퀀스

보다 고급 기능(인증, 쿠키, 리다이렉트 등)이 필요하면 WebOb나 Paste 같은 서드파티 라이브러리를 고려할 수 있다.

## 11.6. XML-RPC로 간단한 원격 프로시저 호출 구현하기

### 문제

원격 머신에서 실행되는 Python 프로그램의 함수나 메서드를 쉽게 실행하고 싶다.

### 해결방법

**서버 구현:**

```python
from xmlrpc.server import SimpleXMLRPCServer

class KeyValueServer:
    _rpc_methods_ = ['get', 'set', 'delete', 'exists', 'keys']
    
    def __init__(self, address):
        self._data = {}
        self._serv = SimpleXMLRPCServer(address, allow_none=True)
        for name in self._rpc_methods_:
            self._serv.register_function(getattr(self, name))
    
    def get(self, name):
        return self._data[name]
    
    def set(self, name, value):
        self._data[name] = value
    
    def delete(self, name):
        del self._data[name]
    
    def exists(self, name):
        return name in self._data
    
    def keys(self):
        return list(self._data)
    
    def serve_forever(self):
        self._serv.serve_forever()

if __name__ == '__main__':
    kvserv = KeyValueServer(('', 15000))
    kvserv.serve_forever()
```

**클라이언트 사용:**

```python
from xmlrpc.client import ServerProxy

s = ServerProxy('http://localhost:15000', allow_none=True)
s.set('foo', 'bar')
s.set('spam', [1, 2, 3])
print(s.keys())  # ['spam', 'foo']
print(s.get('foo'))  # 'bar'
```

### 논의

XML-RPC는 간단한 원격 프로시저 호출 서비스를 설정하는 매우 쉬운 방법이다. 하지만 제한사항이 있다:

- 문자열, 숫자, 리스트, 딕셔너리 등 특정 데이터 타입만 지원
- 바이너리 데이터는 특별한 처리 필요
- 단일 스레드 구현으로 성능이 제한적
- XML 인코딩으로 인해 다른 방식보다 느림

하지만 다양한 프로그래밍 언어에서 이해할 수 있는 형식이라는 장점이 있다.

## 11.7. 인터프리터 간 간단한 통신

### 문제

여러 Python 인터프리터 인스턴스가 실행 중이며(서로 다른 머신에 있을 수 있음), 메시지를 통해 데이터를 교환하고 싶다.

### 해결방법

```python
from multiprocessing.connection import Listener, Client
import traceback

# 서버
def echo_client(conn):
    try:
        while True:
            msg = conn.recv()
            conn.send(msg)
    except EOFError:
        print('Connection closed')

def echo_server(address, authkey):
    serv = Listener(address, authkey=authkey)
    while True:
        try:
            client = serv.accept()
            echo_client(client)
        except Exception:
            traceback.print_exc()

echo_server(('', 25000), authkey=b'peekaboo')

# 클라이언트
c = Client(('localhost', 25000), authkey=b'peekaboo')
c.send('hello')
print(c.recv())  # 'hello'
c.send([1, 2, 3, 4, 5])
print(c.recv())  # [1, 2, 3, 4, 5]
```

### 논의

저수준 소켓과 달리 메시지가 완전히 보존된다(send()로 보낸 각 객체가 recv()로 완전히 수신됨). 객체는 pickle을 사용하여 직렬화되므로 pickle과 호환되는 모든 객체를 주고받을 수 있다.

**UNIX 도메인 소켓 사용:**

```python
s = Listener('/tmp/myconn', authkey=b'peekaboo')
```

**Windows 명명된 파이프 사용:**

```python
s = Listener(r'\\.\pipe\myconn', authkey=b'peekaboo')
```

이 방식은 공개 서비스보다는 내부 네트워크에서 사용하기 적합하며, authkey로 인증을 제공한다.

## 11.8. 원격 프로시저 호출 구현하기

### 문제

소켓, multiprocessing 연결, ZeroMQ 같은 메시지 전달 레이어 위에 간단한 RPC를 구현하고 싶다.

### 해결방법

**RPC 핸들러:**

```python
import pickle

class RPCHandler:
    def __init__(self):
        self._functions = {}
    
    def register_function(self, func):
        self._functions[func.__name__] = func
    
    def handle_connection(self, connection):
        try:
            while True:
                func_name, args, kwargs = pickle.loads(connection.recv())
                try:
                    r = self._functions[func_name](*args, **kwargs)
                    connection.send(pickle.dumps(r))
                except Exception as e:
                    connection.send(pickle.dumps(e))
        except EOFError:
            pass
```

**RPC 서버:**

```python
from multiprocessing.connection import Listener
from threading import Thread

def rpc_server(handler, address, authkey):
    sock = Listener(address, authkey=authkey)
    while True:
        client = sock.accept()
        t = Thread(target=handler.handle_connection, args=(client,))
        t.daemon = True
        t.start()

def add(x, y):
    return x + y

def sub(x, y):
    return x - y

handler = RPCHandler()
handler.register_function(add)
handler.register_function(sub)

rpc_server(handler, ('localhost', 17000), authkey=b'peekaboo')
```

**RPC 프록시:**

```python
class RPCProxy:
    def __init__(self, connection):
        self._connection = connection
    
    def __getattr__(self, name):
        def do_rpc(*args, **kwargs):
            self._connection.send(pickle.dumps((name, args, kwargs)))
            result = pickle.loads(self._connection.recv())
            if isinstance(result, Exception):
                raise result
            return result
        return do_rpc

# 사용
from multiprocessing.connection import Client

c = Client(('localhost', 17000), authkey=b'peekaboo')
proxy = RPCProxy(c)
print(proxy.add(2, 3))  # 5
print(proxy.sub(2, 3))  # -1
```

### 논의

pickle 대신 JSON을 사용할 수도 있다:

```python
import json

# 핸들러에서
func_name, args, kwargs = json.loads(connection.recv())
connection.send(json.dumps(r))

# 프록시에서
self._connection.send(json.dumps((name, args, kwargs)))
result = json.loads(self._connection.recv())
```

보안이 중요하므로 신뢰할 수 없는 클라이언트로부터의 RPC는 허용하지 말아야 한다.

## 11.9. 클라이언트 간단히 인증하기

### 문제

분산 시스템에서 서버에 연결하는 클라이언트를 인증할 간단한 방법이 필요하지만, SSL의 복잡성은 원하지 않는다.

### 해결방법

```python
import hmac
import os

def client_authenticate(connection, secret_key):
    """클라이언트를 원격 서비스에 인증"""
    message = connection.recv(32)
    hash = hmac.new(secret_key, message)
    digest = hash.digest()
    connection.send(digest)

def server_authenticate(connection, secret_key):
    """클라이언트 인증 요청"""
    message = os.urandom(32)
    connection.send(message)
    hash = hmac.new(secret_key, message)
    digest = hash.digest()
    response = connection.recv(len(digest))
    return hmac.compare_digest(digest, response)
```

**서버에서 사용:**

```python
from socket import socket, AF_INET, SOCK_STREAM

secret_key = b'peekaboo'

def echo_handler(client_sock):
    if not server_authenticate(client_sock, secret_key):
        client_sock.close()
        return
    while True:
        msg = client_sock.recv(8192)
        if not msg:
            break
        client_sock.sendall(msg)

def echo_server(address):
    s = socket(AF_INET, SOCK_STREAM)
    s.bind(address)
    s.listen(5)
    while True:
        c, a = s.accept()
        echo_handler(c)

echo_server(('', 18000))
```

**클라이언트에서 사용:**

```python
from socket import socket, AF_INET, SOCK_STREAM

secret_key = b'peekaboo'
s = socket(AF_INET, SOCK_STREAM)
s.connect(('localhost', 18000))
client_authenticate(s, secret_key)
s.send(b'Hello World')
```

### 논의

HMAC 인증은 내부 메시징 시스템과 프로세스 간 통신에서 일반적으로 사용된다. 중요한 점은 인증이 암호화와 같지 않다는 것이다. 인증 후 통신은 평문으로 전송되며, 비밀 키는 전송되지 않는다.

## 11.10. 네트워크 서비스에 SSL 추가하기

### 문제

서버와 클라이언트가 SSL을 사용하여 인증하고 전송 데이터를 암호화하는 소켓 기반 네트워크 서비스를 구현해야 한다.

### 해결방법

**기본 SSL 서버:**

```python
from socket import socket, AF_INET, SOCK_STREAM
import ssl

KEYFILE = 'server_key.pem'
CERTFILE = 'server_cert.pem'

def echo_client(s):
    while True:
        data = s.recv(8192)
        if data == b'':
            break
        s.send(data)
    s.close()

def echo_server(address):
    s = socket(AF_INET, SOCK_STREAM)
    s.bind(address)
    s.listen(1)
    
    s_ssl = ssl.wrap_socket(s,
                           keyfile=KEYFILE,
                           certfile=CERTFILE,
                           server_side=True)
    
    while True:
        try:
            c, a = s_ssl.accept()
            echo_client(c)
        except Exception as e:
            print('{}: {}'.format(e.__class__.__name__, e))

echo_server(('', 20000))
```

**SSL 클라이언트:**

```python
from socket import socket, AF_INET, SOCK_STREAM
import ssl

s = socket(AF_INET, SOCK_STREAM)
s_ssl = ssl.wrap_socket(s,
                        cert_reqs=ssl.CERT_REQUIRED,
                        ca_certs='server_cert.pem')
s_ssl.connect(('localhost', 20000))
s_ssl.send(b'Hello World?')
print(s_ssl.recv(8192))
```

**기존 서버에 SSL 추가 (Mixin 사용):**

```python
import ssl

class SSLMixin:
    def __init__(self, *args, keyfile=None, certfile=None, 
                 ca_certs=None, cert_reqs=ssl.NONE, **kwargs):
        self._keyfile = keyfile
        self._certfile = certfile
        self._ca_certs = ca_certs
        self._cert_reqs = cert_reqs
        super().__init__(*args, **kwargs)
    
    def get_request(self):
        client, addr = super().get_request()
        client_ssl = ssl.wrap_socket(client,
                                     keyfile=self._keyfile,
                                     certfile=self._certfile,
                                     ca_certs=self._ca_certs,
                                     cert_reqs=self._cert_reqs,
                                     server_side=True)
        return client_ssl, addr

# XML-RPC 서버에 SSL 적용
from xmlrpc.server import SimpleXMLRPCServer

class SSLSimpleXMLRPCServer(SSLMixin, SimpleXMLRPCServer):
    pass

kvserv = KeyValueServer(('', 15000),
                       keyfile=KEYFILE,
                       certfile=CERTFILE)
kvserv.serve_forever()
```

**서버 인증서 검증하는 클라이언트:**

```python
from xmlrpc.client import SafeTransport, ServerProxy
import ssl

class VerifyCertSafeTransport(SafeTransport):
    def __init__(self, cafile, certfile=None, keyfile=None):
        SafeTransport.__init__(self)
        self._ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLSv1)
        self._ssl_context.load_verify_locations(cafile)
        if certfile:
            self._ssl_context.load_cert_chain(certfile, keyfile)
        self._ssl_context.verify_mode = ssl.CERT_REQUIRED
    
    def make_connection(self, host):
        s = super().make_connection((host, {'context': self._ssl_context}))
        return s

s = ServerProxy('https://localhost:15000',
               transport=VerifyCertSafeTransport('server_cert.pem'),
               allow_none=True)
```

### 논의

**자체 서명 인증서 생성:**

```bash
openssl req -new -x509 -days 365 -nodes -out server_cert.pem \
    -keyout server_key.pem
```

생성 시 "Common Name" 필드에 서버의 DNS 호스트명(테스트 시에는 "localhost")을 입력한다.

SSL 설정의 주요 도전 과제는 키, 인증서 및 기타 사항의 초기 구성이다. 각 SSL 연결의 엔드포인트는 일반적으로 개인 키와 서명된 인증서 파일이 필요하다.

## 11.11. 프로세스 간 소켓 파일 디스크립터 전달하기

### 문제

여러 Python 인터프리터 프로세스가 실행 중이며, 한 인터프리터에서 다른 인터프리터로 열린 파일 디스크립터를 전달하고 싶다.

### 해결방법

```python
import multiprocessing
from multiprocessing.reduction import recv_handle, send_handle
import socket

def worker(in_p, out_p):
    out_p.close()
    while True:
        fd = recv_handle(in_p)
        print('CHILD: GOT FD', fd)
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM, fileno=fd) as s:
            while True:
                msg = s.recv(1024)
                if not msg:
                    break
                print('CHILD: RECV {!r}'.format(msg))
                s.send(msg)

def server(address, in_p, out_p, worker_pid):
    in_p.close()
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, True)
    s.bind(address)
    s.listen(1)
    while True:
        client, addr = s.accept()
        print('SERVER: Got connection from', addr)
        send_handle(out_p, client.fileno(), worker_pid)
        client.close()

if __name__ == '__main__':
    c1, c2 = multiprocessing.Pipe()
    worker_p = multiprocessing.Process(target=worker, args=(c1, c2))
    worker_p.start()
    server_p = multiprocessing.Process(target=server,
                                      args=(('', 15000), c1, c2, worker_p.pid))
    server_p.start()
    c1.close()
    c2.close()
```

### 논의

파일 디스크립터 전달은 확장 가능한 시스템을 구축하는 데 유용한 도구다. 예를 들어, 멀티코어 머신에서 여러 Python 인터프리터 인스턴스를 실행하고 파일 디스크립터 전달을 사용하여 각 인터프리터가 처리하는
클라이언트 수를 더 균등하게 분산할 수 있다.

**별도 프로그램으로 서버/워커 분리:**

서버 (servermp.py):

```python
from multiprocessing.connection import Listener
from multiprocessing.reduction import send_handle
import socket

def server(work_address, port):
    work_serv = Listener(work_address, authkey=b'peekaboo')
    worker = work_serv.accept()
    worker_pid = worker.recv()
    
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, True)
    s.bind(('', port))
    s.listen(1)
    while True:
        client, addr = s.accept()
        send_handle(worker, client.fileno(), worker_pid)
        client.close()
```

워커 (workermp.py):

```python
from multiprocessing.connection import Client
from multiprocessing.reduction import recv_handle
import os
from socket import socket, AF_INET, SOCK_STREAM

def worker(server_address):
    serv = Client(server_address, authkey=b'peekaboo')
    serv.send(os.getpid())
    while True:
        fd = recv_handle(serv)
        with socket(AF_INET, SOCK_STREAM, fileno=fd) as client:
            while True:
                msg = client.recv(1024)
                if not msg:
                    break
                client.send(msg)
```

## 11.12. 이벤트 기반 I/O 이해하기

### 문제

"이벤트 기반" 또는 "비동기" I/O 기반 패키지에 대해 들었지만, 그것이 정확히 무엇을 의미하는지, 실제로 어떻게 작동하는지, 프로그램에 어떤 영향을 미치는지 완전히 이해하지 못한다.

### 해결방법

**기본 이벤트 핸들러 클래스:**

```python
class EventHandler:
    def fileno(self):
        raise NotImplemented('must implement')
    
    def wants_to_receive(self):
        return False
    
    def handle_receive(self):
        pass
    
    def wants_to_send(self):
        return False
    
    def handle_send(self):
        pass
```

**이벤트 루프:**

```python
import select

def event_loop(handlers):
    while True:
        wants_recv = [h for h in handlers if h.wants_to_receive()]
        wants_send = [h for h in handlers if h.wants_to_send()]
        can_recv, can_send, _ = select.select(wants_recv, wants_send, [])
        for h in can_recv:
            h.handle_receive()
        for h in can_send:
            h.handle_send()
```

**UDP 서버 예제:**

```python
import socket
import time

class UDPServer(EventHandler):
    def __init__(self, address):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.sock.bind(address)
    
    def fileno(self):
        return self.sock.fileno()
    
    def wants_to_receive(self):
        return True

class UDPTimeServer(UDPServer):
    def handle_receive(self):
        msg, addr = self.sock.recvfrom(1)
        self.sock.sendto(time.ctime().encode('ascii'), addr)

class UDPEchoServer(UDPServer):
    def handle_receive(self):
        msg, addr = self.sock.recvfrom(8192)
        self.sock.sendto(msg, addr)

if __name__ == '__main__':
    handlers = [UDPTimeServer(('', 14000)), UDPEchoServer(('', 15000))]
    event_loop(handlers)
```

**TCP 서버 예제:**

```python
class TCPServer(EventHandler):
    def __init__(self, address, client_handler, handler_list):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, True)
        self.sock.bind(address)
        self.sock.listen(1)
        self.client_handler = client_handler
        self.handler_list = handler_list
    
    def fileno(self):
        return self.sock.fileno()
    
    def wants_to_receive(self):
        return True
    
    def handle_receive(self):
        client, addr = self.sock.accept()
        self.handler_list.append(self.client_handler(client, self.handler_list))

class TCPClient(EventHandler):
    def __init__(self, sock, handler_list):
        self.sock = sock
        self.handler_list = handler_list
        self.outgoing = bytearray()
    
    def fileno(self):
        return self.sock.fileno()
    
    def close(self):
        self.sock.close()
        self.handler_list.remove(self)
    
    def wants_to_send(self):
        return True if self.outgoing else False
    
    def handle_send(self):
        nsent = self.sock.send(self.outgoing)
        self.outgoing = self.outgoing[nsent:]

class TCPEchoClient(TCPClient):
    def wants_to_receive(self):
        return True
    
    def handle_receive(self):
        data = self.sock.recv(8192)
        if not data:
            self.close()
        else:
            self.outgoing.extend(data)
```

### 논의

이벤트 기반 I/O의 장점은 스레드나 프로세스를 사용하지 않고도 매우 많은 동시 연결을 처리할 수 있다는 것이다. select() 호출로 수백 또는 수천 개의 소켓을 모니터링하고 발생하는 이벤트에 응답할 수 있다.

단점은 진정한 동시성이 없다는 것이다. 이벤트 핸들러 메서드 중 하나가 블록되거나 장시간 계산을 수행하면 모든 것의 진행이 중단된다.

**스레드 풀과 함께 사용:**

```python
from concurrent.futures import ThreadPoolExecutor
import os

class ThreadPoolHandler(EventHandler):
    def __init__(self, nworkers):
        if os.name == 'posix':
            self.signal_done_sock, self.done_sock = socket.socketpair()
        else:
            # Windows 구현
            server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            server.bind(('127.0.0.1', 0))
            server.listen(1)
            self.signal_done_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self.signal_done_sock.connect(server.getsockname())
            self.done_sock, _ = server.accept()
            server.close()
        
        self.pending = []
        self.pool = ThreadPoolExecutor(nworkers)
    
    def fileno(self):
        return self.done_sock.fileno()
    
    def _complete(self, callback, r):
        self.pending.append((callback, r.result()))
        self.signal_done_sock.send(b'x')
    
    def run(self, func, args=(), kwargs={}, *, callback):
        r = self.pool.submit(func, *args, **kwargs)
        r.add_done_callback(lambda r: self._complete(callback, r))
    
    def wants_to_receive(self):
        return True
    
    def handle_receive(self):
        for callback, result in self.pending:
            callback(result)
        self.done_sock.recv(1)
        self.pending = []
```

**스레드 풀을 사용하는 서버:**

```python
def fib(n):
    if n < 2:
        return 1
    else:
        return fib(n - 1) + fib(n - 2)

class UDPFibServer(UDPServer):
    def handle_receive(self):
        msg, addr = self.sock.recvfrom(128)
        n = int(msg)
        pool.run(fib, (n,), callback=lambda r: self.respond(r, addr))
    
    def respond(self, result, addr):
        self.sock.sendto(str(result).encode('ascii'), addr)

if __name__ == '__main__':
    pool = ThreadPoolHandler(16)
    handlers = [pool, UDPFibServer(('', 16000))]
    event_loop(handlers)
```

## 11.13. 대용량 배열 송수신하기

### 문제

네트워크 연결을 통해 대용량 연속 데이터 배열을 가능한 한 적은 복사로 송수신해야 한다.

### 해결방법

```python
# zerocopy.py
def send_from(arr, dest):
    view = memoryview(arr).cast('B')
    while len(view):
        nsent = dest.send(view)
        view = view[nsent:]

def recv_into(arr, source):
    view = memoryview(arr).cast('B')
    while len(view):
        nrecv = source.recv_into(view)
        view = view[nrecv:]
```

**사용 예제:**

서버:

```python
from socket import *
import numpy

s = socket(AF_INET, SOCK_STREAM)
s.bind(('', 25000))
s.listen(1)
c, a = s.accept()

# 대용량 배열 전송
a = numpy.arange(0.0, 50000000.0)
send_from(a, c)
```

클라이언트:

```python
from socket import *
import numpy

c = socket(AF_INET, SOCK_STREAM)
c.connect(('localhost', 25000))

# 배열 수신
a = numpy.zeros(shape=50000000, dtype=float)
recv_into(a, c)
print(a[0:10])  # array([0., 1., 2., 3., 4., 5., 6., 7., 8., 9.])
```

### 논의

데이터 집약적인 분산 컴퓨팅에서는 대용량 데이터를 송수신해야 하는 경우가 많다. 하지만 일반적인 방법은 데이터를 복사하게 되어 비효율적이다.

이 레시피는 memoryview를 사용하여 복사 없이 작동한다:

```python
view = memoryview(arr).cast('B')
```

이 코드는 배열 arr을 unsigned byte의 memoryview로 캐스팅한다. memoryview는 기존 배열의 오버레이이며, 다른 타입으로 캐스팅하여 데이터를 다르게 해석할 수 있다.

소켓 함수(send(), recv_into())는 memoryview를 직접 사용하여 복사 없이 메모리에서 직접 작동한다. 각 작업 후 view는 전송/수신된 바이트 수만큼 슬라이싱되어 새로운 view를 생성하며,
이 역시 메모리 오버레이이므로 복사가 발생하지 않는다.

주의할 점은 수신자가 미리 얼마나 많은 데이터가 전송될지 알아야 한다는 것이다. 필요하다면 송신자가 먼저 크기를 전송한 후 배열 데이터를 전송하는 방식을 사용할 수 있다.