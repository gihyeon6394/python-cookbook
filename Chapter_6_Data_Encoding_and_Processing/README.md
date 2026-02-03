# CHAPTER 6 Data Encoding and Processing

이 챕터는 CSV, JSON, XML, 바이너리 레코드 등 다양한 공통 encoding 형식의 데이터를 Python으로 처리하는 패턴을 다룬다. 특정 알고리즘보다는 프로그램과 데이터 간의 입출력 문제에 초점을
맞준다.

## 6.1 CSV 데이터 읽기와 쓰기

`csv` 모듈을 사용한다. 수동으로 `split(',')`을 사용하면 따옴표 내 쉼표 등 복잡한 CSV 규칙 처리가 불가능하다.

### 읽기

```python
import csv

# tuple로 읽기
with open('stocks.csv') as f:
    f_csv = csv.reader(f)
    headers = next(f_csv)
    for row in f_csv:
        # row[0] = Symbol, row[4] = Change 등 인덱스 접근
        ...

# namedtuple로 읽기 (컬럼명으로 접근 가능)
from collections import namedtuple
with open('stocks.csv') as f:
    f_csv = csv.reader(f)
    headings = next(f_csv)
    Row = namedtuple('Row', headings)
    for r in f_csv:
        row = Row(*r)
        print(row.Symbol, row.Change)

# dict로 읽기
with open('stocks.csv') as f:
    f_csv = csv.DictReader(f)
    for row in f_csv:
        print(row['Symbol'], row['Change'])
```

### 쓰기

```python
headers = ['Symbol', 'Price', 'Date', 'Time', 'Change', 'Volume']
rows = [('AA', 39.48, '6/11/2007', '9:36am', -0.18, 181800), ...]

with open('stocks.csv', 'w') as f:
    f_csv = csv.writer(f)
    f_csv.writerow(headers)
    f_csv.writerows(rows)

# dict 리스트로 쓰기
with open('stocks.csv', 'w') as f:
    f_csv = csv.DictWriter(f, headers)
    f_csv.writeheader()
    f_csv.writerows(rows)   # rows는 dict의 리스트
```

### 주의사항

`csv` 모듈은 모든 값을 **문자열**로 반환한다. 타입 변환이 필요하면 직접 구현해야 한다.

```python
col_types = [str, float, str, str, float, int]
with open('stocks.csv') as f:
    f_csv = csv.reader(f)
    headers = next(f_csv)
    for row in f_csv:
        row = tuple(convert(value) for convert, value in zip(col_types, row))
```

탭 구분 등 다른 delimiter는 `delimiter` 파라미터로 지정한다. 대량 데이터 분석이 목적이면 **Pandas**의 `read_csv()`를 고려하자.

## 6.2 JSON 데이터 읽기와 쓰기

`json` 모듈의 `dumps()`/`loads()` (문자열), `dump()`/`load()` (파일)을 사용한다.

```python
import json

data = {'name': 'ACME', 'shares': 100, 'price': 542.23}

# 문자열로 변환 / 복원
json_str = json.dumps(data)
data = json.loads(json_str)

# 파일로 저장 / 읽기
with open('data.json', 'w') as f:
    json.dump(data, f)
with open('data.json', 'r') as f:
    data = json.load(f)
```

### Python ↔ JSON 타입 매핑

| Python          | JSON    |
|-----------------|---------|
| `True`          | `true`  |
| `False`         | `false` |
| `None`          | `null`  |
| `dict`          | object  |
| `list`, `tuple` | array   |

### 고급 옵션

```python
# 보기 좋은 출력
print(json.dumps(data, indent=4, sort_keys=True))

# OrderedDict로 키 순서 보존
from collections import OrderedDict
data = json.loads(json_str, object_pairs_hook=OrderedDict)

# JSON을 커스텀 객체로 변환
class JSONObject:
    def __init__(self, d):
        self.__dict__ = d

data = json.loads(json_str, object_hook=JSONObject)
print(data.name)   # 속성 접근 가능
```

### 커스텀 객체의 serialization

기본적으로 클래스 인스턴스는 JSON serializable이 아니다. `default` 파라미터로 커스텀 변환 함수를 제공하면 된다.

```python
def serialize_instance(obj):
    d = {'__classname__': type(obj).__name__}
    d.update(vars(obj))
    return d

def unserialize_object(d):
    clsname = d.pop('__classname__', None)
    if clsname:
        cls = classes[clsname]          # 클래스 이름 → 클래스 매핑
        obj = cls.__new__(cls)
        for key, value in d.items():
            setattr(obj, key, value)
        return obj
    return d

s = json.dumps(p, default=serialize_instance)
a = json.loads(s, object_hook=unserialize_object)
```

## 6.3 간단한 XML 데이터 파싱

`xml.etree.ElementTree` 모듈을 사용한다. `parse()`로 문서를 읽고, `find()`, `iterfind()`, `findtext()`로 원하는 tag를 검색한다.

```python
from xml.etree.ElementTree import parse

doc = parse('feed.xml')
for item in doc.iterfind('channel/item'):
    title = item.findtext('title')
    date  = item.findtext('pubDate')
    link  = item.findtext('link')
    print(title, date, link)
```

각 Element는 `.tag` (tag 이름), `.text` (내부 텍스트), `.get()` (속성 값) 등의 속성·메서드를 가진다. 더 빠르고 표준 준수가 중요하면 **lxml**을 사용하면 된다 (동일한
API).

## 6.4 거대한 XML 파일의 증분 파싱

전체 XML을 메모리에 읽으면 용량이 방대할 수 있다. `iterparse()`를 사용하여 증분적으로 처리하고, 처리 완료된 element를 즉시 제거하여 메모리를 절약한다.

```python
from xml.etree.ElementTree import iterparse

def parse_and_remove(filename, path):
    path_parts = path.split('/')
    doc = iterparse(filename, ('start', 'end'))
    next(doc)   # root element 건너뛰기
    tag_stack, elem_stack = [], []
    for event, elem in doc:
        if event == 'start':
            tag_stack.append(elem.tag)
            elem_stack.append(elem)
        elif event == 'end':
            if tag_stack == path_parts:
                yield elem
                elem_stack[-2].remove(elem)   # 핵심: 부모에서 제거 → 메모리 해제
            try:
                tag_stack.pop()
                elem_stack.pop()
            except IndexError:
                pass
```

실제 사용 시 동일 데이터에 대해 전체 파싱은 ~450 MB, 증분 파싱은 ~7 MB의 메모리를 사용한다. 속도는 약 2배 차이 있지만 메모리는 60배 절약된다.

## 6.5 Dictionary를 XML로 변환

`Element` 클래스를 사용하여 XML을 생성한다. 문자열 직접 조합 대신 Element를 사용하면 특수 문자(`<`, `>` 등)가 자동으로 이스케이프된다.

```python
from xml.etree.ElementTree import Element, tostring

def dict_to_xml(tag, d):
    elem = Element(tag)
    for key, val in d.items():
        child = Element(key)
        child.text = str(val)
        elem.append(child)
    return elem

s = {'name': 'GOOG', 'shares': 100, 'price': 490.1}
e = dict_to_xml('stock', s)
print(tostring(e))
# b'<stock><name>GOOG</name><shares>100</shares><price>490.1</price></stock>'
```

속성 추가는 `e.set('_id', '1234')`로 한다. 원소 순서가 중요하면 `OrderedDict`를 사용하자.

## 6.6 XML 파싱·수정·저장

`ElementTree`로 읽은 후, 부모 element를 리스트처럼 조작하여 수정한다.

```python
from xml.etree.ElementTree import parse, Element

doc  = parse('pred.xml')
root = doc.getroot()

root.remove(root.find('sri'))          # element 삭제
e = Element('spam')
e.text = 'This is a test'
root.insert(2, e)                      # 특정 위치에 삽입

doc.write('newpred.xml', xml_declaration=True)  # 파일로 저장
```

## 6.7 Namespace가 있는 XML 파싱

Namespace가 포함된 tag는 `{URI}tagname` 형태로 완전한 경로를 지정해야 한다. 이를 깔끔하게 관리하려면 utility 클래스를 사용한다.

```python
class XMLNamespaces:
    def __init__(self, **kwargs):
        self.namespaces = {name: '{'+uri+'}' for name, uri in kwargs.items()}
    def __call__(self, path):
        return path.format_map(self.namespaces)

ns = XMLNamespaces(html='http://www.w3.org/1999/xhtml')
doc.findtext(ns('content/{html}html/{html}head/{html}title'))
# → 'Hello World'
```

## 6.8 관계형 데이터베이스와의 상호작용

Python의 표준 DB-API(PEP 249)를 따라 SQL 쿼리로 데이터를 처리한다. 아래 예시는 `sqlite3`을 사용했지만 MySQL, PostgreSQL 등도 동일한 인터페이스로 작동한다.

```python
import sqlite3

db = sqlite3.connect('database.db')
c  = db.cursor()

# 테이블 생성 및 데이터 삽입
c.execute('CREATE TABLE portfolio (symbol TEXT, shares INTEGER, price REAL)')
c.executemany('INSERT INTO portfolio VALUES (?,?,?)', stocks)
db.commit()

# 조회
for row in db.execute('SELECT * FROM portfolio'):
    print(row)

# 사용자 입력을 파라미터로 받는 경우 반드시 ? 사용
for row in db.execute('SELECT * FROM portfolio WHERE price >= ?', (100,)):
    print(row)
```

**SQL injection 방지:** `%` 포맷팅이나 `.format()`으로 SQL 문자열을 조합하는 것은 **절대 하지 않아야 한다**. `?` (또는 해당 모듈의 paramstyle) wildcard를
반드시 사용한다.

더 복잡한 작업이면 **SQLAlchemy** 같은 ORM을 고려하자.

## 6.9 Hexadecimal Encoding/Decoding

`binascii` 또는 `base64` 모듈을 사용한다.

```python
import binascii, base64

s = b'hello'

# binascii (대·소문자 구분 없음)
h = binascii.b2a_hex(s)       # b'68656c6c6f'
binascii.a2b_hex(h)           # b'hello'

# base64 모듈 (대문자만 사용)
h = base64.b16encode(s)       # b'68656C6C6F'
base64.b16decode(h)           # b'hello'
```

출력은 항상 byte string이므로 텍스트로 출력하려면 `.decode('ascii')`가 필요하다.

## 6.10 Base64 Encoding/Decoding

```python
import base64

s = b'hello'
a = base64.b64encode(s)       # b'aGVsbG8='
base64.b64decode(a)           # b'hello'

# 텍스트와 혼용 시 ascii로 decode
a = base64.b64encode(s).decode('ascii')   # 'aGVsbG8='
```

## 6.11 구조체 배열(Binary Arrays of Structures) 읽기·쓰기

`struct` 모듈의 `Struct` 클래스를 사용하여 바이너리 레코드를 pack/unpack한다.

### 구조체 format 코드 참고

| 코드  | 타입                  | 바이트 |
|-----|---------------------|-----|
| `i` | 32비트 정수             | 4   |
| `d` | 64비트 float (double) | 8   |
| `f` | 32비트 float          | 4   |
| `<` | Little endian       | —   |
| `>` | Big endian          | —   |
| `!` | Network byte order  | —   |

### 쓰기

```python
from struct import Struct

def write_records(records, format, f):
    record_struct = Struct(format)
    for r in records:
        f.write(record_struct.pack(*r))

records = [(1, 2.3, 4.5), (6, 7.8, 9.0), (12, 13.4, 56.7)]
with open('data.b', 'wb') as f:
    write_records(records, '<idd', f)
```

### 읽기 (두 가지 방식)

```python
# 방식 1: 청크 단위 증분 읽기
def read_records(format, f):
    record_struct = Struct(format)
    chunks = iter(lambda: f.read(record_struct.size), b'')
    return (record_struct.unpack(chunk) for chunk in chunks)

# 방식 2: 전체를 읽은 후 unpack_from으로 offset별 처리 (메모리 복사 없음)
def unpack_records(format, data):
    record_struct = Struct(format)
    return (record_struct.unpack_from(data, offset)
            for offset in range(0, len(data), record_struct.size))
```

`unpack_from()`은 큰 byte 문자열에서 직접 필드를 추출하여 임시 slice·복사를 피할 수 있어 효율적이다. 결과를 `namedtuple`로 감싸면 속성 접근이 가능하다. 대량 배열이면 **numpy
**의 `np.fromfile()`도 고려하자.

## 6.12 중첩·가변 크기 바이너리 구조체 읽기

복잡한 바이너리 파일 형식을 처리하기 위해 **descriptor**, **metaclass**, **memoryview**를 활용한 고급 프레임워크를 구축한다.

### 핵심 컴포넌트

```python
import struct

# 단일 필드를 나타내는 descriptor
class StructField:
    def __init__(self, format, offset):
        self.format = format
        self.offset = offset
    def __get__(self, instance, cls):
        if instance is None:
            return self
        r = struct.unpack_from(self.format, instance._buffer, self.offset)
        return r[0] if len(r) == 1 else r

# metaclass: _fields_ 정의를 자동으로 StructField descriptor로 변환
class StructureMeta(type):
    def __init__(self, clsname, bases, clsdict):
        fields = getattr(self, '_fields_', [])
        byte_order, offset = '', 0
        for format, fieldname in fields:
            if isinstance(format, StructureMeta):   # 중첩 구조체
                setattr(self, fieldname, NestedStruct(fieldname, format, offset))
                offset += format.struct_size
            else:
                if format.startswith(('<', '>', '!', '@')):
                    byte_order = format[0]
                    format = format[1:]
                format = byte_order + format
                setattr(self, fieldname, StructField(format, offset))
                offset += struct.calcsize(format)
        setattr(self, 'struct_size', offset)

class Structure(metaclass=StructureMeta):
    def __init__(self, bytedata):
        self._buffer = memoryview(bytedata)   # memoryview → slice 시 복사 없음
    @classmethod
    def from_file(cls, f):
        return cls(f.read(cls.struct_size))
```

### 사용 예시: 중첩 구조체

```python
class Point(Structure):
    _fields_ = [('<d', 'x'), ('d', 'y')]

class PolyHeader(Structure):
    _fields_ = [
        ('<i', 'file_code'),
        (Point, 'min'),          # 중첩 구조체
        (Point, 'max'),          # 중첩 구조체
        ('i', 'num_polys')
    ]

phead = PolyHeader.from_file(f)
print(phead.min.x, phead.max.y)   # 중첩 속성 접근
```

### 가변 크기 레코드: SizedRecord

```python
class SizedRecord:
    def __init__(self, bytedata):
        self._buffer = memoryview(bytedata)

    @classmethod
    def from_file(cls, f, size_fmt, includes_size=True):
        sz_nbytes = struct.calcsize(size_fmt)
        sz, = struct.unpack(size_fmt, f.read(sz_nbytes))
        buf = f.read(sz - includes_size * sz_nbytes)
        return cls(buf)

    def iter_as(self, code):
        if isinstance(code, str):
            s = struct.Struct(code)
            for off in range(0, len(self._buffer), s.size):
                yield s.unpack_from(self._buffer, off)
        elif isinstance(code, StructureMeta):
            size = code.struct_size
            for off in range(0, len(self._buffer), size):
                yield code(self._buffer[off:off+size])
```

### 최종 완성: read_polys

```python
def read_polys(filename):
    polys = []
    with open(filename, 'rb') as f:
        phead = PolyHeader.from_file(f)
        for n in range(phead.num_polys):
            rec  = SizedRecord.from_file(f, '<i')
            poly = [(p.x, p.y) for p in rec.iter_as(Point)]
            polys.append(poly)
    return polys
```

### 설계 원칙 요약

| 원칙                    | 구현                                                             |
|-----------------------|----------------------------------------------------------------|
| **Lazy unpacking**    | `Structure.__init__`에서 memoryview만 저장; 실제 unpack은 속성 접근 시에만 발생 |
| **Byte order sticky** | `_fields_`에서 한 번 지정하면 이후 필드에 계속 적용                             |
| **Zero-copy slicing** | `memoryview` 사용 → 중첩 구조체 slice 시 메모리 복사 없음                     |
| **Metaclass**         | `_fields_` 선언만으로 descriptor와 offset을 자동 생성                     |

## 6.13 데이터 요약과 통계

대량 데이터 분석이 필요하면 **Pandas** 라이브러리를 사용한다.

```python
import pandas

# CSV 읽기
rats = pandas.read_csv('rats.csv', skip_footer=1)

# 특정 열의 고유 값
rats['Current Activity'].unique()

# 조건으로 필터링
crew = rats[rats['Current Activity'] == 'Dispatch Crew']

# 값 빈도 상위 10건
crew['ZIP Code'].value_counts()[:10]

# 날짜별 그룹핑 및 건수 집계
dates       = crew.groupby('Completion Date')
date_counts = dates.size()
date_counts.sort()
```

Pandas는 CSV 로드, 그룹핑, 통계 집계, 필터링 등 데이터 분석의 거의 모든 작업을 간결하게 처리할 수 있다.