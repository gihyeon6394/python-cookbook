# CHAPTER 10 Modules and Packages

모듈과 패키지는 모든 대규모 프로젝트와 Python 설치 자체의 핵심이다. 이 챕터는 패키지 구성 방법, 대형 모듈을 여러 파일로 분할하기, namespace 패키지 생성과 같은 모듈과 패키지와 관련된 일반적인
프로그래밍 기법에 초점을 맞춘다. import 문의 동작을 커스터마이즈할 수 있는 레시피도 제공한다.

## 10.1. 계층적 모듈 패키지 만들기

모듈의 계층적 컬렉션으로 구성된 패키지로 코드를 구성하고 싶다.

패키지 구조를 만드는 것은 간단하다. 파일시스템에서 원하는 대로 코드를 구성하고 모든 디렉토리가 `__init__.py` 파일을 정의하도록 한다. 예를 들어:

```
graphics/
    __init__.py
    primitive/
        __init__.py
        line.py
        fill.py
        text.py
    formats/
        __init__.py
        png.py
        jpg.py
```

이렇게 하면 다음과 같은 다양한 import 문을 수행할 수 있다:

```python
import graphics.primitive.line
from graphics.primitive import line
import graphics.formats.jpg as jpg
```

### Discussion

모듈 계층 구조를 정의하는 것은 파일시스템에 디렉토리 구조를 만드는 것만큼 쉽다. `__init__.py` 파일의 목적은 패키지의 다른 레벨이 발견될 때 실행되는 선택적 초기화 코드를 포함하는 것이다.

예를 들어, `import graphics` 문이 있으면 `graphics/__init__.py` 파일이 import되고 graphics 네임스페이스의 내용을 형성한다.
`import graphics.formats.jpg`와 같은 import의 경우, `graphics/__init__.py`와 `graphics/formats/__init__.py` 파일이 모두
`graphics/formats/jpg.py` 파일의 최종 import 전에 import된다.

대부분의 경우 `__init__.py` 파일을 비워두는 것이 좋다. 그러나 특정 상황에서는 코드를 포함할 수 있다. 예를 들어, `__init__.py` 파일은 다음과 같이 서브모듈을 자동으로 로드하는 데 사용될 수
있다:

```python
# graphics/formats/__init__.py
from . import jpg
from . import png
```

이러한 파일의 경우, 사용자는 `graphics.formats.jpg`와 `graphics.formats.png`에 대한 별도의 import 대신 단일 `import graphics.formats`를 사용하기만 하면
된다.

Python 3.3은 `__init__.py` 파일이 없어도 패키지 import를 수행하는 것처럼 보인다. `__init__.py`를 정의하지 않으면 실제로 "namespace package"라고 알려진 것을
생성하게 된다(Recipe 10.5 참조). 새 패키지 생성을 시작하는 경우라면 `__init__.py` 파일을 포함하라.

## 10.2. Import의 모든 것 제어

사용자가 `from module import *` 문을 사용할 때 모듈이나 패키지에서 export되는 심볼에 대한 정확한 제어를 원한다.

모듈에서 export되는 이름을 명시적으로 나열하는 변수 `__all__`을 정의한다:

```python
# somemodule.py
def spam():
    pass

def grok():
    pass

blah = 42

# 'spam'과 'grok'만 export
__all__ = ['spam', 'grok']
```

### Discussion

`from module import *`의 사용은 강력히 권장되지 않지만, 많은 이름을 정의하는 모듈에서는 여전히 자주 사용된다. 아무것도 하지 않으면 이 형태의 import는 밑줄로 시작하지 않는 모든 이름을
export한다. 반면에 `__all__`이 정의되면 명시적으로 나열된 이름만 export된다.

`__all__`을 빈 리스트로 정의하면 아무것도 export되지 않는다. `__all__`에 정의되지 않은 이름이 포함되어 있으면 import 시 AttributeError가 발생한다.

## 10.3. 상대 이름을 사용하여 패키지 서브모듈 Import

패키지로 구성된 코드가 있고 import 문에 패키지 이름을 하드코딩하지 않고 다른 패키지 서브모듈에서 서브모듈을 import하고 싶다.

동일한 패키지의 다른 모듈에서 패키지의 모듈을 import하려면 패키지 상대 import를 사용한다. 예를 들어, 파일시스템에 다음과 같이 구성된 mypackage 패키지가 있다고 가정하자:

```
mypackage/
    __init__.py
    A/
        __init__.py
        spam.py
        grok.py
    B/
        __init__.py
        bar.py
```

모듈 `mypackage.A.spam`이 동일한 디렉토리에 있는 모듈 grok을 import하려면 다음과 같은 import 문을 포함해야 한다:

```python
# mypackage/A/spam.py
from . import grok
```

동일한 모듈이 다른 디렉토리에 있는 모듈 B.bar를 import하려면 다음과 같은 import 문을 사용할 수 있다:

```python
# mypackage/A/spam.py
from ..B import bar
```

두 import 문 모두 spam.py 파일의 위치를 기준으로 작동하며 최상위 패키지 이름을 포함하지 않는다.

### Discussion

패키지 내에서 동일한 패키지의 모듈과 관련된 import는 완전히 지정된 절대 이름이나 표시된 구문을 사용한 상대 import를 사용할 수 있다:

```python
# mypackage/A/spam.py
from mypackage.A import grok  # OK
from . import grok             # OK
import grok                    # Error (not found)
```

`mypackage.A`와 같은 절대 이름을 사용하는 단점은 소스 코드에 최상위 패키지 이름을 하드코딩한다는 것이다. 이는 코드를 재구성하려는 경우 코드를 더 취약하고 작업하기 어렵게 만든다.

import 문의 `.` 및 `..` 구문은 이상해 보일 수 있지만, 디렉토리 이름을 지정하는 것으로 생각하라. `.`는 현재 디렉토리를 의미하고 `..B`는 `../B` 디렉토리를 의미한다. 이 구문은 from
형태의 import에서만 작동한다:

```python
from . import grok  # OK
import .grok        # ERROR
```

상대 import는 적절한 패키지 내에 있는 모듈에 대해서만 작동한다는 점에 유의해야 한다. 특히 스크립트의 최상위 레벨에 있는 간단한 모듈 내에서는 작동하지 않는다.

## 10.4. 모듈을 여러 파일로 분할

모듈을 여러 파일로 분할하고 싶다. 그러나 별도의 파일을 단일 논리 모듈로 통합하여 기존 코드를 손상시키지 않고 이를 수행하고 싶다.

프로그램 모듈은 패키지로 전환하여 별도의 파일로 분할할 수 있다. 다음의 간단한 모듈을 고려하자:

```python
# mymodule.py
class A:
    def spam(self):
        print('A.spam')

class B(A):
    def bar(self):
        print('B.bar')
```

mymodule.py를 각 클래스 정의에 대해 하나씩 두 개의 파일로 분할하고 싶다고 가정하자. 그렇게 하려면 mymodule.py 파일을 mymodule이라는 디렉토리로 교체한다. 해당 디렉토리에서 다음 파일을
생성한다:

```
mymodule/
    __init__.py
    a.py
    b.py
```

a.py 파일에 다음 코드를 넣는다:

```python
# a.py
class A:
    def spam(self):
        print('A.spam')
```

b.py 파일에 다음 코드를 넣는다:

```python
# b.py
from .a import A

class B(A):
    def bar(self):
        print('B.bar')
```

마지막으로 `__init__.py` 파일에서 두 파일을 함께 연결한다:

```python
# __init__.py
from .a import A
from .b import B
```

이러한 단계를 따르면 결과 mymodule 패키지는 단일 논리 모듈처럼 보인다:

```python
>>> import mymodule
>>> a = mymodule.A()
>>> a.spam()
A.spam
>>> b = mymodule.B()
>>> b.bar()
B.bar
```

### Discussion

이 레시피의 주요 관심사는 사용자가 많은 작은 모듈로 작업하기를 원하는지 아니면 단일 모듈로 작업하기를 원하는지에 대한 설계 질문이다. 후자의 경우 mymodule을 하나의 큰 소스 파일로 생각하는 것이 가장
일반적이다. 그러나 이 레시피는 여러 파일을 단일 논리 네임스페이스로 결합하는 방법을 보여준다. 핵심은 패키지 디렉토리를 생성하고 `__init__.py` 파일을 사용하여 부분을 결합하는 것이다.

모듈이 분할되면 파일 간 참조에 주의를 기울여야 한다. 예를 들어, 이 레시피에서 클래스 B는 기본 클래스로 클래스 A에 접근해야 한다. `from .a import A`의 패키지 상대 import가 이를 가져오는
데 사용된다.

이 레시피의 한 가지 확장은 "lazy" import의 도입을 포함한다. 표시된 대로 `__init__.py` 파일은 필요한 모든 서브컴포넌트를 한 번에 import한다. 그러나 매우 큰 모듈의 경우 필요할 때만
컴포넌트를 로드하고 싶을 수 있다. 이를 위해 `__init__.py`의 약간 다른 버전이 있다:

```python
# __init__.py
def A():
    from .a import A
    return A()

def B():
    from .b import B
    return B()
```

이 버전에서 클래스 A와 B는 처음 접근할 때 원하는 클래스를 로드하는 함수로 대체되었다.

lazy loading의 주요 단점은 상속 및 타입 검사가 중단될 수 있다는 것이다. 예를 들어, 코드를 약간 변경해야 할 수 있다:

```python
if isinstance(x, mymodule.A):  # Error
    ...
if isinstance(x, mymodule.a.A):  # Ok
    ...
```

## 10.5. 공통 네임스페이스 아래 별도의 코드 디렉토리 Import

코드의 대규모 기반이 있으며 부분적으로 다른 사람들이 유지 관리하고 배포할 수 있다. 각 부분은 패키지와 같은 파일 디렉토리로 구성된다. 그러나 각 부분을 별도로 명명된 패키지로 설치하는 대신 모든 부분이 공통
패키지 접두사 아래 함께 결합되기를 원한다.

본질적으로 여기서 문제는 분리되어 유지 관리되는 서브패키지의 대규모 컬렉션에 대한 네임스페이스 역할을 하는 최상위 Python 패키지를 정의하고 싶다는 것이다. 이 문제는 프레임워크 개발자가 사용자가 플러그인이나
애드온 패키지를 배포하도록 권장하고 싶어하는 대규모 애플리케이션 프레임워크에서 자주 발생한다.

공통 네임스페이스 아래 별도의 디렉토리를 통합하려면 일반 Python 패키지처럼 코드를 구성하지만 컴포넌트가 결합될 디렉토리에서 `__init__.py` 파일을 생략한다. 예를 들어, 다음과 같은 두 개의 다른
Python 코드 디렉토리가 있다고 가정하자:

```
foo-package/
    spam/
        blah.py

bar-package/
    spam/
        grok.py
```

이러한 디렉토리에서 spam이라는 이름이 공통 네임스페이스로 사용되고 있다. 어느 디렉토리에도 `__init__.py` 파일이 없다는 것을 확인하라.

이제 foo-package와 bar-package를 모두 Python 모듈 경로에 추가하고 일부 import를 시도하면 어떻게 되는지 보자:

```python
>>> import sys
>>> sys.path.extend(['foo-package', 'bar-package'])
>>> import spam.blah
>>> import spam.grok
```

마법처럼 두 개의 다른 패키지 디렉토리가 병합되고 spam.blah 또는 spam.grok을 import할 수 있다는 것을 관찰할 수 있다.

### Discussion

여기서 작동하는 메커니즘은 "namespace package"로 알려진 기능이다. 본질적으로 namespace 패키지는 표시된 것처럼 공통 네임스페이스 아래 서로 다른 코드 디렉토리를 병합하도록 설계된 특수한 종류의
패키지다.

namespace 패키지를 만드는 핵심은 공통 네임스페이스 역할을 할 최상위 디렉토리에 `__init__.py` 파일이 없는지 확인하는 것이다. 누락된 `__init__.py` 파일은 패키지 import 시 흥미로운
일이 발생하게 한다. 오류를 발생시키는 대신 인터프리터는 일치하는 패키지 이름을 포함하는 모든 디렉토리 목록을 생성하기 시작한다. 그런 다음 특수 namespace 패키지 모듈이 생성되고 디렉토리 목록의 읽기 전용
복사본이 `__path__` 변수에 저장된다:

```python
>>> import spam
>>> spam.__path__
_NamespacePath(['foo-package/spam', 'bar-package/spam'])
```

namespace 패키지의 중요한 기능은 누구나 자신의 코드로 네임스페이스를 확장할 수 있다는 것이다. 예를 들어, 다음과 같이 자신만의 코드 디렉토리를 만들었다고 가정하자:

```
my-package/
    spam/
        custom.py
```

코드 디렉토리를 다른 패키지와 함께 sys.path에 추가하면 다른 spam 패키지 디렉토리와 원활하게 병합된다:

```python
>>> import spam.custom
>>> import spam.grok
>>> import spam.blah
```

디버깅 도구로서 패키지가 namespace 패키지로 작동하는지 확인하는 주요 방법은 `__file__` 속성을 확인하는 것이다. 완전히 누락된 경우 패키지는 namespace다:

```python
>>> spam.__file__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'module' object has no attribute '__file__'
>>> spam
<module 'spam' (namespace)>
```

## 10.6. 모듈 리로드

소스를 변경했기 때문에 이미 로드된 모듈을 다시 로드하고 싶다.

이전에 로드된 모듈을 다시 로드하려면 `imp.reload()`를 사용한다:

```python
>>> import spam
>>> import imp
>>> imp.reload(spam)
<module 'spam' from './spam.py'>
```

### Discussion

모듈 리로드는 디버깅 및 개발 중에 유용한 경우가 많지만 예상대로 작동하지 않는다는 사실 때문에 프로덕션 코드에서는 일반적으로 안전하지 않다.

`reload()` 연산은 모듈의 기본 딕셔너리의 내용을 지우고 모듈의 소스 코드를 다시 실행하여 새로 고친다. 모듈 객체 자체의 identity는 변경되지 않는다. 따라서 이 작업은 프로그램에서 import된 모든
곳에서 모듈을 업데이트한다.

그러나 `reload()`는 `from module import name`과 같은 문을 사용하여 import된 정의를 업데이트하지 않는다. 다음 코드를 고려하자:

```python
# spam.py
def bar():
    print('bar')

def grok():
    print('grok')
```

대화형 세션을 시작한다:

```python
>>> import spam
>>> from spam import grok
>>> spam.bar()
bar
>>> grok()
grok
```

Python을 종료하지 않고 spam.py의 소스 코드를 편집하여 `grok()` 함수가 다음과 같이 보이도록 한다:

```python
def grok():
    print('New grok')
```

대화형 세션으로 돌아가서 reload를 수행하고 이 실험을 시도한다:

```python
>>> import imp
>>> imp.reload(spam)
<module 'spam' from './spam.py'>
>>> spam.bar()
bar
>>> grok()  # 이전 출력 주목
grok
>>> spam.grok()  # 새 출력 주목
New grok
```

이 예에서 두 개의 버전의 `grok()` 함수가 로드된 것을 볼 수 있다. 일반적으로 이것은 원하는 것이 아니며 결국 큰 두통으로 이어지는 종류의 일이다.

이러한 이유로 모듈의 리로드는 프로덕션 코드에서 피해야 할 것이다. 디버깅이나 인터프리터로 실험하고 시도하는 대화형 세션을 위해 저장하라.

## 10.7. 디렉토리나 Zip 파일을 메인 스크립트로 실행 가능하게 만들기

간단한 스크립트를 넘어 여러 파일을 포함하는 애플리케이션으로 성장한 프로그램이 있다. 사용자가 프로그램을 실행할 수 있는 쉬운 방법을 원한다.

애플리케이션 프로그램이 여러 파일로 성장한 경우 자체 디렉토리에 넣고 `__main__.py` 파일을 추가할 수 있다. 예를 들어, 다음과 같은 디렉토리를 만들 수 있다:

```
myapplication/
    spam.py
    bar.py
    grok.py
    __main__.py
```

`__main__.py`가 있으면 다음과 같이 최상위 디렉토리에서 Python 인터프리터를 실행하기만 하면 된다:

```bash
bash % python3 myapplication
```

인터프리터는 `__main__.py` 파일을 메인 프로그램으로 실행한다.

이 기법은 모든 코드를 zip 파일로 패키징하는 경우에도 작동한다:

```bash
bash % ls
spam.py bar.py grok.py __main__.py
bash % zip -r myapp.zip *.py
bash % python3 myapp.zip
... output from __main__.py ...
```

### Discussion

디렉토리나 zip 파일을 생성하고 `__main__.py` 파일을 추가하는 것은 더 큰 Python 애플리케이션을 패키징하는 한 가지 방법이다. 코드가 Python 라이브러리에 설치되는 표준 라이브러리 모듈로
사용되도록 의도되지 않았다는 점에서 패키지와 약간 다르다. 대신 누군가에게 실행하도록 전달하려는 코드 번들일 뿐이다.

## 10.8. 패키지 내 데이터 파일 읽기

패키지에 코드가 읽어야 하는 데이터 파일이 포함되어 있다. 가능한 한 이식 가능한 방식으로 이를 수행해야 한다.

다음과 같이 구성된 파일이 있는 패키지가 있다고 가정하자:

```
mypackage/
    __init__.py
    somedata.dat
    spam.py
```

이제 spam.py 파일이 somedata.dat 파일의 내용을 읽기를 원한다고 가정하자. 이를 수행하려면 다음 코드를 사용한다:

```python
# spam.py
import pkgutil

data = pkgutil.get_data(__package__, 'somedata.dat')
```

결과 변수 data는 파일의 원시 내용을 포함하는 바이트 문자열이 된다.

### Discussion

데이터 파일을 읽으려면 `open()`과 같은 내장 I/O 함수를 사용하는 코드를 작성하고 싶을 수 있다. 그러나 이 접근 방식에는 몇 가지 문제가 있다.

첫째, 패키지는 인터프리터의 현재 작업 디렉토리에 대한 제어가 거의 없다. 따라서 모든 I/O 작업은 절대 파일 이름을 사용하도록 프로그래밍해야 한다. 각 모듈에는 전체 경로가 있는 `__file__` 변수가
포함되어 있으므로 위치를 알아내는 것이 불가능하지는 않지만 지저분하다.

둘째, 패키지는 종종 .zip 또는 .egg 파일로 설치되어 파일시스템의 일반 디렉토리와 같은 방식으로 파일을 보존하지 않는다. 따라서 아카이브에 포함된 데이터 파일에 `open()`을 사용하려고 하면 전혀 작동하지
않는다.

`pkgutil.get_data()` 함수는 패키지가 어디에 또는 어떻게 설치되었는지에 관계없이 데이터 파일을 가져오기 위한 고급 도구다. 단순히 "작동"하고 파일 내용을 바이트 문자열로 반환한다.

`get_data()`의 첫 번째 인자는 패키지 이름을 포함하는 문자열이다. 직접 제공하거나 `__package__`와 같은 특수 변수를 사용할 수 있다. 두 번째 인자는 패키지 내 파일의 상대 이름이다.

## 10.9. sys.path에 디렉토리 추가

Python 코드가 sys.path에 나열된 디렉토리에 없기 때문에 import할 수 없다. Python의 경로에 새 디렉토리를 추가하고 싶지만 코드에 하드와이어링하고 싶지 않다.

sys.path에 새 디렉토리를 추가하는 두 가지 일반적인 방법이 있다. 첫째, PYTHONPATH 환경 변수를 사용하여 추가할 수 있다:

```bash
bash % env PYTHONPATH=/some/dir:/other/dir python3
Python 3.3.0 (default, Oct 4 2012, 10:17:33)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.path
['', '/some/dir', '/other/dir', ...]
```

두 번째 접근 방식은 다음과 같이 디렉토리를 나열하는 .pth 파일을 만드는 것이다:

```
# myapplication.pth
/some/dir
/other/dir
```

이 .pth 파일은 Python의 site-packages 디렉토리 중 하나에 배치되어야 한다. 일반적으로 /usr/local/lib/python3.3/site-packages 또는 ~
/.local/lib/python3.3/site-packages에 있다. 인터프리터 시작 시 파일시스템에 존재하는 한 .pth 파일에 나열된 디렉토리가 sys.path에 추가된다.

### Discussion

파일을 찾는 데 문제가 있는 경우 sys.path의 값을 수동으로 조정하는 코드를 작성하고 싶을 수 있다:

```python
import sys
sys.path.insert(0, '/some/dir')
sys.path.insert(0, '/other/dir')
```

이것은 "작동"하지만 실제로는 매우 취약하며 가능하면 피해야 한다. 이 접근 방식의 문제 중 일부는 소스에 하드코딩된 디렉토리 이름을 추가한다는 것이다. 코드가 새 위치로 이동되면 유지 관리 문제가 발생할 수 있다.
소스 코드를 편집하지 않고 조정할 수 있는 방식으로 다른 곳에서 경로를 구성하는 것이 일반적으로 훨씬 낫다.

## 10.10. 문자열로 주어진 이름을 사용하여 모듈 Import

import하고 싶은 모듈의 이름이 있지만 문자열로 유지되고 있다. 문자열에서 import 명령을 호출하고 싶다.

`importlib.import_module()` 함수를 사용하여 이름이 문자열로 주어진 모듈이나 패키지의 일부를 수동으로 import한다:

```python
>>> import importlib
>>> math = importlib.import_module('math')
>>> math.sin(2)
0.9092974268256817
>>> mod = importlib.import_module('urllib.request')
>>> u = mod.urlopen('http://www.python.org')
```

`import_module`은 단순히 import와 동일한 단계를 수행하지만 결과 모듈 객체를 결과로 반환한다. 변수에 저장하고 나중에 일반 모듈처럼 사용하기만 하면 된다.

패키지로 작업하는 경우 `import_module()`을 사용하여 상대 import를 수행할 수도 있다. 그러나 추가 인자를 제공해야 한다:

```python
import importlib
# 'from . import b'와 동일
b = importlib.import_module('.b', __package__)
```

### Discussion

`import_module()`을 사용하여 모듈을 수동으로 import하는 문제는 어떤 방식으로든 모듈을 조작하거나 래핑하는 코드를 작성할 때 가장 일반적으로 발생한다. 예를 들어, 이름으로 모듈을 로드하고 로드된
코드에 패치를 수행해야 하는 어떤 종류의 커스터마이즈된 import 메커니즘을 구현하고 있을 수 있다.

이전 코드에서는 내장 `__import__()` 함수가 import를 수행하는 데 사용되는 것을 때때로 볼 수 있다. 이것이 작동하지만 `importlib.import_module()`은 일반적으로 사용하기 더
쉽다.

## 10.11. Import Hook을 사용하여 원격 머신에서 모듈 로드

Python의 import 문을 커스터마이즈하여 원격 머신에서 모듈을 투명하게 로드할 수 있도록 하고 싶다.

먼저 보안에 대한 심각한 면책 조항이 있다. 이 레시피에서 논의된 아이디어는 어떤 종류의 추가 보안 및 인증 계층 없이는 완전히 나쁠 것이다. 그럼에도 불구하고 주요 목표는 실제로 Python의 import 문의
내부 작동을 깊이 파고드는 것이다.

이 레시피의 핵심에는 import 문의 기능을 확장하려는 욕구가 있다. 이를 수행하는 몇 가지 접근 방식이 있지만 설명을 위해 다음 Python 코드 디렉토리를 만들어 시작한다:

```
testcode/
    spam.py
    fib.py
    grok/
        __init__.py
        blah.py
```

이러한 파일의 내용은 중요하지 않지만 각 파일에 몇 가지 간단한 문과 함수를 넣어 테스트하고 import될 때 출력을 볼 수 있도록 한다.

이러한 파일에 원격 접근을 허용하는 것이 목표다. 아마도 가장 쉬운 방법은 웹 서버에 게시하는 것이다. testcode 디렉토리로 이동하여 다음과 같이 Python을 실행한다:

```bash
bash % cd testcode
bash % python3 -m http.server 15000
Serving HTTP on 0.0.0.0 port 15000 ...
```

원격 모듈을 로드하는 첫 번째 접근 방식은 이를 수행하기 위한 명시적 로딩 함수를 만드는 것이다:

```python
import imp
import urllib.request
import sys

def load_module(url):
    u = urllib.request.urlopen(url)
    source = u.read().decode('utf-8')
    mod = sys.modules.setdefault(url, imp.new_module(url))
    code = compile(source, url, 'exec')
    mod.__file__ = url
    mod.__package__ = ''
    exec(code, mod.__dict__)
    return mod
```

이 함수는 단순히 소스 코드를 다운로드하고 `compile()`을 사용하여 코드 객체로 컴파일하고 새로 생성된 모듈 객체의 딕셔너리에서 실행한다.

훨씬 더 세련된 접근 방식은 커스텀 importer를 만드는 것이다. 이를 수행하는 첫 번째 방법은 meta path importer로 알려진 것을 만드는 것이다. 다음은 예다:

```python
# urlimport.py
import sys
import importlib.abc
import imp
from urllib.request import urlopen
from urllib.error import HTTPError, URLError
from html.parser import HTMLParser

# 디버깅
import logging
log = logging.getLogger(__name__)

# 주어진 URL에서 링크 가져오기
def _get_links(url):
    class LinkParser(HTMLParser):
        def handle_starttag(self, tag, attrs):
            if tag == 'a':
                attrs = dict(attrs)
                links.add(attrs.get('href').rstrip('/'))
    links = set()
    try:
        log.debug('Getting links from %s' % url)
        u = urlopen(url)
        parser = LinkParser()
        parser.feed(u.read().decode('utf-8'))
    except Exception as e:
        log.debug('Could not get links. %s', e)
    log.debug('links: %r', links)
    return links

class UrlMetaFinder(importlib.abc.MetaPathFinder):
    def __init__(self, baseurl):
        self._baseurl = baseurl
        self._links = { }
        self._loaders = { baseurl : UrlModuleLoader(baseurl) }
    
    def find_module(self, fullname, path=None):
        log.debug('find_module: fullname=%r, path=%r', fullname, path)
        if path is None:
            baseurl = self._baseurl
        else:
            if not path[0].startswith(self._baseurl):
                return None
            baseurl = path[0]
        
        parts = fullname.split('.')
        basename = parts[-1]
        log.debug('find_module: baseurl=%r, basename=%r', baseurl, basename)
        
        # 링크 캐시 확인
        if basename not in self._links:
            self._links[baseurl] = _get_links(baseurl)
        
        # 패키지인지 확인
        if basename in self._links[baseurl]:
            log.debug('find_module: trying package %r', fullname)
            fullurl = self._baseurl + '/' + basename
            # 패키지 로드 시도 (__init__.py 접근)
            loader = UrlPackageLoader(fullurl)
            try:
                loader.load_module(fullname)
                self._links[fullurl] = _get_links(fullurl)
                self._loaders[fullurl] = UrlModuleLoader(fullurl)
                log.debug('find_module: package %r loaded', fullname)
            except ImportError as e:
                log.debug('find_module: package failed. %s', e)
                loader = None
            return loader
        
        # 일반 모듈
        filename = basename + '.py'
        if filename in self._links[baseurl]:
            log.debug('find_module: module %r found', fullname)
            return self._loaders[baseurl]
        else:
            log.debug('find_module: module %r not found', fullname)
            return None
    
    def invalidate_caches(self):
        log.debug('invalidating link cache')
        self._links.clear()

# URL용 모듈 로더
class UrlModuleLoader(importlib.abc.SourceLoader):
    def __init__(self, baseurl):
        self._baseurl = baseurl
        self._source_cache = {}
    
    def module_repr(self, module):
        return '<urlmodule %r from %r>' % (module.__name__, module.__file__)
    
    def load_module(self, fullname):
        code = self.get_code(fullname)
        mod = sys.modules.setdefault(fullname, imp.new_module(fullname))
        mod.__file__ = self.get_filename(fullname)
        mod.__loader__ = self
        mod.__package__ = fullname.rpartition('.')[0]
        exec(code, mod.__dict__)
        return mod
    
    def get_code(self, fullname):
        src = self.get_source(fullname)
        return compile(src, self.get_filename(fullname), 'exec')
    
    def get_filename(self, fullname):
        return self._baseurl + '/' + fullname.split('.')[-1] + '.py'
    
    def get_source(self, fullname):
        filename = self.get_filename(fullname)
        log.debug('loader: reading %r', filename)
        if filename in self._source_cache:
            log.debug('loader: cached %r', filename)
            return self._source_cache[filename]
        try:
            u = urlopen(filename)
            source = u.read().decode('utf-8')
            log.debug('loader: %r loaded', filename)
            self._source_cache[filename] = source
            return source
        except (HTTPError, URLError) as e:
            log.debug('loader: %r failed. %s', filename, e)
            raise ImportError("Can't load %s" % filename)
    
    def is_package(self, fullname):
        return False

# 유틸리티 함수
_installed_meta_cache = { }

def install_meta(address):
    if address not in _installed_meta_cache:
        finder = UrlMetaFinder(address)
        _installed_meta_cache[address] = finder
        sys.meta_path.append(finder)
        log.debug('%r installed on sys.meta_path', finder)

def remove_meta(address):
    if address in _installed_meta_cache:
        finder = _installed_meta_cache.pop(address)
        sys.meta_path.remove(finder)
        log.debug('%r removed from sys.meta_path', finder)
```

사용 예:

```python
>>> import urlimport
>>> urlimport.install_meta('http://localhost:15000')
>>> import fib
I'm fib
>>> import spam
I'm spam
>>> import grok.blah
I'm grok.__init__
I'm grok.blah
>>> grok.blah.__file__
'http://localhost:15000/grok/blah.py'
```

### Discussion

이 레시피를 더 자세히 논의하기 전에 Python의 모듈, 패키지 및 import 메커니즘은 전체 언어의 가장 복잡한 부분 중 하나라는 점을 강조해야 한다. importlib 모듈에 대한 문서와 PEP 302를 읽을
가치가 있다.

import 문을 확장하는 것은 간단하지만 여러 움직이는 부분을 포함한다. 가장 높은 수준에서 import 작업은 sys.meta_path에서 찾을 수 있는 "meta-path" finder 목록에 의해 처리된다.

`import fib`과 같은 문을 실행할 때 인터프리터는 sys.meta_path의 finder 객체를 살펴보고 적절한 모듈 로더를 찾기 위해 `find_module()` 메서드를 호출한다.

이 특정 솔루션은 UrlMetaFinder의 인스턴스를 sys.meta_path의 마지막 항목으로 설치하는 것을 포함한다. 모듈이 import될 때마다 sys.meta_path의 finder가 순서대로 모듈을 찾기
위해 참조된다. 이 예에서 UrlMetaFinder 인스턴스는 모듈을 일반 위치에서 찾을 수 없을 때 트리거되는 최후의 수단 finder가 된다.

## 10.12. Import 시 모듈 패치

기존 모듈의 함수에 패치를 적용하거나 decorator를 적용하고 싶다. 그러나 모듈이 실제로 import되고 다른 곳에서 사용되는 경우에만 수행하고 싶다.

여기서 본질적인 문제는 모듈이 로드될 때 이에 대한 응답으로 작업을 수행하고 싶다는 것이다. 아마도 모듈이 로드되었을 때 알려주는 어떤 종류의 콜백 함수를 트리거하고 싶을 것이다.

이 문제는 Recipe 10.11에서 논의된 것과 동일한 import hook 메커니즘을 사용하여 해결할 수 있다. 다음은 가능한 솔루션이다:

```python
# postimport.py
import importlib
import sys
from collections import defaultdict

_post_import_hooks = defaultdict(list)

class PostImportFinder:
    def __init__(self):
        self._skip = set()
    
    def find_module(self, fullname, path=None):
        if fullname in self._skip:
            return None
        self._skip.add(fullname)
        return PostImportLoader(self)

class PostImportLoader:
    def __init__(self, finder):
        self._finder = finder
    
    def load_module(self, fullname):
        importlib.import_module(fullname)
        module = sys.modules[fullname]
        for func in _post_import_hooks[fullname]:
            func(module)
        self._finder._skip.remove(fullname)
        return module

def when_imported(fullname):
    def decorate(func):
        if fullname in sys.modules:
            func(sys.modules[fullname])
        else:
            _post_import_hooks[fullname].append(func)
        return func
    return decorate

sys.meta_path.insert(0, PostImportFinder())
```

이 코드를 사용하려면 `when_imported()` decorator를 사용한다:

```python
>>> from postimport import when_imported
>>> @when_imported('threading')
... def warn_threads(mod):
...     print('Threads? Are you crazy?')
...
>>> import threading
Threads? Are you crazy?
```

더 실용적인 예로, 기존 정의에 decorator를 적용하고 싶을 수 있다:

```python
from functools import wraps
from postimport import when_imported

def logged(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print('Calling', func.__name__, args, kwargs)
        return func(*args, **kwargs)
    return wrapper

# 예
@when_imported('math')
def add_logging(mod):
    mod.cos = logged(mod.cos)
    mod.sin = logged(mod.sin)
```

### Discussion

이 레시피는 Recipe 10.11에서 논의된 import hook에 의존하지만 약간의 변형이 있다.

`@when_imported` decorator의 역할은 import 시 트리거되는 핸들러 함수를 등록하는 것이다. decorator는 sys.modules를 확인하여 모듈이 이미 로드되었는지 확인한다. 그렇다면
핸들러가 즉시 호출된다. 그렇지 않으면 핸들러가 `_post_import_hooks` 딕셔너리의 목록에 추가된다.

모듈 import 후 `_post_import_hooks`에서 보류 중인 작업을 트리거하기 위해 PostImportFinder 클래스가 sys.meta_path의 첫 번째 항목으로 설치된다.

## 10.13. 개인용 패키지 설치

타사 패키지를 설치하고 싶지만 시스템 Python에 패키지를 설치할 권한이 없다. 또는 시스템의 모든 사용자가 아닌 자신만의 사용을 위해 패키지를 설치하고 싶을 수 있다.

Python에는 일반적으로 ~/.local/lib/python3.3/site-packages와 같은 디렉토리에 있는 사용자별 설치 디렉토리가 있다. 이 디렉토리에 패키지를 강제로 설치하려면 설치 명령에
`--user` 옵션을 제공한다:

```bash
python3 setup.py install --user
```

또는

```bash
pip install --user packagename
```

사용자 site-packages 디렉토리는 일반적으로 sys.path의 시스템 site-packages 디렉토리보다 앞에 나타난다. 따라서 이 기법을 사용하여 설치한 패키지는 시스템에 이미 설치된 패키지보다
우선한다.

### Discussion

일반적으로 패키지는 시스템 전체 site-packages 디렉토리에 설치되며, 이는 /usr/local/lib/python3.3/site-packages와 같은 위치에 있다. 그러나 이렇게 하려면 일반적으로 관리자
권한과 sudo 명령 사용이 필요하다.

사용자별 디렉토리에 패키지를 설치하는 것은 커스텀 설치를 만들 수 있는 효과적인 해결 방법인 경우가 많다.

대안으로 다음 레시피에서 논의되는 가상 환경을 만들 수도 있다.

## 10.14. 새 Python 환경 생성

모듈과 패키지를 설치할 수 있는 새 Python 환경을 만들고 싶다. 그러나 Python의 새 복사본을 설치하거나 시스템 Python 설치에 영향을 줄 수 있는 변경을 하지 않고 이를 수행하고 싶다.

pyvenv 명령을 사용하여 새로운 "가상" 환경을 만들 수 있다. 이 명령은 Python 인터프리터와 동일한 디렉토리 또는 Windows의 Scripts 디렉토리에 설치된다:

```bash
bash % pyvenv Spam
bash %
```

pyvenv에 제공된 이름은 생성될 디렉토리의 이름이다. 생성 시 Spam 디렉토리는 다음과 같이 보인다:

```bash
bash % cd Spam
bash % ls
bin include lib pyvenv.cfg
bash %
```

bin 디렉토리에서 사용할 수 있는 Python 인터프리터를 찾을 수 있다:

```bash
bash % Spam/bin/python3
Python 3.3.0 (default, Oct 6 2012, 15:45:22)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> from pprint import pprint
>>> import sys
>>> pprint(sys.path)
['',
 '/usr/local/lib/python33.zip',
 '/usr/local/lib/python3.3',
 '/usr/local/lib/python3.3/plat-darwin',
 '/usr/local/lib/python3.3/lib-dynload',
 '/Users/beazley/Spam/lib/python3.3/site-packages']
```

이 인터프리터의 주요 기능은 site-packages 디렉토리가 새로 생성된 환경으로 설정되었다는 것이다. 타사 패키지를 설치하기로 결정하면 일반 시스템 site-packages 디렉토리가 아닌 여기에 설치된다.

### Discussion

가상 환경의 생성은 주로 타사 패키지의 설치 및 관리와 관련이 있다. 예에서 볼 수 있듯이 sys.path 변수에는 일반 시스템 Python의 디렉토리가 포함되어 있지만 site-packages 디렉토리는 새
디렉토리로 재배치되었다.

새로운 가상 환경을 사용하면 다음 단계는 종종 distribute 또는 pip와 같은 패키지 관리자를 설치하는 것이다. 이러한 도구와 후속 패키지를 설치할 때는 가상 환경의 일부인 인터프리터를 사용해야 한다. 이렇게
하면 패키지가 새로 생성된 site-packages 디렉토리에 설치된다.

가상 환경은 Python 설치의 복사본처럼 보일 수 있지만 실제로는 몇 개의 파일과 심볼릭 링크로만 구성된다. 모든 표준 라이브러리 파일과 인터프리터 실행 파일은 원래 Python 설치에서 가져온다. 따라서 이러한
환경을 만드는 것은 쉽고 거의 기계 리소스를 사용하지 않는다.

기본적으로 가상 환경은 완전히 깨끗하며 타사 애드온을 포함하지 않는다. 이미 설치된 패키지를 가상 환경의 일부로 포함하려면 `--system-site-packages` 옵션을 사용하여 환경을 만든다:

```bash
bash % pyvenv --system-site-packages Spam
bash %
```

## 10.15. 패키지 배포

유용한 라이브러리를 작성했으며 다른 사람들에게 제공하고 싶다.

코드를 배포하기 시작하려면 먼저 고유한 이름을 지정하고 디렉토리 구조를 정리해야 한다. 일반적인 라이브러리 패키지는 다음과 같이 보일 수 있다:

```
projectname/
    README.txt
    Doc/
        documentation.txt
    projectname/
        __init__.py
        foo.py
        bar.py
        utils/
            __init__.py
            spam.py
            grok.py
    examples/
        helloworld.py
        ...
```

패키지를 배포할 수 있는 것으로 만들려면 먼저 다음과 같은 setup.py 파일을 작성한다:

```python
# setup.py
from distutils.core import setup

setup(name='projectname',
      version='1.0',
      author='Your Name',
      author_email='you@youraddress.com',
      url='http://www.you.com/projectname',
      packages=['projectname', 'projectname.utils'],
)
```

다음으로 패키지에 포함하려는 다양한 비소스 파일을 나열하는 MANIFEST.in 파일을 만든다:

```
# MANIFEST.in
include *.txt
recursive-include examples *
recursive-include Doc *
```

setup.py 및 MANIFEST.in 파일이 패키지의 최상위 디렉토리에 나타나는지 확인한다. 이를 완료하면 다음과 같은 명령을 입력하여 소스 배포를 만들 수 있어야 한다:

```bash
bash % python3 setup.py sdist
```

이것은 플랫폼에 따라 projectname-1.0.zip 또는 projectname-1.0.tar.gz와 같은 파일을 생성한다. 모두 작동하면 이 파일은 다른 사람에게 제공하거나 Python Package
Index에 업로드하기에 적합하다.

### Discussion

순수 Python 코드의 경우 일반 setup.py 파일을 작성하는 것은 일반적으로 간단하다. 한 가지 잠재적인 함정은 패키지 소스 코드를 구성하는 모든 서브디렉토리를 수동으로 나열해야 한다는 것이다. 일반적인
실수는 패키지의 최상위 디렉토리만 나열하고 패키지 서브컴포넌트를 포함하는 것을 잊는 것이다. 이것이 setup.py의 패키지 사양에
`packages=['projectname', 'projectname.utils']` 목록이 포함된 이유다.

대부분의 Python 프로그래머가 알고 있듯이 setuptools, distribute 등을 포함한 많은 타사 패키징 옵션이 있다. 이러한 패키지 중 일부는 표준 라이브러리에 있는 distutils 라이브러리를
대체한다. 이러한 패키지에 의존하는 경우 사용자도 먼저 필요한 패키지 관리자를 설치하지 않으면 소프트웨어를 설치하지 못할 수 있다는 점에 유의하라. 이 때문에 가능한 한 간단하게 유지하면 거의 잘못될 수 없다.
최소한 표준 Python 3 설치를 사용하여 코드를 설치할 수 있는지 확인하라.

C extension과 관련된 코드의 패키징 및 배포는 상당히 복잡해질 수 있다. C extension에 대한 Chapter 15에 이에 대한 몇 가지 세부 사항이 있다. 특히 Recipe 15.2를 참조하라.