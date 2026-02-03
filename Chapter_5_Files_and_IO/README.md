# CHAPTER 5 Files and I/O

모든 프로그램은 입출력(I/O)이 필요하다. 이 챕터는 텍스트·바이너리 파일, 파일 encoding, filename과 디렉토리 조작 등 파일 작업의 일반적인 패턴을 다룬다.

## 5.1 텍스트 데이터 읽기와 쓰기

`open()` 함수의 모드를 `rt`(읽기), `wt`(쓰기), `at`(추가)로 지정하여 텍스트 파일을 처리한다.

```python
# 파일 전체를 문자열로 읽기
with open('somefile.txt', 'rt') as f:
    data = f.read()

# 줄 단위로 읽기
with open('somefile.txt', 'rt') as f:
    for line in f:
        ...

# 텍스트 쓰기
with open('somefile.txt', 'wt') as f:
    f.write(text1)

# print를 파일로 리디렉팅
with open('somefile.txt', 'wt') as f:
    print(line1, file=f)
```

기본 encoding은 시스템 기본값(보통 `utf-8`)이다. 다른 encoding이 필요하면 `encoding` 파라미터로 지정한다.

```python
with open('somefile.txt', 'rt', encoding='latin-1') as f:
    ...
```

### 주요 encoding 정리

| encoding  | 설명                                                      |
|-----------|---------------------------------------------------------|
| `ascii`   | 7비트 문자 (U+0000 ~ U+007F)                                |
| `latin-1` | 바이트 0-255를 U+0000 ~ U+00FF로 직접 매핑. decoding 오류가 발생하지 않음 |
| `utf-8`   | 웹 애플리케이션에서 가장 안전한 선택                                    |
| `utf-16`  | 16비트 기반 Unicode encoding                                |

### newline 처리와 encoding 오류

기본적으로 Python은 **universal newline** 모드로 동작하여 `\r\n`, `\r` 등을 모두 `\n`으로 변환한다. 이 변환을 비활성화하려면 `newline=''`을 지정한다.

```python
with open('somefile.txt', 'rt', newline='') as f:
    ...
```

encoding 오류가 발생하면 `errors` 파라미터로 처리 방식을 지정할 수 있다.

```python
# 잘못된 문자를 U+FFFD로 대체
f = open('sample.txt', 'rt', encoding='ascii', errors='replace')

# 잘못된 문자 무시
g = open('sample.txt', 'rt', encoding='ascii', errors='ignore')
```

## 5.2 파일로 print 출력

`print()`의 `file` 키워드 인수를 사용한다. 파일은 반드시 텍스트 모드로 열어야 한다.

```python
with open('somefile.txt', 'wt') as f:
    print('Hello World!', file=f)
```

## 5.3 구분자와 줄바꿈 문자 변경

`print()`의 `sep`과 `end` 키워드 인수를 활용한다.

```python
>>> print('ACME', 50, 91.5, sep=',')
ACME,50,91.5

>>> print('ACME', 50, 91.5, sep=',', end='!!\n')
ACME,50,91.5!!
```

`str.join()`은 문자열만 처리 가능하여 타입 변환이 추가로 필요한 반면, `print(*row, sep=',')`로는 이런 번거로움 없이 간결하게 출력할 수 있다.

```python
>>> row = ('ACME', 50, 91.5)
>>> print(*row, sep=',')
ACME,50,91.5
```

## 5.4 바이너리 데이터 읽기와 쓰기

모드를 `rb`(읽기), `wb`(쓰기)로 지정한다. 모든 데이터는 **byte string** 형태로 처리된다.

```python
with open('somefile.bin', 'rb') as f:
    data = f.read()

with open('somefile.bin', 'wb') as f:
    f.write(b'Hello World')
```

바이너리와 텍스트 간 변환이 필요하면 `decode()`/`encode()`를 사용한다.

```python
with open('somefile.bin', 'rb') as f:
    text = f.read(16).decode('utf-8')

with open('somefile.bin', 'wb') as f:
    f.write('Hello World'.encode('utf-8'))
```

`array` 등 buffer interface를 구현한 객체는 바이트 변환 없이 직접 파일에 쓸 수 있고, `readinto()` 메서드로 기존 buffer에 바이너리 데이터를 직접 읽을 수 있다.

```python
import array
nums = array.array('i', [1, 2, 3, 4])
with open('data.bin', 'wb') as f:
    f.write(nums)

# readinto로 직접 읽기
a = array.array('i', [0, 0, 0, 0])
with open('data.bin', 'rb') as f:
    f.readinto(a)
```

## 5.5 이미 존재하지 않는 파일에만 쓰기

`open()`의 모드를 `x`로 사용하면 파일이 이미 존재할 경우 `FileExistsError`가 발생한다.

```python
with open('somefile', 'xt') as f:   # 텍스트 모드
    f.write('Hello\n')

# 파일이 이미 존재하면 FileExistsError 발생
# 바이너리 모드는 'xb' 사용
```

## 5.6 문자열에 대한 파일 형태의 I/O

파일 객체 인터페이스가 필요한 코드에 문자열이나 바이너리 데이터를 넘기려면 `io.StringIO`와 `io.BytesIO`를 사용한다.

```python
import io

# 텍스트
s = io.StringIO()
s.write('Hello World\n')
print('This is a test', file=s)
print(s.getvalue())   # 'Hello World\nThis is a test\n'

# 바이너리
b = io.BytesIO()
b.write(b'binary data')
print(b.getvalue())   # b'binary data'
```

단, `StringIO`와 `BytesIO`는 실제 파일 descriptor가 없으므로 실제 파일·pipe·소켓을 요구하는 코드와는 사용할 수 없다.

## 5.7 압축된 데이터파일 읽기와 쓰기

`gzip`과 `bz2` 모듈이 각각 제공하는 `open()` 함수를 사용한다. 기본 모드는 바이너리이므로 텍스트를 원하면 반드시 `'rt'`/`'wt'`를 지정해야 한다.

```python
import gzip, bz2

# gzip 읽기/쓰기
with gzip.open('somefile.gz', 'rt') as f:
    text = f.read()
with gzip.open('somefile.gz', 'wt') as f:
    f.write(text)

# bz2 읽기/쓰기
with bz2.open('somefile.bz2', 'rt') as f:
    text = f.read()
```

압축 레벨은 `compresslevel` 키워드로 조절 가능하다 (기본값 9, 최대 압축). `gzip.open()`과 `bz2.open()`은 이미 바이너리 모드로 열린 파일 객체 위에 레이어링할 수도 있다.

```python
f = open('somefile.gz', 'rb')
with gzip.open(f, 'rt') as g:
    text = g.read()
```

## 5.8 고정 크기 레코드로 파일 순회

`iter()`와 `functools.partial()`을 조합하여 고정 크기 단위로 파일을 읽는다.

```python
from functools import partial

RECORD_SIZE = 32
with open('somefile.data', 'rb') as f:
    records = iter(partial(f.read, RECORD_SIZE), b'')
    for r in records:
        ...
```

`iter(callable, sentinel)` 형태로 사용하면 `callable`의 반환값이 `sentinel`(`b''`, 즉 EOF)과 일치할 때 반복이 종료된다. 파일 크기가 레코드 크기의 정확한 배수가 아닌
경우 마지막 레코드는 더 짧을 수 있다.

## 5.9 Mutable Buffer에 바이너리 데이터 읽기

`readinto()` 메서드를 사용하여 중간 복사 없이 기존 buffer에 직접 데이터를 읽는다.

```python
import os.path

def read_into_buffer(filename):
    buf = bytearray(os.path.getsize(filename))
    with open(filename, 'rb') as f:
        f.readinto(buf)
    return buf

buf = read_into_buffer('sample.bin')
buf[0:5] = b'Hallo'   # in-place 수정 가능
```

`memoryview`를 활용하면 zero-copy slice와 직접 수정이 가능하다.

```python
buf = bytearray(b'Hello World')
m = memoryview(buf)
m[-5:] = b'WORLD'
print(buf)   # bytearray(b'Hello WORLD')
```

`readinto()`의 반환값(실제로 읽은 바이트 수)을 반드시 확인해야 한다. 예상보다 적으면 데이터가 잘못된 것일 수 있다.

## 5.10 Memory Mapping 바이너리 파일

`mmap` 모듈을 사용하여 파일을 메모리에 매핑하면 slicing과 같은 인덱스 접근으로 파일을 읽고 쓸 수 있다.

```python
import os, mmap

def memory_map(filename, access=mmap.ACCESS_WRITE):
    size = os.path.getsize(filename)
    fd = os.open(filename, os.O_RDWR)
    return mmap.mmap(fd, size, access=access)

# 사용 예시
with memory_map('data') as m:
    print(m[0:11])          # b'Hello World'
    m[0:11] = b'Hallo World'  # 직접 수정 → 파일에 반영
```

### access 모드 정리

| 모드                  | 설명                          |
|---------------------|-----------------------------|
| `mmap.ACCESS_WRITE` | 읽기/쓰기, 변경 사항 파일에 반영 (기본값)   |
| `mmap.ACCESS_READ`  | 읽기 전용                       |
| `mmap.ACCESS_COPY`  | 변경 사항을 로컬에만 유지, 파일에 반영하지 않음 |

memory mapping은 파일 전체를 메모리로 복사하는 것이 아니라, OS가 가상 메모리 영역을 예약하고 접근 시에만 필요한 부분을 로드하는 방식이다. 여러 Python 인터프리터가 같은 파일을 memory
map하면 데이터 교환도 가능하다.

## 5.11 경로 조작 (Pathname Manipulation)

파일 경로 조작은 반드시 `os.path` 모듈을 사용해야 한다 (OS 간 호환성).

```python
import os

path = '/Users/beazley/Data/data.csv'
os.path.basename(path)          # 'data.csv'
os.path.dirname(path)           # '/Users/beazley/Data'
os.path.join('tmp', 'data', 'data.csv')   # 'tmp/data/data.csv'
os.path.expanduser('~/Data/data.csv')     # 홈 디렉토리 확장
os.path.splitext(path)          # ('...path...', '.csv')
```

## 5.12 파일 존재 여부 테스트

`os.path` 모듈의 함수들로 파일·디렉토리의 존재와 종류를 확인한다.

```python
import os

os.path.exists('/etc/passwd')       # True/False
os.path.isfile('/etc/passwd')       # 일반 파일 여부
os.path.isdir('/etc/passwd')        # 디렉토리 여부
os.path.islink('/usr/local/bin/python3')  # symlink 여부
os.path.realpath('/usr/local/bin/python3') # 실제 경로 해석

# 메타데이터
os.path.getsize('/etc/passwd')      # 파일 크기
os.path.getmtime('/etc/passwd')     # 수정 시간
```

파일 메타데이터를 가져올 때 **권한 문제**로 `PermissionError`가 발생할 수 있음을 주의하자.

## 5.13 디렉토리 리스팅

`os.listdir()`로 기본 리스팅을 가져오고, list comprehension과 `os.path` 함수로 필터링한다.

```python
import os, glob
from fnmatch import fnmatch

# 기본 리스팅
names = os.listdir('somedir')

# 특정 파일만 필터링
pyfiles = [name for name in os.listdir('somedir') if name.endswith('.py')]

# glob 패턴 매칭
pyfiles = glob.glob('somedir/*.py')

# fnmatch 패턴 매칭
pyfiles = [name for name in os.listdir('somedir') if fnmatch(name, '*.py')]
```

파일 크기·수정 시간 등 추가 메타데이터가 필요하면 `os.path.getsize()`와 `os.stat()`을 활용한다.

## 5.14 Filename Encoding 우회

기본적으로 모든 파일명은 `sys.getfilesystemencoding()` (보통 `utf-8`)로 encoding된다. 이를 우회하려면 **바이트 문자열**을 파일명으로 사용한다.

```python
import os

# 디렉토리 리스팅을 바이트로 받기
os.listdir(b'.')   # [b'jalapen\xcc\x83o.txt', ...]

# 바이트 파일명으로 파일 열기
with open(b'jalapen\xcc\x83o.txt') as f:
    print(f.read())
```

## 5.15 잘못된 Filename 출력 처리

encoding에 맞지 않는 파일명을 출력하려면 `UnicodeEncodeError`를 캐치하고 surrogate 문자를 적절히 처리해야 한다.

```python
def bad_filename(filename):
    return repr(filename)[1:-1]

for name in files:
    try:
        print(name)
    except UnicodeEncodeError:
        print(bad_filename(name))
```

대안으로 surrogate escape를 활용하여 원래 바이트를 latin-1로 다시 decoding하여 출력할 수도 있다.

```python
import sys

def bad_filename(filename):
    temp = filename.encode(sys.getfilesystemencoding(), errors='surrogateescape')
    return temp.decode('latin-1')
```

## 5.16 이미 열린 파일의 Encoding 변경

바이너리 모드 파일에 텍스트 encoding을 추가하려면 `io.TextIOWrapper()`로 감싸는다. 이미 텍스트 모드인 파일의 encoding을 바꾸려면 먼저 `detach()`로 텍스트 레이어를 제거한 후
새 레이어를 추가한다.

```python
import io, sys

# 기존 바이너리 파일에 encoding 추가
import urllib.request
u = urllib.request.urlopen('http://www.python.org')
f = io.TextIOWrapper(u, encoding='utf-8')
text = f.read()

# sys.stdout의 encoding 변경
sys.stdout = io.TextIOWrapper(sys.stdout.detach(), encoding='latin-1')
```

Python I/O 시스템은 레이어 구조로 되어 있다: `TextIOWrapper`(텍스트 encoding) → `BufferedWriter`(바이너리 버퍼) → `FileIO`(OS 레벨 파일). encoding
변경은 최상위 `TextIOWrapper` 레이어를 교체하는 것이다.

## 5.17 텍스트 파일에 바이트 쓰기

텍스트 모드 파일의 하위 buffer에 직접 바이트를 쓰면 된다.

```python
import sys
sys.stdout.buffer.write(b'Hello\n')   # 텍스트 encoding 우회
```

## 5.18 기존 File Descriptor를 파일 객체로 감싸기

OS에서 제공한 정수 file descriptor를 `open()`의 첫 번째 인수로 전달하면 Python 파일 객체로 변환된다.

```python
import os

fd = os.open('somefile.txt', os.O_WRONLY | os.O_CREAT)
f = open(fd, 'wt')
f.write('hello world\n')
f.close()   # fd도 함께 닫힘

# fd를 닫지 않고 파일 객체만 닫으려면:
f = open(fd, 'wt', closefd=False)
```

이 기법은 소켓·pipe 등 기존 I/O 채널에 파일 인터페이스를 추가하는 데 유용하다. 단, **Unix 기반 시스템에서만** 안정적으로 동작한다.

## 5.19 임시 파일과 디렉토리 생성

`tempfile` 모듈의 함수들을 사용한다. 생성과 정리가 자동으로 처리된다.

```python
from tempfile import TemporaryFile, NamedTemporaryFile, TemporaryDirectory

# 이름 없는 임시 파일 (닫히면 자동 삭제)
with TemporaryFile('w+t') as f:
    f.write('Hello World\n')
    f.seek(0)
    data = f.read()

# 이름 있는 임시 파일 (다른 프로세스에서 열 수 있음)
with NamedTemporaryFile('w+t', delete=False) as f:
    print('filename:', f.name)

# 임시 디렉토리 (닫히면 내용 포함 자동 삭제)
with TemporaryDirectory() as dirname:
    print('dirname:', dirname)
```

`prefix`, `suffix`, `dir` 키워드 인수로 임시 파일의 이름과 위치를 커스팅할 수 있다. `tempfile` 모듈은 보안적으로 가능한 한 안전하게 임시 파일을 생성한다.

## 5.20 Serial Port와의 통신

`pySerial` 패키지를 사용한다. Serial 통신은 항상 **바이너리**이므로 bytes로 처리해야 한다.

```python
import serial

ser = serial.Serial('/dev/tty.usbmodem641',
                    baudrate=9600,
                    bytesize=8,
                    parity='N',
                    stopbits=1)

ser.write(b'G1 X50 Y50\r\n')
resp = ser.readline()
```

RTS-CTS 핸드셰이킹 등 고급 기능은 `Serial()` 생성자의 키워드 인수로 간단히 활성화할 수 있다.

## 5.21 Python 객체의 Serialization

`pickle` 모듈을 사용하여 객체를 바이트 스트림으로 변환하거나 복원한다.

```python
import pickle

# 파일에 저장
with open('somefile', 'wb') as f:
    pickle.dump(data, f)

# 파일에서 복원
with open('somefile', 'rb') as f:
    data = pickle.load(f)

# 문자열로 변환/복원
s = pickle.dumps(data)
data = pickle.loads(s)
```

`pickle`은 Python에 특화된 self-describing 포맷으로, 객체 타입 정보를 자동으로 포함한다. 여러 객체도 순차적으로 dump하고 load할 수 있다.

### pickle 주의사항

| 항목               | 내용                                                                          |
|------------------|-----------------------------------------------------------------------------|
| **보안**           | `pickle.load()`는 **신뢰할 수 없는 데이터에 절대 사용하지 않아야 한다**. 임의의 코드 실행이 가능함           |
| **장기 저장**        | 소스 코드 변경 시 기존 pickle 데이터가 깨질 수 있으므로 장기 저장에 부적합. XML, CSV, JSON을 사용하는 것이 권장됨 |
| **pickle 불가 객체** | 열린 파일, 네트워크 연결, thread, process 등 외부 시스템 상태를 가진 객체는 기본적으로 pickle 불가         |
| **불가 객체 해결**     | `__getstate__()`과 `__setstate__()` 메서드를 정의하여 커스텀 pickle 로직 구현 가능            |
| **대량 배열 데이터**    | `numpy`·`array` 등 큰 배열 데이터는 pickle보다 직접 파일 저장이나 HDF5 등이 더 효율적               |

`__getstate__`/`__setstate__`를 활용한 커스텀 pickle 예시:

```python
import threading, time

class Countdown:
    def __init__(self, n):
        self.n = n
        self.thr = threading.Thread(target=self.run)
        self.thr.daemon = True
        self.thr.start()

    def run(self):
        while self.n > 0:
            print('T-minus', self.n)
            self.n -= 1
            time.sleep(5)

    def __getstate__(self):
        return self.n                  # thread 제외하고 현재 카운트만 저장

    def __setstate__(self, n):
        self.__init__(n)               # 복원 시 다시 thread를 시작
```