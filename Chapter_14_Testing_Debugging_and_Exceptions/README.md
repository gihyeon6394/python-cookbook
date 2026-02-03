# CHAPTER 14 Testing, Debugging, and Exceptions

테스팅은 중요하지만 디버깅은 그다지 즐겁지 않다. Python은 코드 실행 전 컴파일러가 분석하지 않기 때문에 테스팅이 개발의 핵심 부분이다. 이 장에서는 테스팅, 디버깅, 예외 처리와 관련된 일반적인 문제들을
다룬다.

## 14.1. stdout으로 전송된 출력 테스팅

### 문제

표준 출력(sys.stdout)으로 출력을 보내는 메서드가 있는 프로그램이 있다. 적절한 입력이 주어졌을 때 올바른 출력이 표시되는지 증명하는 테스트를 작성하고 싶다.

### 해결

`unittest.mock` 모듈의 `patch()` 함수를 사용하면 단일 테스트에 대해서만 sys.stdout을 mock하고 다시 되돌릴 수 있다.

예제 함수:

```python
# mymodule.py
def urlprint(protocol, host, domain):
    url = '{}://{}.{}'.format(protocol, host, domain)
    print(url)
```

테스트 코드:

```python
from io import StringIO
from unittest import TestCase
from unittest.mock import patch
import mymodule

class TestURLPrint(TestCase):
    def test_url_gets_to_stdout(self):
        protocol = 'http'
        host = 'www'
        domain = 'example.com'
        expected_url = '{}://{}.{}\n'.format(protocol, host, domain)
        
        with patch('sys.stdout', new=StringIO()) as fake_out:
            mymodule.urlprint(protocol, host, domain)
            self.assertEqual(fake_out.getvalue(), expected_url)
```

### 논의

`unittest.mock.patch()` 함수는 context manager로 사용되어 sys.stdout의 값을 StringIO 객체로 대체한다. with 문이 완료되면 patch는 테스트 실행 전 상태로 모든
것을 편리하게 되돌린다.

특정 C 확장은 sys.stdout 설정을 우회하여 표준 출력에 직접 쓸 수 있다는 점에 주의해야 한다. 이 레시피는 그런 경우에는 도움이 되지 않지만 순수 Python 코드에서는 잘 작동한다.

## 14.2. Unit Test에서 객체 패칭

### 문제

unit test를 작성할 때 선택한 객체에 patch를 적용하여 테스트에서 어떻게 사용되었는지 검증해야 한다.

### 해결

`unittest.mock.patch()` 함수는 decorator, context manager, 또는 독립적으로 사용할 수 있다.

decorator로 사용:

```python
from unittest.mock import patch
import example

@patch('example.func')
def test1(x, mock_func):
    example.func(x)
    mock_func.assert_called_with(x)
```

context manager로 사용:

```python
with patch('example.func') as mock_func:
    example.func(x)
    mock_func.assert_called_with(x)
```

수동으로 사용:

```python
p = patch('example.func')
mock_func = p.start()
example.func(x)
mock_func.assert_called_with(x)
p.stop()
```

여러 객체를 패치하려면 decorator와 context manager를 stack할 수 있다:

```python
@patch('example.func1')
@patch('example.func2')
@patch('example.func3')
def test1(mock1, mock2, mock3):
    ...
```

### 논의

`patch()`는 제공한 정규화된 이름을 가진 기존 객체를 가져와 새 값으로 교체한다. 원래 값은 decorated 함수나 context manager가 완료된 후 복원된다.

기본적으로 값은 MagicMock 인스턴스로 교체된다. MagicMock은 호출 및 인스턴스를 모방하며 사용 정보를 기록하고 assertion을 만들 수 있게 한다.

```python
>>> from unittest.mock import MagicMock
>>> m = MagicMock(return_value = 10)
>>> m(1, 2, debug=True)
10
>>> m.assert_called_with(1, 2, debug=True)
>>> m.upper.return_value = 'HELLO'
>>> m.upper('hello')
'HELLO'
>>> assert m.upper.called
```

실제 예제 - 웹에서 데이터를 가져오는 함수 테스트:

```python
import unittest
from unittest.mock import patch
import io
import example

sample_data = io.BytesIO(b'''\
"IBM",91.1\r
"AA",13.25\r
"MSFT",27.72\r
\r
''')

class Tests(unittest.TestCase):
    @patch('example.urlopen', return_value=sample_data)
    def test_dowprices(self, mock_urlopen):
        p = example.dowprices()
        self.assertTrue(mock_urlopen.called)
        self.assertEqual(p, {'IBM': 91.1, 'AA': 13.25, 'MSFT': 27.72})
```

중요하지만 미묘한 점은 `urllib.request.urlopen` 대신 `example.urlopen`을 패치한다는 것이다. 패치를 만들 때는 테스트되는 코드에서 사용되는 이름을 사용해야 한다.

## 14.3. Unit Test에서 예외 조건 테스팅

### 문제

예외가 발생하는지 깔끔하게 테스트하는 unit test를 작성하고 싶다.

### 해결

예외를 테스트하려면 `assertRaises()` 메서드를 사용한다:

```python
import unittest

def parse_int(s):
    return int(s)

class TestConversion(unittest.TestCase):
    def test_bad_int(self):
        self.assertRaises(ValueError, parse_int, 'N/A')
```

예외의 값을 검사해야 한다면 다른 접근이 필요하다:

```python
import errno

class TestIO(unittest.TestCase):
    def test_file_not_found(self):
        try:
            f = open('/file/not/found')
        except IOError as e:
            self.assertEqual(e.errno, errno.ENOENT)
        else:
            self.fail('IOError not raised')
```

### 논의

`assertRaises()` 메서드는 예외의 존재를 테스트하는 편리한 방법을 제공한다. 일반적인 함정은 예외를 직접 처리하려고 하는 테스트를 작성하는 것이다.

`assertRaises()`의 한 가지 제한사항은 생성된 예외 객체의 값을 테스트하는 수단을 제공하지 않는다는 것이다. 그렇게 하려면 수동으로 테스트해야 한다.

`assertRaisesRegex()` 메서드를 사용하면 예외를 테스트하고 동시에 예외의 문자열 표현에 대해 정규 표현식 매치를 수행할 수 있다:

```python
class TestConversion(unittest.TestCase):
    def test_bad_int(self):
        self.assertRaisesRegex(ValueError, 'invalid literal .*', 
                               parse_int, 'N/A')
```

`assertRaises()`와 `assertRaisesRegex()`는 context manager로도 사용할 수 있다:

```python
class TestConversion(unittest.TestCase):
    def test_bad_int(self):
        with self.assertRaisesRegex(ValueError, 'invalid literal .*'):
            r = parse_int('N/A')
```

## 14.4. 테스트 출력을 파일에 로깅

### 문제

unit test 실행 결과를 표준 출력 대신 파일에 기록하고 싶다.

### 해결

테스팅 파일 하단에 다음과 같은 코드 조각을 포함하는 것이 일반적이다:

```python
import unittest

class MyTest(unittest.TestCase):
    ...

if __name__ == '__main__':
    unittest.main()
```

출력을 리디렉션하려면 main() 호출을 풀고 자체 main() 함수를 작성해야 한다:

```python
import sys

def main(out=sys.stderr, verbosity=2):
    loader = unittest.TestLoader()
    suite = loader.loadTestsFromModule(sys.modules[__name__])
    unittest.TextTestRunner(out, verbosity=verbosity).run(suite)

if __name__ == '__main__':
    with open('testing.out', 'w') as f:
        main(f)
```

### 논의

unittest 모듈은 기본적으로 먼저 test suite를 조립하여 작동한다. suite가 조립되면 포함된 테스트가 실행된다.

`unittest.TestLoader` 인스턴스는 test suite를 조립하는 데 사용된다. `loadTestsFromModule()`은 테스트를 수집하는 여러 메서드 중 하나다.

`TextTestRunner` 클래스는 test runner 클래스의 예다. 이 클래스의 주요 목적은 test suite에 포함된 테스트를 실행하는 것이다.

## 14.5. 테스트 실패 건너뛰기 또는 예상

### 문제

unit test에서 선택한 테스트를 건너뛰거나 예상된 실패로 표시하고 싶다.

### 해결

unittest 모듈에는 선택한 테스트 메서드에 적용하여 처리를 제어할 수 있는 decorator가 있다:

```python
import unittest
import os
import platform

class Tests(unittest.TestCase):
    def test_0(self):
        self.assertTrue(True)
    
    @unittest.skip('skipped test')
    def test_1(self):
        self.fail('should have failed!')
    
    @unittest.skipIf(os.name=='posix', 'Not supported on Unix')
    def test_2(self):
        import winreg
    
    @unittest.skipUnless(platform.system() == 'Darwin', 'Mac specific test')
    def test_3(self):
        self.assertTrue(True)
    
    @unittest.expectedFailure
    def test_4(self):
        self.assertEqual(2+2, 5)
```

### 논의

`skip()` decorator는 실행하고 싶지 않은 테스트를 건너뛰는 데 사용할 수 있다. `skipIf()`와 `skipUnless()`는 특정 플랫폼이나 Python 버전에만 적용되는 테스트를 작성하는 유용한
방법이다. `@expectedFailure` decorator는 알려진 실패이지만 테스트 프레임워크가 더 많은 정보를 보고하지 않기를 원하는 테스트를 표시하는 데 사용한다.

메서드를 건너뛰는 decorator는 전체 테스팅 클래스에도 적용할 수 있다:

```python
@unittest.skipUnless(platform.system() == 'Darwin', 'Mac specific tests')
class DarwinTests(unittest.TestCase):
    ...
```

## 14.6. 여러 예외 처리

### 문제

여러 다른 예외를 던질 수 있는 코드가 있고, 중복 코드나 길고 복잡한 코드 없이 발생할 수 있는 모든 잠재적 예외를 고려해야 한다.

### 해결

단일 코드 블록으로 다른 예외를 모두 처리할 수 있다면 tuple로 그룹화할 수 있다:

```python
try:
    client_obj.get_url(url)
except (URLError, ValueError, SocketTimeout):
    client_obj.remove_url(url)
```

예외 중 하나를 다르게 처리해야 한다면 자체 except 절에 넣는다:

```python
try:
    client_obj.get_url(url)
except (URLError, ValueError):
    client_obj.remove_url(url)
except SocketTimeout:
    client_obj.handle_url_timeout(url)
```

많은 예외는 상속 계층으로 그룹화된다. 이러한 예외의 경우 기본 클래스를 지정하여 모두 catch할 수 있다:

```python
try:
    f = open(filename)
except OSError:
    ...
```

이는 OSError가 FileNotFoundError와 PermissionError 예외 모두에 공통인 기본 클래스이기 때문에 작동한다.

### 논의

`as` 키워드를 사용하여 발생한 예외에 대한 핸들을 얻을 수 있다:

```python
try:
    f = open(filename)
except OSError as e:
    if e.errno == errno.ENOENT:
        logger.error('File not found')
    elif e.errno == errno.EACCES:
        logger.error('Permission denied')
    else:
        logger.error('Unexpected error: %d', e.errno)
```

except 절은 나열된 순서대로 확인되며 첫 번째 일치가 실행된다는 점에 유의해야 한다.

디버깅 팁으로, 특정 예외의 클래스 계층에 대해 확실하지 않다면 예외의 `__mro__` 속성을 검사하여 빠르게 볼 수 있다:

```python
>>> FileNotFoundError.__mro__
(<class 'FileNotFoundError'>, <class 'OSError'>, <class 'Exception'>,
 <class 'BaseException'>, <class 'object'>)
```

## 14.7. 모든 예외 잡기

### 문제

모든 예외를 잡는 코드를 작성하고 싶다.

### 해결

모든 예외를 잡으려면 Exception에 대한 예외 핸들러를 작성한다:

```python
try:
    ...
except Exception as e:
    ...
    log('Reason:', e)  # 중요!
```

이렇게 하면 SystemExit, KeyboardInterrupt, GeneratorExit를 제외한 모든 예외를 잡는다. 이러한 예외도 잡으려면 Exception을 BaseException으로 변경한다.

### 논의

모든 예외를 잡는 것은 복잡한 작업에서 발생할 수 있는 모든 가능한 예외를 기억하지 못하는 프로그래머가 때때로 사용하는 방법이다. 주의하지 않으면 디버그할 수 없는 코드를 작성하는 좋은 방법이기도 하다.

모든 예외를 잡기로 선택했다면 어딘가에 예외의 실제 이유를 로그하거나 보고하는 것이 절대적으로 중요하다. 그렇지 않으면 어느 시점에서 머리가 폭발할 것이다.

예제:

```python
def parse_int(s):
    try:
        n = int(v)
    except Exception as e:
        print("Couldn't parse")
        print('Reason:', e)
```

모든 것이 동등하다면 예외 처리에서 가능한 한 정확하게 하는 것이 더 낫다. 그러나 모든 예외를 잡아야 한다면 좋은 진단 정보를 제공하거나 원인이 손실되지 않도록 예외를 전파해야 한다.

## 14.8. 사용자 정의 예외 생성

### 문제

애플리케이션을 구축 중이며 lower-level 예외를 애플리케이션 컨텍스트에서 더 의미 있는 사용자 정의 예외로 래핑하고 싶다.

### 해결

새 예외 생성은 쉽다 - Exception(또는 더 적절한 경우 다른 기존 예외 타입)에서 상속하는 클래스로 정의하면 된다:

```python
class NetworkError(Exception):
    pass

class HostnameError(NetworkError):
    pass

class TimeoutError(NetworkError):
    pass

class ProtocolError(NetworkError):
    pass
```

사용자는 일반적인 방식으로 이러한 예외를 사용할 수 있다:

```python
try:
    msg = s.recv()
except TimeoutError as e:
    ...
except ProtocolError as e:
    ...
```

### 논의

사용자 정의 예외 클래스는 거의 항상 내장 Exception 클래스에서 상속하거나 Exception에서 상속하는 로컬에 정의된 기본 예외에서 상속해야 한다.

모든 예외는 BaseException에서도 파생되지만 새 예외의 기본 클래스로 사용해서는 안 된다. BaseException은 KeyboardInterrupt나 SystemExit와 같은 시스템 종료 예외를 위해
예약되어 있다.

상속을 통한 사용자 정의 예외의 그룹화에 대한 디자인 고려사항이 있다. 복잡한 애플리케이션에서는 예외의 다른 클래스를 함께 그룹화하는 추가 기본 클래스를 도입하는 것이 합리적일 수 있다:

```python
try:
    s.send(msg)
except ProtocolError:
    ...
```

또는 광범위한 오류를 잡을 수 있는 기능도 제공한다:

```python
try:
    s.send(msg)
except NetworkError:
    ...
```

Exception의 `__init__()` 메서드를 재정의하는 새 예외를 정의하려면 항상 전달된 모든 인수로 `Exception.__init__()`를 호출해야 한다:

```python
class CustomError(Exception):
    def __init__(self, message, status):
        super().__init__(message, status)
        self.message = message
        self.status = status
```

Exception의 기본 동작은 전달된 모든 인수를 수락하고 tuple로 .args 속성에 저장하는 것이다.

## 14.9. 다른 예외에 대한 응답으로 예외 발생

### 문제

다른 예외를 잡은 것에 대한 응답으로 예외를 발생시키고 싶지만 traceback에 두 예외에 대한 정보를 모두 포함하고 싶다.

### 해결

예외를 연쇄하려면 단순 raise 문 대신 `raise from` 문을 사용한다:

```python
>>> def example():
...     try:
...         int('N/A')
...     except ValueError as e:
...         raise RuntimeError('A parsing error occurred') from e
...
>>> example()
Traceback (most recent call last):
  File "<stdin>", line 3, in example
ValueError: invalid literal for int() with base 10: 'N/A'

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 5, in example
RuntimeError: A parsing error occurred
```

traceback에서 볼 수 있듯이 두 예외가 모두 캡처된다. 이러한 예외를 잡으려면 일반 except 문을 사용한다. 예외 체인을 따르려면 예외 객체의 `__cause__` 속성을 볼 수 있다:

```python
try:
    example()
except RuntimeError as e:
    print("It didn't work:", e)
    if e.__cause__:
        print('Cause:', e.__cause__)
```

### 논의

암묵적 형태의 연쇄 예외는 except 블록 내부에서 다른 예외가 발생할 때 발생한다. 이 경우 예외의 `__cause__` 속성이 설정되지 않는다. 대신 `__context__` 속성이 이전 예외로 설정된다.

어떤 이유로 연쇄를 억제하려면 `raise from None`을 사용한다:

```python
>>> def example3():
...     try:
...         int('N/A')
...     except ValueError:
...         raise RuntimeError('A parsing error occurred') from None
...
>>> example3()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 5, in example3
RuntimeError: A parsing error occurred
```

코드를 설계할 때 다른 except 블록 내부의 raise 문 사용에 주의를 기울여야 한다. 대부분의 경우 이러한 raise 문은 `raise from` 문으로 변경되어야 한다.

## 14.10. 마지막 예외 다시 발생시키기

### 문제

except 블록에서 예외를 잡았지만 이제 다시 발생시키고 싶다.

### 해결

단순히 raise 문을 단독으로 사용한다:

```python
>>> def example():
...     try:
...         int('N/A')
...     except ValueError:
...         print("Didn't work")
...         raise
...
>>> example()
Didn't work
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 3, in example
ValueError: invalid literal for int() with base 10: 'N/A'
```

### 논의

이 문제는 일반적으로 예외에 대한 응답으로 어떤 조치를 취해야 할 때(예: 로깅, 정리 등) 발생하지만 그 후에는 단순히 예외를 전파하고 싶을 때 발생한다.

일반적인 사용은 catch-all 예외 핸들러에서 발생할 수 있다:

```python
try:
    ...
except Exception as e:
    # 어떤 방식으로든 예외 정보 처리
    ...
    # 예외 전파
    raise
```

## 14.11. 경고 메시지 발행

### 문제

프로그램이 경고 메시지(예: deprecated 기능이나 사용 문제에 대해)를 발행하도록 하고 싶다.

### 해결

프로그램이 경고 메시지를 발행하도록 하려면 `warnings.warn()` 함수를 사용한다:

```python
import warnings

def func(x, y, logfile=None, debug=False):
    if logfile is not None:
        warnings.warn('logfile argument deprecated', DeprecationWarning)
    ...
```

`warn()`의 인수는 경고 메시지와 경고 클래스다. 일반적으로 UserWarning, DeprecationWarning, SyntaxWarning, RuntimeWarning, ResourceWarning,
FutureWarning 중 하나다.

경고 처리는 interpreter 실행 방법과 기타 구성에 따라 달라진다. `-W all` 옵션으로 Python을 실행하면 다음과 같은 출력을 얻는다:

```bash
bash % python3 -W all example.py
example.py:5: DeprecationWarning: logfile argument is deprecated
  warnings.warn('logfile argument is deprecated', DeprecationWarning)
```

경고를 예외로 전환하려면 `-W error` 옵션을 사용한다.

### 논의

경고 메시지를 발행하는 것은 소프트웨어를 유지 관리하고 전체 예외 수준까지 올라가지 않는 문제에 대해 사용자를 지원하는 유용한 기술이다.

기본적으로 모든 경고 메시지가 나타나는 것은 아니다. Python에 대한 `-W` 옵션은 경고 메시지의 출력을 제어할 수 있다. `-W all`은 모든 경고 메시지를 출력하고, `-W ignore`는 모든 경고를
무시하며, `-W error`는 경고를 예외로 전환한다.

대안으로 `warnings.simplefilter()` 함수를 사용하여 출력을 제어할 수 있다.

## 14.12. 기본 프로그램 충돌 디버깅

### 문제

프로그램이 깨졌고 디버깅을 위한 간단한 전략을 원한다.

### 해결

프로그램이 예외로 충돌하는 경우 `python3 -i someprogram.py`로 프로그램을 실행하면 단순히 둘러보는 데 유용한 도구가 될 수 있다. `-i` 옵션은 프로그램이 종료되는 즉시 대화형 shell을
시작한다.

예제:

```python
# sample.py
def func(n):
    return n + 10

func('Hello')
```

실행:

```bash
bash % python3 -i sample.py
Traceback (most recent call last):
  File "sample.py", line 6, in <module>
    func('Hello')
  File "sample.py", line 4, in func
    return n + 10
TypeError: Can't convert 'int' object to str implicitly
>>> func(10)
20
```

명백한 것이 보이지 않으면 추가 단계는 충돌 후 Python debugger를 시작하는 것이다:

```python
>>> import pdb
>>> pdb.pm()
> sample.py(4)func()
-> return n + 10
(Pdb) w
 sample.py(6)<module>()
-> func('Hello')
> sample.py(4)func()
-> return n + 10
(Pdb) print n
'Hello'
(Pdb) q
```

대화형 shell을 얻기 어려운 환경에 코드가 깊이 묻혀 있는 경우(예: 서버) 종종 오류를 잡고 traceback을 직접 생성할 수 있다:

```python
import traceback
import sys

try:
    func(arg)
except:
    print('**** AN ERROR OCCURRED ****')
    traceback.print_exc(file=sys.stderr)
```

프로그램이 충돌하지 않지만 잘못된 답을 생성하거나 작동 방식이 미스터리한 경우, 관심 있는 곳에 몇 개의 `print()` 호출을 삽입하는 것이 잘못된 것이 아니다. `traceback.print_stack()`
함수는 해당 시점에서 즉시 프로그램의 stack trace를 생성한다:

```python
>>> def sample(n):
...     if n > 0:
...         sample(n-1)
...     else:
...         traceback.print_stack(file=sys.stderr)
...
>>> sample(5)
```

또는 `pdb.set_trace()`를 사용하여 프로그램의 어느 지점에서든 debugger를 수동으로 시작할 수 있다:

```python
import pdb

def func(arg):
    ...
    pdb.set_trace()
    ...
```

### 논의

디버깅을 필요 이상으로 복잡하게 만들지 마라. 간단한 오류는 프로그램 traceback을 읽는 방법을 아는 것만으로 해결할 수 있다. 코드에 몇 개의 선택된 `print()` 함수를 삽입하는 것도 잘 작동할 수
있다.

debugger의 일반적인 사용은 충돌한 함수 내부의 변수를 검사하는 것이다. 이러한 충돌이 발생한 후 debugger에 들어가는 방법을 아는 것은 유용한 기술이다.

## 14.13. 프로그램 프로파일링 및 타이밍

### 문제

프로그램이 시간을 어디에 소비하는지 알아내고 타이밍 측정을 하고 싶다.

### 해결

전체 프로그램의 시간을 측정하려면 Unix time 명령을 사용하는 것이 충분히 쉽다:

```bash
bash % time python3 someprogram.py
real    0m13.937s
user    0m12.162s
sys     0m0.098s
```

프로그램이 무엇을 하고 있는지 보여주는 상세한 보고서를 원한다면 cProfile 모듈을 사용할 수 있다:

```bash
bash % python3 -m cProfile someprogram.py
         859647 function calls in 16.016 CPU seconds

   Ordered by: standard name

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
   263169    0.080    0.000    0.080    0.000 someprogram.py:16(frange)
      513    0.001    0.000    0.002    0.000 someprogram.py:30(generate_mandel)
   ...
```

선택된 함수의 프로파일링을 위해 짧은 decorator가 유용할 수 있다:

```python
# timethis.py
import time
from functools import wraps

def timethis(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        r = func(*args, **kwargs)
        end = time.perf_counter()
        print('{}.{} : {}'.format(func.__module__, func.__name__, end - start))
        return r
    return wrapper
```

사용 예:

```python
>>> @timethis
... def countdown(n):
...     while n > 0:
...         n -= 1
...
>>> countdown(10000000)
__main__.countdown : 0.803001880645752
```

문 블록의 시간을 측정하려면 context manager를 정의할 수 있다:

```python
from contextlib import contextmanager

@contextmanager
def timeblock(label):
    start = time.perf_counter()
    try:
        yield
    finally:
        end = time.perf_counter()
        print('{} : {}'.format(label, end - start))
```

사용 예:

```python
>>> with timeblock('counting'):
...     n = 10000000
...     while n > 0:
...         n -= 1
...
counting : 1.5551159381866455
```

작은 코드 조각의 성능을 연구하려면 timeit 모듈이 유용할 수 있다:

```python
>>> from timeit import timeit
>>> timeit('math.sqrt(2)', 'import math')
0.1432319980012835
>>> timeit('sqrt(2)', 'from math import sqrt')
0.10836604500218527
```

### 논의

성능 측정을 할 때 얻는 모든 결과는 근사치라는 점을 알아야 한다. `time.perf_counter()` 함수는 주어진 플랫폼에서 가능한 가장 높은 해상도의 타이머를 제공한다. 그러나 여전히 wall-clock
시간을 측정하며 기계 부하와 같은 많은 다른 요인에 의해 영향을 받을 수 있다.

wall-clock 시간 대신 프로세스 시간에 관심이 있다면 대신 `time.process_time()`을 사용한다.

상세한 타이밍 분석을 수행하려면 time, timeit 및 기타 관련 모듈에 대한 문서를 읽어 중요한 플랫폼 관련 차이점과 기타 함정을 이해해야 한다.

## 14.14. 프로그램을 더 빠르게 만들기

### 문제

프로그램이 너무 느리게 실행되고 C 확장이나 JIT(Just-In-Time) 컴파일러와 같은 더 극단적인 솔루션의 도움 없이 속도를 높이고 싶다.

### 해결

최적화의 첫 번째 규칙은 "하지 마라"일 수 있지만, 두 번째 규칙은 거의 확실히 "중요하지 않은 것을 최적화하지 마라"다. 프로그램이 느리게 실행된다면 코드를 프로파일링하는 것으로 시작할 수 있다.

프로그램은 내부 데이터 처리 루프와 같은 몇 가지 hotspot에서 시간을 소비한다는 것을 알게 될 것이다. 이러한 위치를 식별하면 다음 섹션에 제시된 실질적인 기술을 사용하여 프로그램을 더 빠르게 실행할 수 있다.

### 함수 사용

많은 프로그래머가 간단한 스크립트를 작성하는 언어로 Python을 사용하기 시작한다. 스크립트를 작성할 때 구조가 거의 없이 코드를 작성하는 관행에 빠지기 쉽다:

```python
# somescript.py
import sys
import csv

with open(sys.argv[1]) as f:
    for row in csv.reader(f):
        # 어떤 종류의 처리
        ...
```

이와 같이 전역 범위에 정의된 코드는 함수에 정의된 코드보다 느리게 실행된다는 것은 잘 알려지지 않은 사실이다. 속도 차이는 local과 global 변수의 구현과 관련이 있다(local과 관련된 작업이 더
빠르다).

프로그램을 더 빠르게 실행하려면 스크립팅 문을 함수에 넣기만 하면 된다:

```python
# somescript.py
import sys
import csv

def main(filename):
    with open(filename) as f:
        for row in csv.reader(f):
            # 어떤 종류의 처리
            ...

main(sys.argv[1])
```

속도 차이는 수행되는 처리에 크게 의존하지만 경험상 15-30%의 속도 향상이 드문 일이 아니다.

### 속성 접근 선택적으로 제거

점(.) 연산자를 사용하여 속성에 접근할 때마다 비용이 발생한다. 내부적으로 이것은 `__getattribute__()`와 `__getattr__()`와 같은 특수 메서드를 트리거하며, 종종 dictionary
lookup으로 이어진다.

`from module import name` 형식의 import를 사용하고 bound method를 선택적으로 사용하여 속성 lookup을 피할 수 있다:

```python
# 느림
import math

def compute_roots(nums):
    result = []
    for n in nums:
        result.append(math.sqrt(n))
    return result

# 빠름
from math import sqrt

def compute_roots(nums):
    result = []
    result_append = result.append
    for n in nums:
        result_append(sqrt(n))
    return result
```

그러나 이러한 변경은 루프와 같이 자주 실행되는 코드에서만 의미가 있다는 점을 강조해야 한다.

### 변수의 지역성 이해

앞서 언급했듯이 local 변수는 global 변수보다 빠르다. 자주 접근하는 이름의 경우 이러한 이름을 가능한 한 local하게 만들어 속도 향상을 얻을 수 있다:

```python
import math

def compute_roots(nums):
    sqrt = math.sqrt  # local 변수로 만들기
    result = []
    result_append = result.append
    for n in nums:
        result_append(sqrt(n))
    return result
```

지역성 인수는 클래스에서 작업할 때도 적용된다. 일반적으로 `self.name`과 같은 값을 조회하는 것은 local 변수에 접근하는 것보다 상당히 느리다. 내부 루프에서는 자주 접근하는 속성을 local 변수로
끌어올리는 것이 도움이 될 수 있다:

```python
# 느림
class SomeClass:
    ...
    def method(self):
        for x in s:
            op(self.value)

# 빠름
class SomeClass:
    ...
    def method(self):
        value = self.value
        for x in s:
            op(value)
```

### 불필요한 추상화 피하기

decorator, property, descriptor와 같은 추가 처리 계층으로 코드를 래핑할 때마다 느려진다:

```python
class A:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    @property
    def y(self):
        return self._y
    
    @y.setter
    def y(self, value):
        self._y = value
```

타이밍 테스트:

```python
>>> from timeit import timeit
>>> a = A(1,2)
>>> timeit('a.x', 'from __main__ import a')
0.07817923510447145
>>> timeit('a.y', 'from __main__ import a')
0.35766440676525235
```

property y에 접근하는 것이 단순 속성 x보다 약 4.5배 느리다. 이 차이가 중요하다면 y를 property로 정의하는 것이 정말 필요한지 자문해야 한다.

### 내장 컨테이너 사용

string, tuple, list, set, dict와 같은 내장 데이터 타입은 모두 C로 구현되어 있으며 상당히 빠르다. 자체 데이터 구조를 대체로 만들려는 경향이 있다면(예: linked list,
balanced tree 등) 내장의 속도를 따라잡기가 어렵거나 불가능할 수 있다.

### 불필요한 데이터 구조나 복사본 만들기 피하기

프로그래머는 때때로 필요하지 않을 때 불필요한 데이터 구조를 만드는 데 지나치게 열중한다:

```python
# 불필요한 첫 번째 list
values = [x for x in sequence]
squares = [x*x for x in values]

# 더 나은 방법
squares = [x*x for x in sequence]
```

`copy.deepcopy()`와 같은 함수의 과도한 사용은 Python의 메모리 모델을 완전히 이해하거나 신뢰하지 못하는 사람이 작성한 코드의 신호일 수 있다.

### 논의

최적화하기 전에 먼저 사용 중인 알고리즘을 연구하는 것이 일반적으로 가치가 있다. O(n**2) 알고리즘의 구현을 조정하려고 하는 것보다 O(n log n) 알고리즘으로 전환하면 훨씬 더 큰 속도 향상을 얻을 수
있다.

최적화해야 한다고 결정했다면 큰 그림을 고려하는 것이 도움이 된다. 일반적으로 프로그램의 모든 부분에 최적화를 적용하고 싶지 않다. 이러한 변경은 코드를 읽고 이해하기 어렵게 만들기 때문이다. 대신 내부 루프와 같은
알려진 성능 병목 현상에만 집중한다.

micro-optimization의 결과를 해석할 때 특히 주의해야 한다. 예를 들어 dictionary를 생성하는 두 가지 기술을 고려한다:

```python
a = {
    'name' : 'AAPL',
    'shares' : 100,
    'price' : 534.22
}

b = dict(name='AAPL', shares=100, price=534.22)
```

후자는 타이핑이 덜 필요하다는 이점이 있다. 그러나 성능 전투에서 두 코드 조각을 비교하면 `dict()` 사용이 3배 느리게 실행된다는 것을 알게 될 것이다. 그러나 현명한 프로그래머는 내부 루프와 같이 실제로
중요할 수 있는 프로그램 부분에만 집중할 것이다.

성능 요구사항이 이 레시피의 간단한 기술을 훨씬 넘어선다면 JIT(Just-In-Time) 컴파일 기술을 기반으로 하는 도구의 사용을 조사할 수 있다. 예를 들어 PyPy 프로젝트는 프로그램의 실행을 분석하고 자주
실행되는 부분에 대해 네이티브 기계 코드를 생성하는 Python interpreter의 대체 구현이다. 때로는 Python 프로그램을 1자릿수 더 빠르게 실행할 수 있으며, 종종 C로 작성된 코드의 속도에 접근하거나
심지어 초과한다.

Numba 프로젝트도 고려할 수 있다. Numba는 최적화하려는 선택된 Python 함수에 decorator로 주석을 다는 동적 컴파일러다. 이러한 함수는 LLVM을 사용하여 네이티브 기계 코드로 컴파일된다.

마지막으로 John Ousterhout의 말이 떠오른다: "최고의 성능 향상은 작동하지 않는 상태에서 작동하는 상태로의 전환이다." 필요할 때까지 최적화에 대해 걱정하지 마라. 프로그램이 올바르게 작동하는지 확인하는
것이 일반적으로 빠르게 실행되도록 하는 것보다 더 중요하다(적어도 처음에는).