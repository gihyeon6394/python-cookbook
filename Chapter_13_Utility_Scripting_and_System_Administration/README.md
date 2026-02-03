# CHAPTER 13 Utility Scripting and System Administration

많은 사람들이 Python을 셸 스크립트의 대체품으로 사용하여 파일 조작, 시스템 구성 등과 같은 일반적인 시스템 작업을 자동화한다. 이 챕터의 주요 목표는 스크립트 작성 시 마주치는 일반적인 작업과 관련된 기능을
설명하는 것이다.

## 13.1. 리다이렉션, 파이프 또는 입력 파일을 통한 스크립트 입력 받기

### 문제

작성한 스크립트가 사용자에게 가장 쉬운 메커니즘을 사용하여 입력을 받을 수 있도록 하고 싶다. 여기에는 명령의 출력을 스크립트로 파이핑하기, 파일을 스크립트로 리다이렉션하기, 또는 명령줄에서 파일 이름이나 파일 이름
목록을 전달하는 것이 포함된다.

### 해결방법

```python
#!/usr/bin/env python3
import fileinput

with fileinput.input() as f_input:
    for line in f_input:
        print(line, end='')
```

이 스크립트를 filein.py로 저장하고 실행 가능하게 만들면 다음과 같이 사용할 수 있다:

```bash
$ ls | ./filein.py              # 디렉토리 목록을 stdout으로 출력
$ ./filein.py /etc/passwd       # /etc/passwd를 stdout으로 읽음
$ ./filein.py < /etc/passwd     # /etc/passwd를 stdout으로 읽음
```

### 논의

**파일명과 줄 번호 포함:**

```python
import fileinput

with fileinput.input('/etc/passwd') as f:
    for line in f:
        print(f.filename(), f.lineno(), line, end='')
```

fileinput.input() 함수는 FileInput 클래스의 인스턴스를 생성하고 반환한다. 이 인스턴스는 컨텍스트 매니저로도 사용할 수 있어 파일이 더 이상 사용되지 않을 때 자동으로 닫힌다.

## 13.2. 오류 메시지와 함께 프로그램 종료하기

### 문제

프로그램을 표준 오류로 메시지를 출력하고 0이 아닌 상태 코드를 반환하며 종료하고 싶다.

### 해결방법

```python
raise SystemExit('It failed!')
```

이렇게 하면 제공된 메시지가 sys.stderr로 출력되고 프로그램이 상태 코드 1로 종료된다.

### 논의

다음과 같이 작성하는 것보다 훨씬 간단하다:

```python
import sys
sys.stderr.write('It failed!\n')
raise SystemExit(1)
```

SystemExit()에 메시지를 전달하기만 하면 import나 sys.stderr로의 쓰기 같은 추가 단계가 필요 없다.

## 13.3. 명령줄 옵션 파싱

### 문제

명령줄에서 제공된 옵션(sys.argv에 있는)을 파싱하는 프로그램을 작성하고 싶다.

### 해결방법

```python
# search.py
import argparse

parser = argparse.ArgumentParser(description='Search some files')

parser.add_argument(dest='filenames', metavar='filename', nargs='*')

parser.add_argument('-p', '--pat', metavar='pattern', required=True,
                   dest='patterns', action='append',
                   help='text pattern to search for')

parser.add_argument('-v', dest='verbose', action='store_true',
                   help='verbose mode')

parser.add_argument('-o', dest='outfile', action='store',
                   help='output file')

parser.add_argument('--speed', dest='speed', action='store',
                   choices={'slow', 'fast'}, default='slow',
                   help='search speed')

args = parser.parse_args()

print(args.filenames)
print(args.patterns)
print(args.verbose)
print(args.outfile)
print(args.speed)
```

**사용 예제:**

```bash
bash % python3 search.py -h
usage: search.py [-h] [-p pattern] [-v] [-o OUTFILE] [--speed {slow,fast}]
                 [filename [filename ...]]

bash % python3 search.py -v -p spam --pat=eggs foo.txt bar.txt
filenames = ['foo.txt', 'bar.txt']
patterns = ['spam', 'eggs']
verbose = True
outfile = None
speed = slow
```

### 논의

**주요 add_argument() 옵션:**

- `dest`: 파싱 결과가 저장될 속성 이름
- `metavar`: 도움말 메시지 생성 시 사용
- `action`: 인수와 관련된 처리 방식 지정
    - `store`: 값 저장
    - `append`: 여러 인수 값을 리스트로 수집
    - `store_true`: Boolean 플래그 설정
- `required`: 인수가 반드시 제공되어야 하는지 여부
- `choices`: 가능한 값들의 집합
- `default`: 기본값

**추가 명령줄 인수 수집:**

```python
parser.add_argument(dest='filenames', metavar='filename', nargs='*')
```

**Boolean 플래그:**

```python
parser.add_argument('-v', dest='verbose', action='store_true',
                   help='verbose mode')
```

**반복 가능한 인수:**

```python
parser.add_argument('-p', '--pat', metavar='pattern', required=True,
                   dest='patterns', action='append',
                   help='text pattern to search for')
```

## 13.4. 런타임에 비밀번호 입력받기

### 문제

비밀번호가 필요한 스크립트를 작성했지만, 스크립트는 대화형 사용을 위한 것이므로 스크립트에 하드코딩하는 대신 사용자에게 비밀번호를 입력받고 싶다.

### 해결방법

```python
import getpass

user = getpass.getuser()
passwd = getpass.getpass()

if svc_login(user, passwd):
    print('Yay!')
else:
    print('Boo!')
```

### 논의

getpass.getuser()는 사용자의 로그인 이름을 셸 환경에 따라 반환하며, 프롬프트를 표시하지 않는다.

**명시적으로 사용자 이름 입력받기:**

```python
user = input('Enter your username: ')
```

일부 시스템에서는 getpass() 메서드가 입력된 비밀번호 숨김을 지원하지 않을 수 있다. 이 경우 Python은 비밀번호가 평문으로 표시될 것임을 경고한다.

## 13.5. 터미널 크기 가져오기

### 문제

프로그램의 출력을 올바르게 포맷하기 위해 터미널 크기를 가져와야 한다.

### 해결방법

```python
import os

sz = os.get_terminal_size()
print(sz)  # os.terminal_size(columns=80, lines=24)
print(sz.columns)  # 80
print(sz.lines)    # 24
```

### 논의

환경 변수를 읽거나 ioctl()과 TTY를 사용하는 저수준 시스템 호출을 실행하는 등 터미널 크기를 얻는 다른 많은 방법이 있지만, 이 간단한 호출 하나로 충분하다.

## 13.6. 외부 명령 실행 및 출력 가져오기

### 문제

외부 명령을 실행하고 그 출력을 Python 문자열로 수집하고 싶다.

### 해결방법

```python
import subprocess

out_bytes = subprocess.check_output(['netstat', '-a'])
out_text = out_bytes.decode('utf-8')
```

**오류 처리:**

```python
try:
    out_bytes = subprocess.check_output(['cmd', 'arg1', 'arg2'])
except subprocess.CalledProcessError as e:
    out_bytes = e.output      # 오류 전 생성된 출력
    code = e.returncode       # 반환 코드
```

**표준 출력과 오류 모두 수집:**

```python
out_bytes = subprocess.check_output(['cmd', 'arg1', 'arg2'],
                                   stderr=subprocess.STDOUT)
```

**타임아웃 설정:**

```python
try:
    out_bytes = subprocess.check_output(['cmd', 'arg1', 'arg2'], timeout=5)
except subprocess.TimeoutExpired as e:
    pass
```

**셸 사용:**

```python
out_bytes = subprocess.check_output('grep python | wc > out', shell=True)
```

셸에서 명령을 실행하면 보안 위험이 있을 수 있다. 사용자 입력에서 파생된 인수의 경우 shlex.quote() 함수를 사용하여 적절하게 인용해야 한다.

### 논의

**고급 통신을 위한 Popen 사용:**

```python
import subprocess

text = b'''
hello world
this is a test
goodbye
'''

p = subprocess.Popen(['wc'],
                    stdout=subprocess.PIPE,
                    stdin=subprocess.PIPE)

stdout, stderr = p.communicate(text)

out = stdout.decode('utf-8')
err = stderr.decode('utf-8')
```

subprocess 모듈은 적절한 TTY와 상호작용할 것으로 예상되는 외부 명령과 통신하는 데 적합하지 않다. 예를 들어, 사용자에게 비밀번호를 입력하라고 요청하는 작업(ssh 세션 등)을 자동화하는 데 사용할 수
없다. 이를 위해서는 pexpect 같은 서드파티 모듈을 사용해야 한다.

## 13.7. 파일 및 디렉토리 복사 또는 이동

### 문제

파일과 디렉토리를 복사하거나 이동해야 하지만, 셸 명령을 호출하고 싶지 않다.

### 해결방법

```python
import shutil

# 파일 복사 (cp src dst)
shutil.copy(src, dst)

# 메타데이터 보존하며 복사 (cp -p src dst)
shutil.copy2(src, dst)

# 디렉토리 트리 복사 (cp -R src dst)
shutil.copytree(src, dst)

# 이동 (mv src dst)
shutil.move(src, dst)
```

**심볼릭 링크 처리:**

```python
# 심볼릭 링크를 따라가지 않고 복사
shutil.copy2(src, dst, follow_symlinks=False)

# 디렉토리 복사 시 심볼릭 링크 보존
shutil.copytree(src, dst, symlinks=True)
```

**파일 무시:**

```python
def ignore_pyc_files(dirname, filenames):
    return [name for name in filenames if name.endswith('.pyc')]

shutil.copytree(src, dst, ignore=ignore_pyc_files)

# 또는 패턴 사용
shutil.copytree(src, dst, ignore=shutil.ignore_patterns('*~', '*.pyc'))
```

### 논의

**os.path 함수 활용:**

```python
import os.path

filename = '/Users/guido/programs/spam.py'

os.path.basename(filename)  # 'spam.py'
os.path.dirname(filename)   # '/Users/guido/programs'
os.path.split(filename)     # ('/Users/guido/programs', 'spam.py')
os.path.join('/new/dir', os.path.basename(filename))  # '/new/dir/spam.py'
os.path.expanduser('~/guido/programs/spam.py')  # '/Users/guido/programs/spam.py'
```

**오류 처리:**

```python
try:
    shutil.copytree(src, dst)
except shutil.Error as e:
    for src, dst, msg in e.args[0]:
        print(dst, src, msg)
```

깨진 심볼릭 링크를 무시하려면 ignore_dangling_symlinks=True 키워드 인수를 사용한다.

## 13.8. 아카이브 생성 및 압축 해제

### 문제

일반적인 형식(예: .tar, .tgz, .zip)으로 아카이브를 생성하거나 압축을 해제해야 한다.

### 해결방법

```python
import shutil

# 아카이브 압축 해제
shutil.unpack_archive('Python-3.3.0.tgz')

# 아카이브 생성
shutil.make_archive('py33', 'zip', 'Python-3.3.0')
# 반환: '/Users/beazley/Downloads/py33.zip'

# 지원되는 아카이브 형식 확인
shutil.get_archive_formats()
# [('bztar', "bzip2'ed tar-file"), ('gztar', "gzip'ed tar-file"),
#  ('tar', 'uncompressed tar file'), ('zip', 'ZIP file')]
```

### 논의

Python에는 다양한 아카이브 형식의 저수준 세부 사항을 처리하는 다른 라이브러리 모듈(tarfile, zipfile, gzip, bz2 등)이 있다. 하지만 아카이브를 만들거나 추출하는 것만이 목적이라면
shutil의 고수준 함수를 사용하는 것이 좋다.

## 13.9. 이름으로 파일 찾기

### 문제

파일 이름 변경 스크립트나 로그 아카이버 유틸리티처럼 파일을 찾는 작업을 포함하는 스크립트를 작성해야 한다.

### 해결방법

```python
#!/usr/bin/env python3.3
import os

def findfile(start, name):
    for relpath, dirs, files in os.walk(start):
        if name in files:
            full_path = os.path.join(start, relpath, name)
            print(os.path.normpath(os.path.abspath(full_path)))

if __name__ == '__main__':
    findfile(sys.argv[1], sys.argv[2])
```

**사용:**

```bash
bash % ./findfile.py . myfile.txt
```

### 논의

os.walk() 메서드는 디렉토리 계층을 순회하며 각 디렉토리에 대해 3-튜플을 반환한다:

- 검사 중인 디렉토리의 상대 경로
- 해당 디렉토리의 모든 디렉토리 이름 목록
- 해당 디렉토리의 파일 이름 목록

**최근 수정된 파일 찾기:**

```python
#!/usr/bin/env python3.3
import os
import time

def modified_within(top, seconds):
    now = time.time()
    for path, dirs, files in os.walk(top):
        for name in files:
            fullpath = os.path.join(path, name)
            if os.path.exists(fullpath):
                mtime = os.path.getmtime(fullpath)
                if mtime > (now - seconds):
                    print(fullpath)

if __name__ == '__main__':
    import sys
    if len(sys.argv) != 3:
        print('Usage: {} dir seconds'.format(sys.argv[0]))
        raise SystemExit(1)
    modified_within(sys.argv[1], float(sys.argv[2]))
```

## 13.10. 구성 파일 읽기

### 문제

일반적인 .ini 구성 파일 형식으로 작성된 구성 파일을 읽고 싶다.

### 해결방법

**구성 파일 예제 (config.ini):**

```ini
; config.ini
[installation]
library=%(prefix)s/lib
include=%(prefix)s/include
bin=%(prefix)s/bin
prefix=/usr/local

[debug]
log_errors=true
show_warnings=False

[server]
port: 8080
nworkers: 32
pid-file=/tmp/spam.pid
root=/www/root
signature:
    =================================
    Brought to you by the Python Cookbook
    =================================
```

**읽기:**

```python
from configparser import ConfigParser

cfg = ConfigParser()
cfg.read('config.ini')

print(cfg.sections())  # ['installation', 'debug', 'server']
print(cfg.get('installation', 'library'))  # '/usr/local/lib'
print(cfg.getboolean('debug', 'log_errors'))  # True
print(cfg.getint('server', 'port'))  # 8080
print(cfg.getint('server', 'nworkers'))  # 32
print(cfg.get('server', 'signature'))
```

**수정 및 쓰기:**

```python
cfg.set('server', 'port', '9000')
cfg.set('debug', 'log_errors', 'False')

import sys
cfg.write(sys.stdout)
```

### 논의

**주요 특징:**

- 대소문자를 구분하지 않는 이름
- 유연한 구문 (= 또는 : 모두 가능)
- Boolean 값의 다양한 표현 (true, TRUE, Yes, 1 모두 동일)
- 변수 치환 지원

**여러 구성 파일 병합:**

```python
# 기본 구성
cfg.get('installation', 'prefix')  # '/usr/local'

# 사용자별 구성 병합
import os
cfg.read(os.path.expanduser('~/.config.ini'))

cfg.get('installation', 'prefix')  # '/Users/beazley/test'
cfg.get('installation', 'library')  # '/Users/beazley/test/lib'
```

변수 보간은 가능한 한 늦게 수행되므로 prefix 변수의 재정의가 library 같은 다른 관련 변수에 영향을 미친다.

## 13.11. 간단한 스크립트에 로깅 추가

### 문제

스크립트와 간단한 프로그램이 로그 파일에 진단 정보를 작성하도록 하고 싶다.

### 해결방법

```python
import logging

def main():
    # 로깅 시스템 구성
    logging.basicConfig(
        filename='app.log',
        level=logging.ERROR
    )
    
    hostname = 'www.python.org'
    item = 'spam'
    filename = 'data.csv'
    mode = 'r'
    
    # 로깅 호출 예제
    logging.critical('Host %s unknown', hostname)
    logging.error("Couldn't find %r", item)
    logging.warning('Feature is deprecated')
    logging.info('Opening file %r, mode=%r', filename, mode)
    logging.debug('Got here')

if __name__ == '__main__':
    main()
```

다섯 가지 로깅 호출(critical(), error(), warning(), info(), debug())은 심각도 수준을 나타낸다.

**출력 형식 변경:**

```python
logging.basicConfig(
    filename='app.log',
    level=logging.WARNING,
    format='%(levelname)s:%(asctime)s:%(message)s')
```

**결과:**

```
CRITICAL:2012-11-20 12:27:13,595:Host www.python.org unknown
ERROR:2012-11-20 12:27:13,595:Could not find 'spam'
WARNING:2012-11-20 12:27:13,595:Feature is deprecated
```

**구성 파일 사용:**

```python
import logging
import logging.config

def main():
    logging.config.fileConfig('logconfig.ini')
    ...
```

**logconfig.ini:**

```ini
[loggers]
keys=root

[handlers]
keys=defaultHandler

[formatters]
keys=defaultFormatter

[logger_root]
level=INFO
handlers=defaultHandler
qualname=root

[handler_defaultHandler]
class=FileHandler
formatter=defaultFormatter
args=('app.log', 'a')

[formatter_defaultFormatter]
format=%(levelname)s:%(name)s:%(message)s
```

### 논의

**표준 오류로 출력:**

```python
logging.basicConfig(level=logging.INFO)
```

basicConfig()는 프로그램에서 한 번만 호출할 수 있다. 나중에 로깅 구성을 변경하려면 루트 로거를 가져와서 직접 변경해야 한다:

```python
logging.getLogger().level = logging.DEBUG
```

## 13.12. 라이브러리에 로깅 추가

### 문제

라이브러리에 로깅 기능을 추가하고 싶지만, 로깅을 사용하지 않는 프로그램을 방해하고 싶지 않다.

### 해결방법

```python
# somelib.py
import logging

log = logging.getLogger(__name__)
log.addHandler(logging.NullHandler())

def func():
    log.critical('A Critical Error!')
    log.debug('A debug message')
```

이 구성으로는 기본적으로 로깅이 발생하지 않는다:

```python
import somelib
somelib.func()  # 출력 없음
```

하지만 로깅 시스템이 구성되면 로그 메시지가 나타나기 시작한다:

```python
import logging
logging.basicConfig()
somelib.func()
# CRITICAL:somelib:A Critical Error!
```

### 논의

**개별 라이브러리 로깅 구성:**

```python
import logging

logging.basicConfig(level=logging.ERROR)

import somelib
somelib.func()
# CRITICAL:somelib:A Critical Error!

# 'somelib'만의 로깅 레벨 변경
logging.getLogger('somelib').level = logging.DEBUG
somelib.func()
# CRITICAL:somelib:A Critical Error!
# DEBUG:somelib:A debug message
```

getLogger(__name__) 호출은 호출 모듈과 같은 이름을 가진 로거 모듈을 생성한다. 모든 모듈은 고유하므로 다른 로거와 분리될 가능성이 높은 전용 로거를 생성한다.

log.addHandler(logging.NullHandler()) 작업은 방금 생성된 로거 객체에 null 핸들러를 연결한다. null 핸들러는 기본적으로 모든 로깅 메시지를 무시한다.

## 13.13. 스톱워치 타이머 만들기

### 문제

다양한 작업을 수행하는 데 걸리는 시간을 기록할 수 있기를 원한다.

### 해결방법

```python
import time

class Timer:
    def __init__(self, func=time.perf_counter):
        self.elapsed = 0.0
        self._func = func
        self._start = None
    
    def start(self):
        if self._start is not None:
            raise RuntimeError('Already started')
        self._start = self._func()
    
    def stop(self):
        if self._start is None:
            raise RuntimeError('Not started')
        end = self._func()
        self.elapsed += end - self._start
        self._start = None
    
    def reset(self):
        self.elapsed = 0.0
    
    @property
    def running(self):
        return self._start is not None
    
    def __enter__(self):
        self.start()
        return self
    
    def __exit__(self, *args):
        self.stop()
```

**사용 예제:**

```python
def countdown(n):
    while n > 0:
        n -= 1

# 사용법 1: 명시적 start/stop
t = Timer()
t.start()
countdown(1000000)
t.stop()
print(t.elapsed)

# 사용법 2: 컨텍스트 매니저로
with t:
    countdown(1000000)
print(t.elapsed)

with Timer() as t2:
    countdown(1000000)
print(t2.elapsed)
```

### 논의

Timer 클래스는 벽시계 시간에 따라 시간을 기록하며 수면 시간을 포함한 모든 시간을 포함한다.

**CPU 시간만 측정:**

```python
t = Timer(time.process_time)
with t:
    countdown(1000000)
print(t.elapsed)
```

time.perf_counter()와 time.process_time() 모두 분수 초 단위로 "시간"을 반환한다. 실제 시간 값 자체는 특별한 의미가 없으며, 의미 있는 결과를 얻으려면 함수를 두 번 호출하고 시간
차이를 계산해야 한다.

## 13.14. 메모리 및 CPU 사용량 제한

### 문제

Unix 시스템에서 실행되는 프로그램의 메모리나 CPU 사용에 제한을 두고 싶다.

### 해결방법

**CPU 시간 제한:**

```python
import signal
import resource
import os

def time_exceeded(signo, frame):
    print("Time's up!")
    raise SystemExit(1)

def set_max_runtime(seconds):
    soft, hard = resource.getrlimit(resource.RLIMIT_CPU)
    resource.setrlimit(resource.RLIMIT_CPU, (seconds, hard))
    signal.signal(signal.SIGXCPU, time_exceeded)

if __name__ == '__main__':
    set_max_runtime(15)
    while True:
        pass
```

**메모리 사용 제한:**

```python
import resource

def limit_memory(maxsize):
    soft, hard = resource.getrlimit(resource.RLIMIT_AS)
    resource.setrlimit(resource.RLIMIT_AS, (maxsize, hard))
```

메모리 제한이 설정되면 더 이상 메모리를 사용할 수 없을 때 프로그램이 MemoryError 예외를 생성하기 시작한다.

### 논의

setrlimit() 함수는 특정 리소스에 대한 소프트 및 하드 제한을 설정하는 데 사용된다:

- **소프트 제한**: 운영 체제가 일반적으로 프로세스를 제한하거나 신호를 통해 알리는 값
- **하드 제한**: 소프트 제한에 사용될 수 있는 값의 상한선

하드 제한은 낮출 수 있지만, 사용자 프로세스가 올릴 수는 없다(프로세스가 스스로 낮췄더라도).

setrlimit() 함수는 자식 프로세스 수, 열린 파일 수 및 유사한 시스템 리소스에 대한 제한을 설정하는 데에도 사용할 수 있다.

이 레시피는 Unix 시스템에서만 작동하며 모든 Unix 시스템에서 작동하지 않을 수 있다.

## 13.15. 웹 브라우저 실행

### 문제

스크립트에서 브라우저를 실행하고 지정한 URL을 가리키도록 하고 싶다.

### 해결방법

```python
import webbrowser

# 기본 브라우저로 페이지 열기
webbrowser.open('http://www.python.org')  # True

# 새 브라우저 창에서 페이지 열기
webbrowser.open_new('http://www.python.org')  # True

# 새 브라우저 탭에서 페이지 열기
webbrowser.open_new_tab('http://www.python.org')  # True
```

**특정 브라우저 사용:**

```python
c = webbrowser.get('firefox')
c.open('http://www.python.org')  # True
c.open_new_tab('http://docs.python.org')  # True
```

지원되는 브라우저 이름의 전체 목록은 Python 문서에서 찾을 수 있다.

### 논의

브라우저를 쉽게 실행하는 것은 많은 스크립트에서 유용한 작업이 될 수 있다. 예를 들어:

- 스크립트가 서버에 배포를 수행하고 작동하는지 확인하기 위해 브라우저를 빠르게 실행하고 싶을 때
- 프로그램이 HTML 페이지 형태로 데이터를 출력하고 결과를 보기 위해 브라우저를 실행하고 싶을 때

어느 경우든 webbrowser 모듈이 간단한 솔루션을 제공한다.