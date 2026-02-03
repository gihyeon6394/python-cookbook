# CHAPTER 12 Concurrency

Python은 스레드 프로그래밍, 서브프로세스 실행, 제너레이터 함수를 활용한 다양한 동시성 프로그래밍 접근 방식을 지원한다. 이 챕터는 일반적인 스레드 프로그래밍 기법과 병렬 처리 접근 방식을 포함하여 동시성
프로그래밍의 다양한 측면과 관련된 레시피를 제시한다.

## 12.1. 스레드 시작과 중지

### 문제

코드의 동시 실행을 위해 스레드를 생성하고 종료하고 싶다.

### 해결방법

**기본 스레드 생성:**

```python
import time
from threading import Thread

def countdown(n):
    while n > 0:
        print('T-minus', n)
        n -= 1
        time.sleep(5)

# 스레드 생성 및 시작
t = Thread(target=countdown, args=(10,))
t.start()

# 스레드 상태 확인
if t.is_alive():
    print('Still running')
else:
    print('Completed')

# 스레드 종료 대기
t.join()
```

**데몬 스레드:**

```python
t = Thread(target=countdown, args=(10,), daemon=True)
t.start()
# 데몬 스레드는 join할 수 없지만 메인 스레드 종료 시 자동으로 제거된다
```

**스레드 종료 제어:**

```python
class CountdownTask:
    def __init__(self):
        self._running = True
    
    def terminate(self):
        self._running = False
    
    def run(self, n):
        while self._running and n > 0:
            print('T-minus', n)
            n -= 1
            time.sleep(5)

c = CountdownTask()
t = Thread(target=c.run, args=(10,))
t.start()

c.terminate()  # 종료 신호
t.join()  # 실제 종료 대기
```

**타임아웃을 사용한 I/O 블로킹 처리:**

```python
import socket

class IOTask:
    def terminate(self):
        self._running = False
    
    def run(self, sock):
        sock.settimeout(5)  # 타임아웃 설정
        while self._running:
            try:
                data = sock.recv(8192)
                break
            except socket.timeout:
                continue
        return
```

### 논의

Global Interpreter Lock (GIL)로 인해 Python 스레드는 한 번에 하나의 스레드만 인터프리터에서 실행되도록 제한된다. 따라서 CPU 집약적인 작업에서 멀티 CPU의 병렬성을 활용하려는 경우
Python 스레드를 사용하면 안 된다. 스레드는 I/O 처리와 블로킹 작업을 수행하는 코드에서 동시 실행을 처리하는 데 훨씬 더 적합하다.

Thread 클래스를 상속하여 스레드를 정의할 수도 있지만, 이는 코드와 threading 라이브러리 간에 추가 종속성을 생성한다. 독립적인 함수로 작성하면 multiprocessing 모듈 등 다른 컨텍스트에서도
사용할 수 있다.

## 12.2. 스레드 시작 여부 확인

### 문제

스레드를 시작했지만 실제로 언제 실행되는지 알고 싶다.

### 해결방법

**Event 객체 사용:**

```python
from threading import Thread, Event
import time

def countdown(n, started_evt):
    print('countdown starting')
    started_evt.set()  # 시작 신호
    while n > 0:
        print('T-minus', n)
        n -= 1
        time.sleep(5)

started_evt = Event()

print('Launching countdown')
t = Thread(target=countdown, args=(10, started_evt))
t.start()

started_evt.wait()  # 스레드 시작 대기
print('countdown is running')
```

### 논의

Event 객체는 일회성 이벤트에 가장 적합하다. 스레드가 이벤트를 반복적으로 신호를 보내려면 Condition 객체를 사용하는 것이 좋다.

**주기적 타이머 예제:**

```python
import threading
import time

class PeriodicTimer:
    def __init__(self, interval):
        self._interval = interval
        self._flag = 0
        self._cv = threading.Condition()
    
    def start(self):
        t = threading.Thread(target=self.run)
        t.daemon = True
        t.start()
    
    def run(self):
        while True:
            time.sleep(self._interval)
            with self._cv:
                self._flag ^= 1
                self._cv.notify_all()
    
    def wait_for_tick(self):
        with self._cv:
            last_flag = self._flag
            while last_flag == self._flag:
                self._cv.wait()

# 사용 예제
ptimer = PeriodicTimer(5)
ptimer.start()

def countdown(nticks):
    while nticks > 0:
        ptimer.wait_for_tick()
        print('T-minus', nticks)
        nticks -= 1

threading.Thread(target=countdown, args=(10,)).start()
```

**Semaphore를 사용한 단일 스레드 깨우기:**

```python
import threading

def worker(n, sema):
    sema.acquire()
    print('Working', n)

sema = threading.Semaphore(0)
nworkers = 10
for n in range(nworkers):
    t = threading.Thread(target=worker, args=(n, sema))
    t.start()

# 한 번에 하나의 워커만 깨움
sema.release()  # Working 0
sema.release()  # Working 1
```

## 12.3. 스레드 간 통신

### 문제

프로그램에 여러 스레드가 있고 이들 간에 안전하게 데이터를 통신하거나 교환하고 싶다.

### 해결방법

**Queue 사용:**

```python
from queue import Queue
from threading import Thread

def producer(out_q):
    while True:
        # 데이터 생성
        data = produce_data()
        out_q.put(data)

def consumer(in_q):
    while True:
        data = in_q.get()
        # 데이터 처리
        process_data(data)

# 공유 큐 생성 및 스레드 시작
q = Queue()
t1 = Thread(target=consumer, args=(q,))
t2 = Thread(target=producer, args=(q,))
t1.start()
t2.start()
```

**센티널 값을 사용한 종료 처리:**

```python
from queue import Queue
from threading import Thread

_sentinel = object()

def producer(out_q):
    while running:
        data = produce_data()
        out_q.put(data)
    out_q.put(_sentinel)  # 종료 신호

def consumer(in_q):
    while True:
        data = in_q.get()
        if data is _sentinel:
            in_q.put(_sentinel)  # 다른 컨슈머에게 전파
            break
        process_data(data)
```

**스레드 안전 우선순위 큐:**

```python
import heapq
import threading

class PriorityQueue:
    def __init__(self):
        self._queue = []
        self._count = 0
        self._cv = threading.Condition()
    
    def put(self, item, priority):
        with self._cv:
            heapq.heappush(self._queue, (-priority, self._count, item))
            self._count += 1
            self._cv.notify()
    
    def get(self):
        with self._cv:
            while len(self._queue) == 0:
                self._cv.wait()
            return heapq.heappop(self._queue)[-1]
```

**작업 완료 확인:**

```python
from queue import Queue
from threading import Thread

def producer(out_q):
    while running:
        data = produce_data()
        out_q.put(data)

def consumer(in_q):
    while True:
        data = in_q.get()
        process_data(data)
        in_q.task_done()  # 완료 표시

q = Queue()
t1 = Thread(target=consumer, args=(q,))
t2 = Thread(target=producer, args=(q,))
t1.start()
t2.start()

q.join()  # 모든 작업 완료 대기
```

**Event를 사용한 즉각적 피드백:**

```python
from queue import Queue
from threading import Thread, Event

def producer(out_q):
    while running:
        data = produce_data()
        evt = Event()
        out_q.put((data, evt))
        evt.wait()  # 컨슈머가 처리할 때까지 대기

def consumer(in_q):
    while True:
        data, evt = in_q.get()
        process_data(data)
        evt.set()  # 완료 신호
```

### 논의

**큐 크기 제한:**

```python
q = Queue(maxsize=100)  # 최대 100개 항목
```

**비블로킹 및 타임아웃 작업:**

```python
import queue

q = queue.Queue()

# 비블로킹 get
try:
    data = q.get(block=False)
except queue.Empty:
    pass

# 비블로킹 put
try:
    q.put(item, block=False)
except queue.Full:
    pass

# 타임아웃 get
try:
    data = q.get(timeout=5.0)
except queue.Empty:
    pass
```

큐에 항목을 넣으면 항목의 복사본이 만들어지지 않고 객체 참조가 전달된다는 점에 주의해야 한다. 공유 상태가 우려된다면 불변 데이터 구조만 전달하거나 깊은 복사를 수행해야 한다.

## 12.4. 임계 영역 잠금

### 문제

프로그램이 스레드를 사용하며 경쟁 조건을 피하기 위해 코드의 임계 영역을 잠그고 싶다.

### 해결방법

```python
import threading

class SharedCounter:
    def __init__(self, initial_value=0):
        self._value = initial_value
        self._value_lock = threading.Lock()
    
    def incr(self, delta=1):
        with self._value_lock:
            self._value += delta
    
    def decr(self, delta=1):
        with self._value_lock:
            self._value -= delta
```

Lock은 with 문과 함께 사용될 때 상호 배제를 보장한다. with 문은 들여쓰기된 문장 동안 lock을 획득하고 제어 흐름이 들여쓰기 블록을 벗어나면 lock을 해제한다.

### 논의

**명시적 lock 획득/해제 (구식 방법):**

```python
def incr(self, delta=1):
    self._value_lock.acquire()
    self._value += delta
    self._value_lock.release()
```

with 문이 더 우아하고 오류가 발생하기 쉽지 않다. 특히 프로그래머가 release() 호출을 잊거나 lock을 보유한 상태에서 예외가 발생하는 경우에 그렇다.

**RLock (재진입 가능 lock):**

```python
import threading

class SharedCounter:
    _lock = threading.RLock()
    
    def __init__(self, initial_value=0):
        self._value = initial_value
    
    def incr(self, delta=1):
        with SharedCounter._lock:
            self._value += delta
    
    def decr(self, delta=1):
        with SharedCounter._lock:
            self.incr(-delta)  # 같은 스레드에서 재진입 가능
```

RLock은 같은 스레드가 여러 번 획득할 수 있는 lock이다. 주로 "모니터" 기반 잠금을 구현하는 데 사용된다.

**Semaphore를 사용한 동시성 제한:**

```python
from threading import Semaphore
import urllib.request

# 최대 5개 스레드만 동시 실행
_fetch_url_sema = Semaphore(5)

def fetch_url(url):
    with _fetch_url_sema:
        return urllib.request.urlopen(url)
```

## 12.5. 데드락 회피 잠금

### 문제

멀티스레드 프로그램에서 스레드가 데드락을 피하면서 한 번에 둘 이상의 lock을 획득해야 한다.

### 해결방법

```python
import threading
from contextlib import contextmanager

_local = threading.local()

@contextmanager
def acquire(*locks):
    # 객체 식별자로 lock 정렬
    locks = sorted(locks, key=lambda x: id(x))
    
    # 이전에 획득한 lock의 순서가 위반되지 않았는지 확인
    acquired = getattr(_local, 'acquired', [])
    if acquired and max(id(lock) for lock in acquired) >= id(locks[0]):
        raise RuntimeError('Lock Order Violation')
    
    # 모든 lock 획득
    acquired.extend(locks)
    _local.acquired = acquired
    
    try:
        for lock in locks:
            lock.acquire()
        yield
    finally:
        # 획득 역순으로 lock 해제
        for lock in reversed(locks):
            lock.release()
        del acquired[-len(locks):]
```

**사용 예제:**

```python
import threading

x_lock = threading.Lock()
y_lock = threading.Lock()

def thread_1():
    while True:
        with acquire(x_lock, y_lock):
            print('Thread-1')

def thread_2():
    while True:
        with acquire(y_lock, x_lock):  # 순서가 달라도 됨
            print('Thread-2')

t1 = threading.Thread(target=thread_1)
t1.daemon = True
t1.start()

t2 = threading.Thread(target=thread_2)
t2.daemon = True
t2.start()
```

### 논의

데드락 회피는 lock 작업을 프로그램이 데드락 상태에 진입하지 않는 방식으로 수행하는 전략이다. 항상 오름차순의 객체 ID로 lock을 획득하면 데드락을 수학적으로 회피할 수 있다.

**식사하는 철학자 문제 해결:**

```python
import threading

def philosopher(left, right):
    while True:
        with acquire(left, right):
            print(threading.currentThread(), 'eating')

NSTICKS = 5
chopsticks = [threading.Lock() for n in range(NSTICKS)]

for n in range(NSTICKS):
    t = threading.Thread(target=philosopher,
                        args=(chopsticks[n], chopsticks[(n+1) % NSTICKS]))
    t.start()
```

## 12.6. 스레드별 상태 저장

### 문제

현재 실행 중인 스레드에만 특정한 상태를 저장하고 다른 스레드에는 보이지 않게 하고 싶다.

### 해결방법

```python
from socket import socket, AF_INET, SOCK_STREAM
import threading

class LazyConnection:
    def __init__(self, address, family=AF_INET, type=SOCK_STREAM):
        self.address = address
        self.family = AF_INET
        self.type = SOCK_STREAM
        self.local = threading.local()
    
    def __enter__(self):
        if hasattr(self.local, 'sock'):
            raise RuntimeError('Already connected')
        self.local.sock = socket(self.family, self.type)
        self.local.sock.connect(self.address)
        return self.local.sock
    
    def __exit__(self, exc_ty, exc_val, tb):
        self.local.sock.close()
        del self.local.sock
```

**사용 예제:**

```python
from functools import partial

def test(conn):
    with conn as s:
        s.send(b'GET /index.html HTTP/1.0\r\n')
        s.send(b'Host: www.python.org\r\n')
        s.send(b'\r\n')
        resp = b''.join(iter(partial(s.recv, 8192), b''))
        print('Got {} bytes'.format(len(resp)))

conn = LazyConnection(('www.python.org', 80))
t1 = threading.Thread(target=test, args=(conn,))
t2 = threading.Thread(target=test, args=(conn,))
t1.start()
t2.start()
t1.join()
t2.join()
```

### 논의

threading.local()의 인스턴스는 각 스레드에 대해 별도의 인스턴스 딕셔너리를 유지한다. 모든 일반적인 인스턴스 작업은 스레드별 딕셔너리를 조작한다.

## 12.7. 스레드 풀 생성

### 문제

클라이언트를 서비스하거나 다른 종류의 작업을 수행하기 위한 워커 스레드 풀을 생성하고 싶다.

### 해결방법

**ThreadPoolExecutor 사용:**

```python
from socket import AF_INET, SOCK_STREAM, socket
from concurrent.futures import ThreadPoolExecutor

def echo_client(sock, client_addr):
    print('Got connection from', client_addr)
    while True:
        msg = sock.recv(65536)
        if not msg:
            break
        sock.sendall(msg)
    print('Client closed connection')
    sock.close()

def echo_server(addr):
    pool = ThreadPoolExecutor(128)
    sock = socket(AF_INET, SOCK_STREAM)
    sock.bind(addr)
    sock.listen(5)
    while True:
        client_sock, client_addr = sock.accept()
        pool.submit(echo_client, client_sock, client_addr)

echo_server(('', 15000))
```

**수동 스레드 풀 구현:**

```python
from socket import socket, AF_INET, SOCK_STREAM
from threading import Thread
from queue import Queue

def echo_client(q):
    sock, client_addr = q.get()
    print('Got connection from', client_addr)
    while True:
        msg = sock.recv(65536)
        if not msg:
            break
        sock.sendall(msg)
    print('Client closed connection')
    sock.close()

def echo_server(addr, nworkers):
    q = Queue()
    for n in range(nworkers):
        t = Thread(target=echo_client, args=(q,))
        t.daemon = True
        t.start()
    
    sock = socket(AF_INET, SOCK_STREAM)
    sock.bind(addr)
    sock.listen(5)
    while True:
        client_sock, client_addr = sock.accept()
        q.put((client_sock, client_addr))

echo_server(('', 15000), 128)
```

**결과 받기:**

```python
from concurrent.futures import ThreadPoolExecutor
import urllib.request

def fetch_url(url):
    u = urllib.request.urlopen(url)
    data = u.read()
    return data

pool = ThreadPoolExecutor(10)

# 작업 제출
a = pool.submit(fetch_url, 'http://www.python.org')
b = pool.submit(fetch_url, 'http://www.pypy.org')

# 결과 받기
x = a.result()
y = b.result()
```

### 논의

무제한 스레드 생성을 허용하는 프로그램은 피해야 한다. 미리 초기화된 스레드 풀을 사용하면 지원되는 동시성 양에 상한선을 설정할 수 있다.

수천 개의 스레드를 생성하는 것은 현대 시스템에서 문제가 되지 않는다. 수천 개의 스레드가 작업을 기다리며 대기 중인 것은 다른 코드의 성능에 거의 영향을 미치지 않는다.

**스택 크기 조정:**

```python
import threading

threading.stack_size(65536)  # 스택 크기를 64KB로 설정
```

## 12.8. 간단한 병렬 프로그래밍

### 문제

CPU 집약적인 작업을 수행하는 프로그램이 있고, 여러 CPU를 활용하여 더 빠르게 실행하고 싶다.

### 해결방법

**기본 예제 (순차 처리):**

```python
import gzip
import io
import glob

def find_robots(filename):
    robots = set()
    with gzip.open(filename) as f:
        for line in io.TextIOWrapper(f, encoding='ascii'):
            fields = line.split()
            if fields[6] == '/robots.txt':
                robots.add(fields[0])
    return robots

def find_all_robots(logdir):
    files = glob.glob(logdir + '/*.log.gz')
    all_robots = set()
    for robots in map(find_robots, files):
        all_robots.update(robots)
    return all_robots

if __name__ == '__main__':
    robots = find_all_robots('logs')
    for ipaddr in robots:
        print(ipaddr)
```

**병렬 처리로 수정:**

```python
import gzip
import io
import glob
from concurrent import futures

def find_robots(filename):
    robots = set()
    with gzip.open(filename) as f:
        for line in io.TextIOWrapper(f, encoding='ascii'):
            fields = line.split()
            if fields[6] == '/robots.txt':
                robots.add(fields[0])
    return robots

def find_all_robots(logdir):
    files = glob.glob(logdir + '/*.log.gz')
    all_robots = set()
    with futures.ProcessPoolExecutor() as pool:
        for robots in pool.map(find_robots, files):
            all_robots.update(robots)
    return all_robots

if __name__ == '__main__':
    robots = find_all_robots('logs')
    for ipaddr in robots:
        print(ipaddr)
```

### 논의

**기본 사용법:**

```python
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor() as pool:
    # pool을 사용한 병렬 작업
    pass
```

**pool.map() 사용:**

```python
def work(x):
    return result

# 비병렬 코드
results = map(work, data)

# 병렬 구현
with ProcessPoolExecutor() as pool:
    results = pool.map(work, data)
```

**pool.submit() 사용:**

```python
def work(x):
    return result

with ProcessPoolExecutor() as pool:
    future_result = pool.submit(work, arg)
    r = future_result.result()  # 블로킹
```

**콜백 함수:**

```python
def when_done(r):
    print('Got:', r.result())

with ProcessPoolExecutor() as pool:
    future_result = pool.submit(work, arg)
    future_result.add_done_callback(when_done)
```

**중요 고려사항:**

- 독립적인 부분으로 쉽게 분해할 수 있는 문제에만 잘 작동한다
- 작업은 단순 함수 형태로 제출해야 한다
- 함수 인수와 반환 값은 pickle과 호환되어야 한다
- 함수는 지속적인 상태를 유지하거나 부작용이 없어야 한다
- Unix에서는 fork() 시스템 호출로 프로세스 풀이 생성된다
- 프로세스 풀과 스레드를 함께 사용할 때는 주의가 필요하다

## 12.9. GIL 다루기

### 문제

Global Interpreter Lock (GIL)에 대해 들었고, 멀티스레드 프로그램의 성능에 영향을 미칠까 걱정된다.

### 해결방법

GIL은 주어진 시간에 하나의 Python 스레드만 실행되도록 허용한다. 가장 눈에 띄는 영향은 멀티스레드 Python 프로그램이 여러 CPU 코어를 완전히 활용할 수 없다는 것이다.

**중요한 점:**

- GIL은 CPU 집약적인 프로그램에만 영향을 미친다
- I/O 위주의 프로그램(네트워크 통신 등)에서는 스레드가 적합한 선택이다
- 수천 개의 Python 스레드를 생성해도 문제없다

**CPU 집약적 작업을 위한 해결책 1: multiprocessing 사용:**

```python
# 스레드 코드
def some_work(args):
    return result

def some_thread():
    while True:
        r = some_work(args)

# 프로세스 풀로 수정
pool = None

def some_thread():
    while True:
        r = pool.apply(some_work, (args,))

if __name__ == '__main__':
    import multiprocessing
    pool = multiprocessing.Pool()
```

**CPU 집약적 작업을 위한 해결책 2: C 확장:**

```c
#include "Python.h"

PyObject *pyfunc(PyObject *self, PyObject *args) {
    ...
    Py_BEGIN_ALLOW_THREADS
    // 스레드된 C 코드
    ...
    Py_END_ALLOW_THREADS
    ...
}
```

ctypes는 기본적으로 C 호출 시 GIL을 해제한다.

### 논의

많은 프로그래머가 스레드 성능 문제에 직면했을 때 모든 문제를 GIL 탓으로 돌린다. 그러나 실제로는 다른 원인일 수 있다(예: 정체된 DNS 조회). GIL이 문제인지 확인하려면 코드를 연구해야 한다.

프로세스 풀을 사용할 때는 데이터 직렬화와 다른 Python 인터프리터와의 통신이 필요하다는 점을 인식해야 한다. 작업은 충분히 커서 추가 통신 오버헤드를 보상할 수 있어야 한다.

## 12.10. Actor 태스크 정의

### 문제

"액터 모델"의 "액터"와 유사한 동작을 가진 태스크를 정의하고 싶다.

### 해결방법

```python
from queue import Queue
from threading import Thread, Event

class ActorExit(Exception):
    pass

class Actor:
    def __init__(self):
        self._mailbox = Queue()
    
    def send(self, msg):
        self._mailbox.put(msg)
    
    def recv(self):
        msg = self._mailbox.get()
        if msg is ActorExit:
            raise ActorExit()
        return msg
    
    def close(self):
        self.send(ActorExit)
    
    def start(self):
        self._terminated = Event()
        t = Thread(target=self._bootstrap)
        t.daemon = True
        t.start()
    
    def _bootstrap(self):
        try:
            self.run()
        except ActorExit:
            pass
        finally:
            self._terminated.set()
    
    def join(self):
        self._terminated.wait()
    
    def run(self):
        while True:
            msg = self.recv()

# 사용 예제
class PrintActor(Actor):
    def run(self):
        while True:
            msg = self.recv()
            print('Got:', msg)

p = PrintActor()
p.start()
p.send('Hello')
p.send('World')
p.close()
p.join()
```

**제너레이터를 사용한 간단한 액터:**

```python
def print_actor():
    while True:
        try:
            msg = yield
            print('Got:', msg)
        except GeneratorExit:
            print('Actor terminating')

p = print_actor()
next(p)  # yield로 진행
p.send('Hello')
p.send('World')
p.close()
```

### 논의

**태그된 메시지 처리:**

```python
class TaggedActor(Actor):
    def run(self):
        while True:
            tag, *payload = self.recv()
            getattr(self, 'do_' + tag)(*payload)
    
    def do_A(self, x):
        print('Running A', x)
    
    def do_B(self, x, y):
        print('Running B', x, y)

a = TaggedActor()
a.start()
a.send(('A', 1))      # do_A(1) 호출
a.send(('B', 2, 3))   # do_B(2, 3) 호출
```

**결과를 반환하는 워커:**

```python
from threading import Event

class Result:
    def __init__(self):
        self._evt = Event()
        self._result = None
    
    def set_result(self, value):
        self._result = value
        self._evt.set()
    
    def result(self):
        self._evt.wait()
        return self._result

class Worker(Actor):
    def submit(self, func, *args, **kwargs):
        r = Result()
        self.send((func, args, kwargs, r))
        return r
    
    def run(self):
        while True:
            func, args, kwargs, r = self.recv()
            r.set_result(func(*args, **kwargs))

worker = Worker()
worker.start()
r = worker.submit(pow, 2, 3)
print(r.result())  # 8
```

## 12.11. Publish/Subscribe 메시징 구현

### 문제

통신 스레드 기반 프로그램이 있고 publish/subscribe 메시징을 구현하고 싶다.

### 해결방법

```python
from collections import defaultdict

class Exchange:
    def __init__(self):
        self._subscribers = set()
    
    def attach(self, task):
        self._subscribers.add(task)
    
    def detach(self, task):
        self._subscribers.remove(task)
    
    def send(self, msg):
        for subscriber in self._subscribers:
            subscriber.send(msg)

_exchanges = defaultdict(Exchange)

def get_exchange(name):
    return _exchanges[name]
```

**사용 예제:**

```python
class Task:
    def send(self, msg):
        pass

task_a = Task()
task_b = Task()

exc = get_exchange('name')

# 구독
exc.attach(task_a)
exc.attach(task_b)

# 메시지 전송
exc.send('msg1')
exc.send('msg2')

# 구독 취소
exc.detach(task_a)
exc.detach(task_b)
```

### 논의

**컨텍스트 매니저 추가:**

```python
from contextlib import contextmanager

class Exchange:
    def __init__(self):
        self._subscribers = set()
    
    def attach(self, task):
        self._subscribers.add(task)
    
    def detach(self, task):
        self._subscribers.remove(task)
    
    @contextmanager
    def subscribe(self, *tasks):
        for task in tasks:
            self.attach(task)
        try:
            yield
        finally:
            for task in tasks:
                self.detach(task)
    
    def send(self, msg):
        for subscriber in self._subscribers:
            subscriber.send(msg)

# 사용
exc = get_exchange('name')
with exc.subscribe(task_a, task_b):
    exc.send('msg1')
    exc.send('msg2')
# task_a와 task_b가 여기서 detach됨
```

**진단 도구:**

```python
class DisplayMessages:
    def __init__(self):
        self.count = 0
    
    def send(self, msg):
        self.count += 1
        print('msg[{}]: {!r}'.format(self.count, msg))

exc = get_exchange('name')
d = DisplayMessages()
exc.attach(d)
```

## 12.12. 스레드 대안으로 제너레이터 사용

### 문제

시스템 스레드의 대안으로 제너레이터(코루틴)를 사용하여 동시성을 구현하고 싶다. 이를 사용자 레벨 스레딩 또는 그린 스레딩이라고도 한다.

### 해결방법

**간단한 태스크 스케줄러:**

```python
from collections import deque

def countdown(n):
    while n > 0:
        print('T-minus', n)
        yield
        n -= 1
    print('Blastoff!')

def countup(n):
    x = 0
    while x < n:
        print('Counting up', x)
        yield
        x += 1

class TaskScheduler:
    def __init__(self):
        self._task_queue = deque()
    
    def new_task(self, task):
        self._task_queue.append(task)
    
    def run(self):
        while self._task_queue:
            task = self._task_queue.popleft()
            try:
                next(task)
                self._task_queue.append(task)
            except StopIteration:
                pass

sched = TaskScheduler()
sched.new_task(countdown(10))
sched.new_task(countdown(5))
sched.new_task(countup(15))
sched.run()
```

**액터 스케줄러:**

```python
from collections import deque

class ActorScheduler:
    def __init__(self):
        self._actors = {}
        self._msg_queue = deque()
    
    def new_actor(self, name, actor):
        self._msg_queue.append((actor, None))
        self._actors[name] = actor
    
    def send(self, name, msg):
        actor = self._actors.get(name)
        if actor:
            self._msg_queue.append((actor, msg))
    
    def run(self):
        while self._msg_queue:
            actor, msg = self._msg_queue.popleft()
            try:
                actor.send(msg)
            except StopIteration:
                pass

if __name__ == '__main__':
    def printer():
        while True:
            msg = yield
            print('Got:', msg)
    
    def counter(sched):
        while True:
            n = yield
            if n == 0:
                break
            sched.send('printer', n)
            sched.send('counter', n-1)
    
    sched = ActorScheduler()
    sched.new_actor('printer', printer())
    sched.new_actor('counter', counter(sched))
    sched.send('counter', 10000)
    sched.run()
```

**고급 네트워크 애플리케이션:**

```python
from collections import deque
from select import select

class YieldEvent:
    def handle_yield(self, sched, task):
        pass
    
    def handle_resume(self, sched, task):
        pass

class Scheduler:
    def __init__(self):
        self._numtasks = 0
        self._ready = deque()
        self._read_waiting = {}
        self._write_waiting = {}
    
    def _iopoll(self):
        rset, wset, eset = select(self._read_waiting,
                                   self._write_waiting, [])
        for r in rset:
            evt, task = self._read_waiting.pop(r)
            evt.handle_resume(self, task)
        for w in wset:
            evt, task = self._write_waiting.pop(w)
            evt.handle_resume(self, task)
    
    def new(self, task):
        self._ready.append((task, None))
        self._numtasks += 1
    
    def add_ready(self, task, msg=None):
        self._ready.append((task, msg))
    
    def _read_wait(self, fileno, evt, task):
        self._read_waiting[fileno] = (evt, task)
    
    def _write_wait(self, fileno, evt, task):
        self._write_waiting[fileno] = (evt, task)
    
    def run(self):
        while self._numtasks:
            if not self._ready:
                self._iopoll()
            task, msg = self._ready.popleft()
            try:
                r = task.send(msg)
                if isinstance(r, YieldEvent):
                    r.handle_yield(self, task)
                else:
                    raise RuntimeError('unrecognized yield event')
            except StopIteration:
                self._numtasks -= 1
```

### 논의

제너레이터 기반 동시성을 프로그래밍할 때 더 일반적인 yield 형태를 사용한다:

```python
def some_generator():
    result = yield data
```

이러한 방식으로 yield를 사용하는 함수를 "코루틴"이라고 한다. 스케줄러 내에서 yield 문은 다음과 같이 처리된다:

```python
f = some_generator()
result = None
while True:
    try:
        data = f.send(result)
        result = ... # 계산 수행
    except StopIteration:
        break
```

yield from 문은 서브루틴이나 프로시저로 작동하는 코루틴을 구현하는 데 사용된다. 제어가 투명하게 새 함수로 전달된다.

**주요 제한사항:**

- 스레드가 제공하는 이점을 얻지 못한다
- CPU 집약적이거나 I/O를 블로킹하는 코드는 전체 태스크 스케줄러를 일시 중단한다
- 대부분의 Python 라이브러리는 제너레이터 기반 스레딩과 잘 작동하도록 작성되지 않았다

## 12.13. 여러 스레드 큐 폴링

### 문제

스레드 큐 모음이 있고, 네트워크 연결 모음에서 들어오는 데이터를 폴링하는 것과 같은 방식으로 들어오는 항목을 폴링하고 싶다.

### 해결방법

```python
import queue
import socket
import os

class PollableQueue(queue.Queue):
    def __init__(self):
        super().__init__()
        # 연결된 소켓 쌍 생성
        if os.name == 'posix':
            self._putsocket, self._getsocket = socket.socketpair()
        else:
            # Windows 호환성
            server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            server.bind(('127.0.0.1', 0))
            server.listen(1)
            self._putsocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self._putsocket.connect(server.getsockname())
            self._getsocket, _ = server.accept()
            server.close()
    
    def fileno(self):
        return self._getsocket.fileno()
    
    def put(self, item):
        super().put(item)
        self._putsocket.send(b'x')
    
    def get(self):
        self._getsocket.recv(1)
        return super().get()
```

**사용 예제:**

```python
import select
import threading

def consumer(queues):
    while True:
        can_read, _, _ = select.select(queues, [], [])
        for r in can_read:
            item = r.get()
            print('Got:', item)

q1 = PollableQueue()
q2 = PollableQueue()
q3 = PollableQueue()

t = threading.Thread(target=consumer, args=([q1, q2, q3],))
t.daemon = True
t.start()

# 큐에 데이터 공급
q1.put(1)
q2.put(10)
q3.put('hello')
q2.put(15)
```

### 논의

이 솔루션은 큐를 소켓과 동등한 상태에 놓는다. 단일 select() 호출로 둘 다의 활동을 폴링할 수 있다. 타임아웃이나 다른 시간 기반 해킹을 사용하여 주기적으로 확인할 필요가 없다.

## 12.14. Unix에서 데몬 프로세스 실행

### 문제

Unix나 Unix 계열 시스템에서 적절한 데몬 프로세스로 실행되는 프로그램을 작성하고 싶다.

### 해결방법

```python
#!/usr/bin/env python3
import os
import sys
import atexit
import signal

def daemonize(pidfile, *, stdin='/dev/null',
              stdout='/dev/null', stderr='/dev/null'):
    if os.path.exists(pidfile):
        raise RuntimeError('Already running')
    
    # 첫 번째 fork (부모로부터 분리)
    try:
        if os.fork() > 0:
            raise SystemExit(0)
    except OSError as e:
        raise RuntimeError('fork #1 failed.')
    
    os.chdir('/')
    os.umask(0)
    os.setsid()
    
    # 두 번째 fork (세션 리더십 포기)
    try:
        if os.fork() > 0:
            raise SystemExit(0)
    except OSError as e:
        raise RuntimeError('fork #2 failed.')
    
    # I/O 버퍼 플러시
    sys.stdout.flush()
    sys.stderr.flush()
    
    # stdin, stdout, stderr 파일 디스크립터 교체
    with open(stdin, 'rb', 0) as f:
        os.dup2(f.fileno(), sys.stdin.fileno())
    with open(stdout, 'ab', 0) as f:
        os.dup2(f.fileno(), sys.stdout.fileno())
    with open(stderr, 'ab', 0) as f:
        os.dup2(f.fileno(), sys.stderr.fileno())
    
    # PID 파일 작성
    with open(pidfile, 'w') as f:
        print(os.getpid(), file=f)
    
    # 종료/시그널 시 PID 파일 제거
    atexit.register(lambda: os.remove(pidfile))
    
    # 종료 시그널 핸들러
    def sigterm_handler(signo, frame):
        raise SystemExit(1)
    
    signal.signal(signal.SIGTERM, sigterm_handler)

def main():
    import time
    sys.stdout.write('Daemon started with pid {}\n'.format(os.getpid()))
    while True:
        sys.stdout.write('Daemon Alive! {}\n'.format(time.ctime()))
        time.sleep(10)

if __name__ == '__main__':
    PIDFILE = '/tmp/daemon.pid'
    
    if len(sys.argv) != 2:
        print('Usage: {} [start|stop]'.format(sys.argv[0]), file=sys.stderr)
        raise SystemExit(1)
    
    if sys.argv[1] == 'start':
        try:
            daemonize(PIDFILE,
                     stdout='/tmp/daemon.log',
                     stderr='/tmp/daemon.log')
        except RuntimeError as e:
            print(e, file=sys.stderr)
            raise SystemExit(1)
        main()
    
    elif sys.argv[1] == 'stop':
        if os.path.exists(PIDFILE):
            with open(PIDFILE) as f:
                os.kill(int(f.read()), signal.SIGTERM)
        else:
            print('Not running', file=sys.stderr)
            raise SystemExit(1)
    
    else:
        print('Unknown command {!r}'.format(sys.argv[1]), file=sys.stderr)
        raise SystemExit(1)
```

**사용:**

```bash
# 데몬 시작
bash % daemon.py start
bash % cat /tmp/daemon.pid
2882
bash % tail -f /tmp/daemon.log
Daemon started with pid 2882
Daemon Alive! Fri Oct 12 13:45:37 2012

# 데몬 중지
bash % daemon.py stop
```

### 논의

데몬 프로세스를 생성하는 단계는 다소 복잡하지만 일반적인 아이디어는 다음과 같다:

1. 첫 번째 os.fork() 작업과 부모의 즉시 종료로 부모 프로세스로부터 분리
2. os.setsid() 호출로 완전히 새로운 프로세스 세션 생성
3. os.chdir()와 os.umask(0)로 현재 작업 디렉토리 변경 및 파일 모드 마스크 재설정
4. 두 번째 os.fork()로 새 제어 터미널을 획득할 수 있는 능력 포기
5. 표준 I/O 스트림을 사용자가 지정한 파일을 가리키도록 재초기화
6. PID 파일 작성 및 종료 시 제거 등록
7. SIGTERM 시그널 핸들러 정의

데몬 프로세스 작성에 대한 자세한 정보는 W. Richard Stevens와 Stephen A. Rago의 "Advanced Programming in the UNIX Environment, 2nd Edition"을
참조하라.