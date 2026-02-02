# CHAPTER 2 Strings and Text

거의 모든 유용한 프로그램은 데이터 파싱이나 출력 생성 등 어떤 형태로든 텍스트 처리와 관련이 있다. 이 장은 문자열 분리, 검색, 치환, 렉싱, 파싱 등 텍스트 조작과 관련된 일반적인 문제들에 초점을 맞춘다.

## 2.1. Splitting Strings on Any of Multiple Delimiters

### 문제

문자열을 필드로 분할해야 하는데, 구분자(및 그 주변의 공백)가 일관되지 않다.

### 해법

문자열 객체의 `split()` 메서드는 매우 간단한 경우를 위한 것이며, 여러 구분자나 구분자 주변의 공백을 고려하지 않는다. 더 많은 유연성이 필요할 때는 `re.split()` 메서드를 사용한다:

```python
>>> line = 'asdf fjdk; afed, fjek,asdf, foo'
>>> import re
>>> re.split(r'[;,\s]\s*', line)
['asdf', 'fjdk', 'afed', 'fjek', 'asdf', 'foo']
```

### 논의

`re.split()` 함수는 구분자로 여러 패턴을 지정할 수 있어 유용하다. 예제에서 구분자는 쉼표(,), 세미콜론(;), 또는 공백과 그 뒤의 추가 공백이다.

정규표현식 패턴이 괄호로 둘러싸인 capture group을 포함하면, 매칭된 텍스트도 결과에 포함된다:

```python
>>> fields = re.split(r'(;|,|\s)\s*', line)
>>> fields
['asdf', ' ', 'fjdk', ';', 'afed', ',', 'fjek', ',', 'asdf', ',', 'foo']
```

구분자 문자가 나중에 출력 문자열을 재구성하는 데 필요할 수 있다:

```python
>>> values = fields[::2]
>>> delimiters = fields[1::2] + ['']
>>> ''.join(v+d for v,d in zip(values, delimiters))
'asdf fjdk;afed,fjek,asdf,foo'
```

결과에 구분자 문자를 원하지 않지만 여전히 괄호를 사용해야 한다면, noncapture group `(?:...)`을 사용한다:

```python
>>> re.split(r'(?:,|;|\s)\s*', line)
['asdf', 'fjdk', 'afed', 'fjek', 'asdf', 'foo']
```

## 2.2. Matching Text at the Start or End of a String

### 문제

파일명 확장자, URL scheme 등 특정 텍스트 패턴에 대해 문자열의 시작이나 끝을 확인해야 한다.

### 해법

`str.startswith()`나 `str.endswith()` 메서드를 사용한다:

```python
>>> filename = 'spam.txt'
>>> filename.endswith('.txt')
True
>>> filename.startswith('file:')
False
>>> url = 'http://www.python.org'
>>> url.startswith('http:')
True
```

여러 선택지를 확인하려면 tuple로 제공한다:

```python
>>> filenames = ['Makefile', 'foo.c', 'bar.py', 'spam.c', 'spam.h']
>>> [name for name in filenames if name.endswith(('.c', '.h'))]
['foo.c', 'spam.c', 'spam.h']
>>> any(name.endswith('.py') for name in filenames)
True
```

### 논의

선택지가 list나 set에 있다면 먼저 `tuple()`로 변환해야 한다:

```python
>>> choices = ['http:', 'ftp:']
>>> url.startswith(tuple(choices))
True
```

슬라이스를 사용할 수도 있지만 덜 우아하다:

```python
>>> filename[-4:] == '.txt'
True
```

정규표현식도 대안이지만 간단한 매칭에는 과한 경우가 많다:

```python
>>> re.match('http:|https:|ftp:', url)
<_sre.SRE_Match object at 0x101253098>
```

## 2.3. Matching Strings Using Shell Wildcard Patterns

### 문제

Unix shell에서 일반적으로 사용되는 wildcard 패턴(예: `*.py`, `Dat[0-9]*.csv`)으로 텍스트를 매칭하고 싶다.

### 해법

`fnmatch` 모듈의 `fnmatch()`와 `fnmatchcase()` 함수를 사용한다:

```python
>>> from fnmatch import fnmatch, fnmatchcase
>>> fnmatch('foo.txt', '*.txt')
True
>>> fnmatch('foo.txt', '?oo.txt')
True
>>> fnmatch('Dat45.csv', 'Dat[0-9]*')
True
>>> names = ['Dat1.csv', 'Dat2.csv', 'config.ini', 'foo.py']
>>> [name for name in names if fnmatch(name, 'Dat*.csv')]
['Dat1.csv', 'Dat2.csv']
```

`fnmatch()`는 시스템의 파일시스템과 동일한 대소문자 구분 규칙을 사용한다. 정확한 대소문자 매칭을 원하면 `fnmatchcase()`를 사용한다:

```python
>>> fnmatchcase('foo.txt', '*.TXT')
False
```

### 논의

이 함수들은 파일명이 아닌 문자열 데이터 처리에도 사용할 수 있다:

```python
>>> addresses = [
...     '5412 N CLARK ST',
...     '1060 W ADDISON ST',
...     '1039 W GRANVILLE AVE',
...     '2122 N CLARK ST',
...     '4802 N BROADWAY',
... ]
>>> [addr for addr in addresses if fnmatchcase(addr, '* ST')]
['5412 N CLARK ST', '1060 W ADDISON ST', '2122 N CLARK ST']
>>> [addr for addr in addresses if fnmatchcase(addr, '54[0-9][0-9] *CLARK*')]
['5412 N CLARK ST']
```

## 2.4. Matching and Searching for Text Patterns

### 문제

특정 패턴에 대해 텍스트를 매칭하거나 검색하고 싶다.

### 해법

단순한 literal 텍스트라면 기본 문자열 메서드(`str.find()`, `str.endswith()`, `str.startswith()` 등)를 사용한다:

```python
>>> text = 'yeah, but no, but yeah, but no, but yeah'
>>> text.startswith('yeah')
True
>>> text.find('no')
10
```

더 복잡한 매칭에는 정규표현식과 `re` 모듈을 사용한다:

```python
>>> text1 = '11/27/2012'
>>> text2 = 'Nov 27, 2012'
>>> import re
>>> if re.match(r'\d+/\d+/\d+', text1):
...     print('yes')
yes
>>> if re.match(r'\d+/\d+/\d+', text2):
...     print('yes')
... else:
...     print('no')
no
```

같은 패턴으로 많은 매칭을 수행한다면 먼저 정규표현식을 컴파일하는 것이 좋다:

```python
>>> datepat = re.compile(r'\d+/\d+/\d+')
>>> if datepat.match(text1):
...     print('yes')
yes
```

`match()`는 항상 문자열의 시작에서 매칭을 시도한다. 모든 발생을 찾으려면 `findall()` 메서드를 사용한다:

```python
>>> text = 'Today is 11/27/2012. PyCon starts 3/13/2013.'
>>> datepat.findall(text)
['11/27/2012', '3/13/2013']
```

### 논의

정규표현식을 정의할 때 괄호로 부분을 둘러싸서 capture group을 도입하는 것이 일반적이다:

```python
>>> datepat = re.compile(r'(\d+)/(\d+)/(\d+)')
>>> m = datepat.match('11/27/2012')
>>> m.group(0)
'11/27/2012'
>>> m.group(1)
'11'
>>> m.group(2)
'27'
>>> m.group(3)
'2012'
>>> m.groups()
('11', '27', '2012')
>>> month, day, year = m.groups()
```

`findall()`은 모든 매칭을 찾아 list로 반환한다. 반복적으로 찾으려면 `finditer()`를 사용한다:

```python
>>> for m in datepat.finditer(text):
...     print(m.groups())
('11', '27', '2012')
('3', '13', '2013')
```

패턴을 지정할 때 raw string(예: `r'(\d+)/(\d+)/(\d+)'`)을 사용하는 것이 일반적이다. 이는 백슬래시 문자를 해석하지 않고 그대로 둔다.

## 2.5. Searching and Replacing Text

### 문제

문자열에서 텍스트 패턴을 검색하고 교체하고 싶다.

### 해법

단순한 literal 패턴에는 `str.replace()` 메서드를 사용한다:

```python
>>> text = 'yeah, but no, but yeah, but no, but yeah'
>>> text.replace('yeah', 'yep')
'yep, but no, but yep, but no, but yep'
```

더 복잡한 패턴에는 `re` 모듈의 `sub()` 함수/메서드를 사용한다:

```python
>>> text = 'Today is 11/27/2012. PyCon starts 3/13/2013.'
>>> import re
>>> re.sub(r'(\d+)/(\d+)/(\d+)', r'\3-\1-\2', text)
'Today is 2012-11-27. PyCon starts 2013-3-13.'
```

`\3`과 같은 백슬래시 숫자는 패턴의 capture group 번호를 참조한다.

반복적인 치환을 수행한다면 먼저 패턴을 컴파일하는 것이 좋다:

```python
>>> datepat = re.compile(r'(\d+)/(\d+)/(\d+)')
>>> datepat.sub(r'\3-\1-\2', text)
'Today is 2012-11-27. PyCon starts 2013-3-13.'
```

더 복잡한 치환을 위해 substitution callback 함수를 지정할 수 있다:

```python
>>> from calendar import month_abbr
>>> def change_date(m):
...     mon_name = month_abbr[int(m.group(1))]
...     return '{} {} {}'.format(m.group(2), mon_name, m.group(3))
>>> datepat.sub(change_date, text)
'Today is 27 Nov 2012. PyCon starts 13 Mar 2013.'
```

치환 횟수도 알고 싶다면 `re.subn()`을 사용한다:

```python
>>> newtext, n = datepat.subn(r'\3-\1-\2', text)
>>> newtext
'Today is 2012-11-27. PyCon starts 2013-3-13.'
>>> n
2
```

## 2.6. Searching and Replacing Case-Insensitive Text

### 문제

대소문자를 무시하고 텍스트를 검색하고 교체해야 한다.

### 해법

`re` 모듈을 사용하고 다양한 연산에 `re.IGNORECASE` 플래그를 제공한다:

```python
>>> text = 'UPPER PYTHON, lower python, Mixed Python'
>>> re.findall('python', text, flags=re.IGNORECASE)
['PYTHON', 'python', 'Python']
>>> re.sub('python', 'snake', text, flags=re.IGNORECASE)
'UPPER snake, lower snake, Mixed snake'
```

교체 텍스트가 매칭된 텍스트의 대소문자와 일치하지 않는 제약이 있다. 이를 수정하려면 support 함수를 사용한다:

```python
def matchcase(word):
    def replace(m):
        text = m.group()
        if text.isupper():
            return word.upper()
        elif text.islower():
            return word.lower()
        elif text[0].isupper():
            return word.capitalize()
        else:
            return word
    return replace

>>> re.sub('python', matchcase('snake'), text, flags=re.IGNORECASE)
'UPPER SNAKE, lower snake, Mixed Snake'
```

## 2.7. Specifying a Regular Expression for the Shortest Match

### 문제

정규표현식으로 텍스트 패턴을 매칭하려고 하는데, 가장 긴 매칭을 찾는다. 대신 가장 짧은 매칭을 찾고 싶다.

### 해법

이 문제는 시작 및 종료 구분자로 둘러싸인 텍스트를 매칭하려는 패턴에서 자주 발생한다:

```python
>>> str_pat = re.compile(r'\"(.*)\"')
>>> text1 = 'Computer says "no."'
>>> str_pat.findall(text1)
['no.']
>>> text2 = 'Computer says "no." Phone says "yes."'
>>> str_pat.findall(text2)
['no." Phone says "yes.']
```

정규표현식의 `*` 연산자는 greedy하므로 가장 긴 매칭을 찾는다. 패턴의 `*` 연산자 뒤에 `?` modifier를 추가하여 수정한다:

```python
>>> str_pat = re.compile(r'\"(.*?)\"')
>>> str_pat.findall(text2)
['no.', 'yes.']
```

이는 매칭을 nongreedy하게 만들어 가장 짧은 매칭을 생성한다.

### 논의

`*`나 `+` 같은 연산자 뒤에 `?`를 추가하면 매칭 알고리즘이 가장 짧은 매칭을 찾도록 강제한다.

## 2.8. Writing a Regular Expression for Multiline Patterns

### 문제

정규표현식으로 텍스트 블록을 매칭하려고 하는데, 여러 줄에 걸쳐 매칭해야 한다.

### 해법

dot(`.`)이 newline을 매칭하지 않는다는 사실을 고려하지 않는 패턴에서 이 문제가 발생한다:

```python
>>> comment = re.compile(r'/\*(.*?)\*/')
>>> text1 = '/* this is a comment */'
>>> text2 = '''/* this is a
... multiline comment */
... '''
>>> comment.findall(text1)
[' this is a comment ']
>>> comment.findall(text2)
[]
```

newline 지원을 추가하여 문제를 해결한다:

```python
>>> comment = re.compile(r'/\*((?:.|\n)*?)\*/')
>>> comment.findall(text2)
[' this is a\n multiline comment ']
```

`(?:.|\n)`은 noncapture group을 지정한다.

### 논의

`re.compile()` 함수는 `re.DOTALL` 플래그를 받는다. 이는 정규표현식의 `.`이 newline을 포함한 모든 문자를 매칭하게 한다:

```python
>>> comment = re.compile(r'/\*(.*?)\*/', re.DOTALL)
>>> comment.findall(text2)
[' this is a\n multiline comment ']
```

## 2.9. Normalizing Unicode Text to a Standard Representation

### 문제

Unicode 문자열로 작업하고 있지만, 모든 문자열이 동일한 underlying representation을 가지도록 해야 한다.

### 해법

Unicode에서 특정 문자는 둘 이상의 유효한 code point sequence로 표현될 수 있다:

```python
>>> s1 = 'Spicy Jalape\u00f1o'
>>> s2 = 'Spicy Jalapen\u0303o'
>>> s1
'Spicy Jalapeño'
>>> s2
'Spicy Jalapeño'
>>> s1 == s2
False
>>> len(s1)
14
>>> len(s2)
15
```

여러 표현은 문자열을 비교하는 프로그램에 문제가 된다. 이를 수정하려면 `unicodedata` 모듈을 사용하여 표준 표현으로 정규화해야 한다:

```python
>>> import unicodedata
>>> t1 = unicodedata.normalize('NFC', s1)
>>> t2 = unicodedata.normalize('NFC', s2)
>>> t1 == t2
True
>>> print(ascii(t1))
'Spicy Jalape\xf1o'

>>> t3 = unicodedata.normalize('NFD', s1)
>>> t4 = unicodedata.normalize('NFD', s2)
>>> t3 == t4
True
>>> print(ascii(t3))
'Spicy Jalapen\u0303o'
```

`normalize()`의 첫 번째 인자는 문자열을 정규화하는 방법을 지정한다:

- **NFC**: 문자가 완전히 composed되어야 함 (가능하면 단일 code point 사용)
- **NFD**: 문자가 combining character를 사용하여 완전히 decomposed되어야 함

Python은 특정 종류의 문자를 처리하기 위한 추가 호환성 기능을 추가하는 **NFKC**와 **NFKD** 정규화 형식도 지원한다.

### 논의

정규화는 Unicode 텍스트를 일관되고 합리적인 방식으로 처리해야 하는 코드의 중요한 부분이다. 예를 들어, 텍스트에서 모든 diacritical mark를 제거하고 싶다면:

```python
>>> t1 = unicodedata.normalize('NFD', s1)
>>> ''.join(c for c in t1 if not unicodedata.combining(c))
'Spicy Jalapeno'
```

## 2.10. Working with Unicode Characters in Regular Expressions

### 문제

정규표현식으로 텍스트를 처리하고 있지만 Unicode 문자 처리가 걱정된다.

### 해법

기본적으로 `re` 모듈은 특정 Unicode 문자 클래스에 대한 기초 지식을 가지고 있다. 예를 들어 `\d`는 이미 모든 unicode digit 문자를 매칭한다:

```python
>>> import re
>>> num = re.compile('\d+')
>>> num.match('123')  # ASCII digits
<_sre.SRE_Match object at 0x1007d9ed0>
>>> num.match('\u0661\u0662\u0663')  # Arabic digits
<_sre.SRE_Match object at 0x101234030>
```

패턴에 특정 Unicode 문자를 포함해야 한다면 Unicode 문자에 대한 일반적인 escape sequence(예: `\uFFFF` 또는 `\UFFFFFFF`)를 사용한다:

```python
>>> arabic = re.compile('[\u0600-\u06ff\u0750-\u077f\u08a0-\u08ff]+')
```

대소문자 무시 매칭과 case folding을 결합할 때 특별한 경우를 알아야 한다:

```python
>>> pat = re.compile('stra\u00dfe', re.IGNORECASE)
>>> s = 'straße'
>>> pat.match(s)  # Matches
<_sre.SRE_Match object at 0x10069d370>
>>> pat.match(s.upper())  # Doesn't match
>>> s.upper()  # Case folds
'STRASSE'
```

## 2.11. Stripping Unwanted Characters from Strings

### 문제

텍스트 문자열의 시작, 끝 또는 중간에서 공백 같은 원하지 않는 문자를 제거하고 싶다.

### 해법

`strip()` 메서드를 사용하여 문자열의 시작이나 끝에서 문자를 제거할 수 있다. `lstrip()`과 `rstrip()`은 각각 왼쪽이나 오른쪽에서 제거한다:

```python
>>> s = ' hello world \n'
>>> s.strip()
'hello world'
>>> s.lstrip()
'hello world \n'
>>> s.rstrip()
' hello world'

>>> t = '-----hello====='
>>> t.lstrip('-')
'hello====='
>>> t.strip('-=')
'hello'
```

### 논의

Stripping은 문자열 중간의 텍스트에는 적용되지 않는다. 내부 공백을 처리하려면 `replace()` 메서드나 정규표현식 치환을 사용해야 한다:

```python
>>> s = ' hello     world \n'
>>> s = s.strip()
>>> s
'hello     world'
>>> s.replace(' ', '')
'helloworld'
>>> import re
>>> re.sub('\s+', ' ', s)
'hello world'
```

문자열 stripping을 다른 반복 처리와 결합하고 싶다면 generator expression이 유용할 수 있다:

```python
with open(filename) as f:
    lines = (line.strip() for line in f)
    for line in lines:
        ...
```

## 2.12. Sanitizing and Cleaning Up Text

### 문제

텍스트 "pýtĥöñ"을 정리하고 싶다.

### 해법

간단한 수준에서는 기본 문자열 함수(`str.upper()`, `str.lower()`)를 사용하여 텍스트를 표준 케이스로 변환할 수 있다. `str.replace()`나 `re.sub()`를 사용한 간단한 교체로
특정 문자 sequence를 제거하거나 변경할 수 있다.

더 나아가려면 `str.translate()` 메서드를 사용할 수 있다:

```python
>>> s = 'pýtĥöñ\fis\tawesome\r\n'
>>> remap = {
...     ord('\t') : ' ',
...     ord('\f') : ' ',
...     ord('\r') : None  # Deleted
... }
>>> a = s.translate(remap)
>>> a
'pýtĥöñ is awesome\n'
```

모든 combining character를 제거하는 더 큰 테이블을 만들 수 있다:

```python
>>> import unicodedata
>>> import sys
>>> cmb_chrs = dict.fromkeys(c for c in range(sys.maxunicode)
...                           if unicodedata.combining(chr(c)))
>>> b = unicodedata.normalize('NFD', a)
>>> b.translate(cmb_chrs)
'python is awesome\n'
```

모든 Unicode decimal digit 문자를 ASCII로 매핑하는 translation table:

```python
>>> digitmap = { c: ord('0') + unicodedata.digit(chr(c))
...              for c in range(sys.maxunicode)
...              if unicodedata.category(chr(c)) == 'Nd' }
>>> x = '\u0661\u0662\u0663'
>>> x.translate(digitmap)
'123'
```

### 논의

I/O decoding과 encoding 함수를 사용하는 또 다른 기법도 있다:

```python
>>> a
'pýtĥöñ is awesome\n'
>>> b = unicodedata.normalize('NFD', a)
>>> b.encode('ascii', 'ignore').decode('ascii')
'python is awesome\n'
```

성능이 중요하다면, 간단한 교체의 경우 `str.replace()` 메서드가 가장 빠른 접근법이다. `translate()` 메서드는 nontrivial character-to-character remapping이나
삭제를 수행해야 할 때 매우 빠르다.

## 2.13. Aligning Text Strings

### 문제

정렬이 적용된 텍스트를 포맷해야 한다.

### 해법

문자열의 `ljust()`, `rjust()`, `center()` 메서드를 사용한다:

```python
>>> text = 'Hello World'
>>> text.ljust(20)
'Hello World         '
>>> text.rjust(20)
'         Hello World'
>>> text.center(20)
'    Hello World     '
>>> text.rjust(20,'=')
'=========Hello World'
>>> text.center(20,'*')
'****Hello World*****'
```

`format()` 함수도 쉽게 정렬에 사용할 수 있다. `<`, `>`, 또는 `^` 문자를 원하는 너비와 함께 사용한다:

```python
>>> format(text, '>20')
'         Hello World'
>>> format(text, '<20')
'Hello World         '
>>> format(text, '^20')
'    Hello World     '
>>> format(text, '=>20s')
'=========Hello World'
>>> format(text, '*^20s')
'****Hello World*****'
```

여러 값을 포맷할 때 `format()` 메서드에서도 이러한 format code를 사용할 수 있다:

```python
>>> '{:>10s} {:>10s}'.format('Hello', 'World')
'     Hello      World'
```

숫자에도 사용할 수 있다:

```python
>>> x = 1.2345
>>> format(x, '>10')
'    1.2345'
>>> format(x, '^10.2f')
'   1.23   '
```

## 2.14. Combining and Concatenating Strings

### 문제

많은 작은 문자열을 큰 문자열로 결합하고 싶다.

### 해법

결합하려는 문자열이 sequence나 iterable에 있다면 `join()` 메서드를 사용하는 것이 가장 빠르다:

```python
>>> parts = ['Is', 'Chicago', 'Not', 'Chicago?']
>>> ' '.join(parts)
'Is Chicago Not Chicago?'
>>> ','.join(parts)
'Is,Chicago,Not,Chicago?'
>>> ''.join(parts)
'IsChicagoNotChicago?'
```

몇 개의 문자열만 결합한다면 `+`를 사용해도 충분하다:

```python
>>> a = 'Is Chicago'
>>> b = 'Not Chicago?'
>>> a + ' ' + b
'Is Chicago Not Chicago?'
```

### 논의

`+` 연산자로 많은 문자열을 결합하는 것은 메모리 복사와 garbage collection으로 인해 심각하게 비효율적이다. 특히 다음과 같은 코드는 피해야 한다:

```python
s = ''
for p in parts:
    s += p
```

이는 `join()` 메서드를 사용하는 것보다 훨씬 느리다. 모든 부분을 먼저 수집한 다음 끝에서 결합하는 것이 좋다.

Generator expression을 사용한 데이터 변환과 연결:

```python
>>> data = ['ACME', 50, 91.1]
>>> ','.join(str(d) for d in data)
'ACME,50,91.1'
```

불필요한 문자열 연결을 조심해야 한다:

```python
print(a + ':' + b + ':' + c)  # Ugly
print(':'.join([a, b, c]))    # Still ugly
print(a, b, c, sep=':')       # Better
```

## 2.15. Interpolating Variables in Strings

### 문제

내장된 변수 이름이 변수 값의 문자열 표현으로 치환되는 문자열을 만들고 싶다.

### 해법

Python은 문자열에 변수 값을 직접 치환하는 것을 지원하지 않는다. 하지만 문자열의 `format()` 메서드를 사용하여 이 기능을 근사할 수 있다:

```python
>>> s = '{name} has {n} messages.'
>>> s.format(name='Guido', n=37)
'Guido has 37 messages.'
```

치환될 값이 실제로 변수에 있다면 `format_map()`과 `vars()`의 조합을 사용할 수 있다:

```python
>>> name = 'Guido'
>>> n = 37
>>> s.format_map(vars())
'Guido has 37 messages.'
```

`vars()`의 미묘한 특징은 인스턴스와도 작동한다는 것이다:

```python
>>> class Info:
...     def __init__(self, name, n):
...         self.name = name
...         self.n = n
>>> a = Info('Guido',37)
>>> s.format_map(vars(a))
'Guido has 37 messages.'
```

### 논의

`format()`과 `format_map()`의 단점은 누락된 값을 우아하게 처리하지 못한다는 것이다. `__missing__()` 메서드를 가진 대체 dictionary 클래스를 정의하여 이를 피할 수 있다:

```python
class safesub(dict):
    def __missing__(self, key):
        return '{' + key + '}'

>>> del n
>>> s.format_map(safesub(vars()))
'Guido has {n} messages.'
```

## 2.16. Reformatting Text to a Fixed Number of Columns

### 문제

긴 문자열을 사용자 지정 열 수를 채우도록 재포맷하고 싶다.

### 해법

`textwrap` 모듈을 사용하여 출력을 위한 텍스트를 재포맷한다:

```python
>>> import textwrap
>>> s = "Look into my eyes, look into my eyes, the eyes, the eyes, \
... the eyes, not around the eyes, don't look around the eyes, \
... look into my eyes, you're under."

>>> print(textwrap.fill(s, 70))
Look into my eyes, look into my eyes, the eyes, the eyes, the eyes,
not around the eyes, don't look around the eyes, look into my eyes,
you're under.

>>> print(textwrap.fill(s, 40))
Look into my eyes, look into my eyes,
the eyes, the eyes, the eyes, not around
the eyes, don't look around the eyes,
look into my eyes, you're under.

>>> print(textwrap.fill(s, 40, initial_indent='    '))
    Look into my eyes, look into my
eyes, the eyes, the eyes, the eyes, not
around the eyes, don't look around the
eyes, look into my eyes, you're under.
```

## 2.17. Handling HTML and XML Entities in Text

### 문제

`&entity;`나 `&#code;` 같은 HTML 또는 XML entity를 해당 텍스트로 교체하고 싶다. 또는 텍스트를 생성하되 특정 문자(예: `<`, `>`, `&`)를 escape해야 한다.

### 해법

텍스트를 생성하는 경우, `html.escape()` 함수를 사용하면 `<`나 `>` 같은 특수 문자를 교체하기 쉽다:

```python
>>> s = 'Elements are written as "<tag>text</tag>".'
>>> import html
>>> print(html.escape(s))
Elements are written as &quot;&lt;tag&gt;text&lt;/tag&gt;&quot;.

>>> print(html.escape(s, quote=False))
Elements are written as "&lt;tag&gt;text&lt;/tag&gt;".
```

ASCII로 텍스트를 내보내고 non-ASCII 문자에 대한 character code entity를 내장하려면 `errors='xmlcharrefreplace'` 인자를 사용한다:

```python
>>> s = 'Spicy Jalapeño'
>>> s.encode('ascii', errors='xmlcharrefreplace')
b'Spicy Jalape&#241;o'
```

텍스트의 entity를 교체하려면 HTML 또는 XML parser와 관련된 유틸리티 함수/메서드를 사용한다:

```python
>>> s = 'Spicy &quot;Jalape&#241;o&quot.'
>>> from html.parser import HTMLParser
>>> p = HTMLParser()
>>> p.unescape(s)
'Spicy "Jalapeño".'

>>> t = 'The prompt is &gt;&gt;&gt;'
>>> from xml.sax.saxutils import unescape
>>> unescape(t)
'The prompt is >>>'
```

## 2.18. Tokenizing Text

### 문제

문자열을 왼쪽에서 오른쪽으로 token stream으로 파싱하고 싶다.

### 해법

문자열을 tokenize하려면 패턴을 매칭하는 것 이상이 필요하다. 패턴의 종류를 식별하는 방법도 필요하다:

```python
import re
NAME = r'(?P<NAME>[a-zA-Z_][a-zA-Z_0-9]*)'
NUM = r'(?P<NUM>\d+)'
PLUS = r'(?P<PLUS>\+)'
TIMES = r'(?P<TIMES>\*)'
EQ = r'(?P<EQ>=)'
WS = r'(?P<WS>\s+)'

master_pat = re.compile('|'.join([NAME, NUM, PLUS, TIMES, EQ, WS]))
```

패턴 객체의 `scanner()` 메서드를 사용하여 tokenize한다:

```python
>>> scanner = master_pat.scanner('foo = 42')
>>> scanner.match()
<_sre.SRE_Match object at 0x100677738>
>>> _.lastgroup, _.group()
('NAME', 'foo')
>>> scanner.match()
<_sre.SRE_Match object at 0x100677738>
>>> _.lastgroup, _.group()
('WS', ' ')
```

이 기법을 generator로 패키징할 수 있다:

```python
from collections import namedtuple

Token = namedtuple('Token', ['type','value'])

def generate_tokens(pat, text):
    scanner = pat.scanner(text)
    for m in iter(scanner.match, None):
        yield Token(m.lastgroup, m.group())

for tok in generate_tokens(master_pat, 'foo = 42'):
    print(tok)
# Token(type='NAME', value='foo')
# Token(type='WS', value=' ')
# Token(type='EQ', value='=')
# Token(type='WS', value=' ')
# Token(type='NUM', value='42')
```

Token stream을 필터링하려면 generator expression을 사용한다:

```python
tokens = (tok for tok in generate_tokens(master_pat, text)
          if tok.type != 'WS')
```

### 논의

Tokenizing을 사용할 때 유의해야 할 중요한 세부 사항:

- 입력에 나타날 수 있는 모든 텍스트 sequence를 해당 re 패턴으로 식별해야 한다
- Master 정규표현식의 token 순서가 중요하다. 패턴이 더 긴 패턴의 substring인 경우, 더 긴 패턴을 먼저 배치해야 한다

## 2.19. Writing a Simple Recursive Descent Parser

### 문제

문법 규칙 세트에 따라 텍스트를 파싱하고 작업을 수행하거나 입력을 나타내는 추상 구문 트리를 구축해야 한다.

### 해법

간단한 산술 표현식의 문법은 EBNF 형식으로 다음과 같다:

```
expr ::= term { (+|-) term }*
term ::= factor { (*|/) factor }*
factor ::= ( expr )
        | NUM
```

Recursive descent expression evaluator 구현:

```python
import re
import collections

# Token specification
NUM = r'(?P<NUM>\d+)'
PLUS = r'(?P<PLUS>\+)'
MINUS = r'(?P<MINUS>-)'
TIMES = r'(?P<TIMES>\*)'
DIVIDE = r'(?P<DIVIDE>/)'
LPAREN = r'(?P<LPAREN>\()'
RPAREN = r'(?P<RPAREN>\))'
WS = r'(?P<WS>\s+)'

master_pat = re.compile('|'.join([NUM, PLUS, MINUS, TIMES,
                                   DIVIDE, LPAREN, RPAREN, WS]))

Token = collections.namedtuple('Token', ['type','value'])

def generate_tokens(text):
    scanner = master_pat.scanner(text)
    for m in iter(scanner.match, None):
        tok = Token(m.lastgroup, m.group())
        if tok.type != 'WS':
            yield tok

class ExpressionEvaluator:
    def parse(self,text):
        self.tokens = generate_tokens(text)
        self.tok = None
        self.nexttok = None
        self._advance()
        return self.expr()
    
    def _advance(self):
        self.tok, self.nexttok = self.nexttok, next(self.tokens, None)
    
    def _accept(self,toktype):
        if self.nexttok and self.nexttok.type == toktype:
            self._advance()
            return True
        else:
            return False
    
    def _expect(self,toktype):
        if not self._accept(toktype):
            raise SyntaxError('Expected ' + toktype)
    
    def expr(self):
        "expression ::= term { ('+'|'-') term }*"
        exprval = self.term()
        while self._accept('PLUS') or self._accept('MINUS'):
            op = self.tok.type
            right = self.term()
            if op == 'PLUS':
                exprval += right
            elif op == 'MINUS':
                exprval -= right
        return exprval
    
    def term(self):
        "term ::= factor { ('*'|'/') factor }*"
        termval = self.factor()
        while self._accept('TIMES') or self._accept('DIVIDE'):
            op = self.tok.type
            right = self.factor()
            if op == 'TIMES':
                termval *= right
            elif op == 'DIVIDE':
                termval /= right
        return termval
    
    def factor(self):
        "factor ::= NUM | ( expr )"
        if self._accept('NUM'):
            return int(self.tok.value)
        elif self._accept('LPAREN'):
            exprval = self.expr()
            self._expect('RPAREN')
            return exprval
        else:
            raise SyntaxError('Expected NUMBER or LPAREN')
```

사용 예:

```python
>>> e = ExpressionEvaluator()
>>> e.parse('2')
2
>>> e.parse('2 + 3')
5
>>> e.parse('2 + 3 * 4')
14
>>> e.parse('2 + (3 + 4) * 5')
37
```

### 논의

Recursive descent parser를 작성하는 전체 아이디어는 간단하다:

- 모든 문법 규칙을 함수나 메서드로 변환한다
- 각 메서드의 작업은 문법 규칙의 각 부분을 왼쪽에서 오른쪽으로 진행하며 token을 소비하는 것이다

구현 기법:

- 규칙의 다음 symbol이 다른 문법 규칙의 이름이면, 같은 이름의 메서드를 호출한다 (descent)
- 다음 symbol이 특정 symbol이어야 하면, 다음 token을 확인하고 정확히 일치하는지 검사한다 (`_expect()` 메서드)
- 다음 symbol이 몇 가지 선택지일 수 있으면, 각 가능성에 대해 다음 token을 확인한다 (`_accept()` 메서드)
- 반복 부분이 있는 문법 규칙의 경우, while loop로 구현한다

## 2.20. Performing Text Operations on Byte Strings

### 문제

Byte string에서 일반적인 텍스트 작업(stripping, searching, replacement)을 수행하고 싶다.

### 해법

Byte string은 이미 텍스트 문자열과 동일한 대부분의 내장 연산을 지원한다:

```python
>>> data = b'Hello World'
>>> data[0:5]
b'Hello'
>>> data.startswith(b'Hello')
True
>>> data.split()
[b'Hello', b'World']
>>> data.replace(b'Hello', b'Hello Cruel')
b'Hello Cruel World'
```

Byte array에도 작동한다:

```python
>>> data = bytearray(b'Hello World')
>>> data[0:5]
bytearray(b'Hello')
>>> data.startswith(b'Hello')
True
```

Byte string에 정규표현식 패턴 매칭을 적용할 수 있지만, 패턴 자체를 byte로 지정해야 한다:

```python
>>> data = b'FOO:BAR,SPAM'
>>> import re
>>> re.split(b'[:,]',data)
[b'FOO', b'BAR', b'SPAM']
```

### 논의

대부분의 텍스트 문자열에서 사용 가능한 거의 모든 연산이 byte string에서도 작동한다. 하지만 몇 가지 주목할 만한 차이가 있다:

**Byte string의 indexing은 개별 문자가 아닌 정수를 생성한다:**

```python
>>> a = 'Hello World'  # Text string
>>> a[0]
'H'
>>> b = b'Hello World'  # Byte string
>>> b[0]
72
>>> b[1]
101
```

**Byte string은 먼저 텍스트 문자열로 decode하지 않으면 깔끔하게 출력되지 않는다:**

```python
>>> s = b'Hello World'
>>> print(s)
b'Hello World'
>>> print(s.decode('ascii'))
Hello World
```

**Byte string에는 문자열 포맷 연산을 사용할 수 없다:**

```python
>>> b'%10s %10d %10.2f' % (b'ACME', 100, 490.1)
TypeError: unsupported operand type(s) for %: 'bytes' and 'tuple'
```

Byte string에 포맷을 적용하려면 일반 텍스트 문자열을 사용하고 인코딩해야 한다:

```python
>>> '{:10s} {:10d} {:10.2f}'.format('ACME', 100, 490.1).encode('ascii')
b'ACME              100     490.10'
```

Byte string을 사용하면 특히 파일시스템과 관련된 특정 연산의 의미가 변경될 수 있다. 텍스트로 작업하는 경우, 프로그램에서 byte string이 아닌 일반 텍스트 문자열을 사용해야 한다.