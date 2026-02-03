# CHAPTER 15 C Extensions

이 장에서는 Python에서 C 코드에 접근하는 문제를 다룬다. Python의 많은 내장 라이브러리가 C로 작성되어 있으며, C 접근은 Python이 기존 라이브러리와 통신하게 만드는 중요한 부분이다. Python은
광범위한 C 프로그래밍 API를 제공하지만, C를 다루는 여러 접근 방식이 있다.

이 장에서 사용할 샘플 C 코드:

```c
/* sample.c */
#include <math.h>

/* 최대공약수 계산 */
int gcd(int x, int y) {
    int g = y;
    while (x > 0) {
        g = x;
        x = y % x;
        y = g;
    }
    return g;
}

/* Mandelbrot 집합 테스트 */
int in_mandel(double x0, double y0, int n) {
    double x=0,y=0,xtemp;
    while (n > 0) {
        xtemp = x*x - y*y + x0;
        y = 2*x*y + y0;
        x = xtemp;
        n -= 1;
        if (x*x + y*y > 4) return 0;
    }
    return 1;
}

/* 두 수 나누기 */
int divide(int a, int b, int *remainder) {
    int quot = a / b;
    *remainder = a % b;
    return quot;
}

/* 배열 평균 */
double avg(double *a, int n) {
    int i;
    double total = 0.0;
    for (i = 0; i < n; i++) {
        total += a[i];
    }
    return total / n;
}

/* C 데이터 구조 */
typedef struct Point {
    double x,y;
} Point;

/* C 데이터 구조를 사용하는 함수 */
double distance(Point *p1, Point *p2) {
    return hypot(p1->x - p2->x, p1->y - p2->y);
}
```

## 15.1. ctypes를 이용한 C 코드 접근

### 문제

shared library나 DLL로 컴파일된 소수의 C 함수가 있다. 추가 C 코드를 작성하거나 서드파티 확장 도구를 사용하지 않고 순수하게 Python에서 이러한 함수를 호출하고 싶다.

### 해결

C 코드가 관련된 작은 문제의 경우 Python의 표준 라이브러리인 ctypes 모듈을 사용하는 것이 충분히 쉽다.

```python
# sample.py
import ctypes
import os

# .so 파일을 현재 파일과 같은 디렉토리에서 찾기
_file = 'libsample.so'
_path = os.path.join(*(os.path.split(__file__)[:-1] + (_file,)))
_mod = ctypes.cdll.LoadLibrary(_path)

# int gcd(int, int)
gcd = _mod.gcd
gcd.argtypes = (ctypes.c_int, ctypes.c_int)
gcd.restype = ctypes.c_int

# int in_mandel(double, double, int)
in_mandel = _mod.in_mandel
in_mandel.argtypes = (ctypes.c_double, ctypes.c_double, ctypes.c_int)
in_mandel.restype = ctypes.c_int

# int divide(int, int, int *)
_divide = _mod.divide
_divide.argtypes = (ctypes.c_int, ctypes.c_int, ctypes.POINTER(ctypes.c_int))
_divide.restype = ctypes.c_int

def divide(x, y):
    rem = ctypes.c_int()
    quot = _divide(x, y, rem)
    return quot, rem.value

# 배열을 위한 특별한 타입 정의
class DoubleArrayType:
    def from_param(self, param):
        typename = type(param).__name__
        if hasattr(self, 'from_' + typename):
            return getattr(self, 'from_' + typename)(param)
        elif isinstance(param, ctypes.Array):
            return param
        else:
            raise TypeError("Can't convert %s" % typename)
    
    def from_array(self, param):
        if param.typecode != 'd':
            raise TypeError('must be an array of doubles')
        ptr, _ = param.buffer_info()
        return ctypes.cast(ptr, ctypes.POINTER(ctypes.c_double))
    
    def from_list(self, param):
        val = ((ctypes.c_double)*len(param))(*param)
        return val
    
    from_tuple = from_list
    
    def from_ndarray(self, param):
        return param.ctypes.data_as(ctypes.POINTER(ctypes.c_double))

DoubleArray = DoubleArrayType()
_avg = _mod.avg
_avg.argtypes = (DoubleArray, ctypes.c_int)
_avg.restype = ctypes.c_double

def avg(values):
    return _avg(values, len(values))

# struct Point { }
class Point(ctypes.Structure):
    _fields_ = [('x', ctypes.c_double),
                ('y', ctypes.c_double)]

# double distance(Point *, Point *)
distance = _mod.distance
distance.argtypes = (ctypes.POINTER(Point), ctypes.POINTER(Point))
distance.restype = ctypes.c_double
```

사용 예:

```python
>>> import sample
>>> sample.gcd(35,42)
7
>>> sample.in_mandel(0,0,500)
1
>>> sample.divide(42,8)
(5, 2)
>>> sample.avg([1,2,3])
2.0
>>> p1 = sample.Point(1,2)
>>> p2 = sample.Point(4,5)
>>> sample.distance(p1,p2)
4.242640687119285
```

### 논의

라이브러리를 로드한 후에는 특정 심볼을 추출하고 타입 시그니처를 붙여야 한다. `.argtypes` 속성은 함수의 입력 인수를 포함하는 tuple이고, `.restype`은 반환 타입이다.

포인터를 포함하는 인수의 경우, 호환 가능한 ctypes 객체를 생성하고 전달해야 한다:

```python
>>> x = ctypes.c_int()
>>> divide(10, 3, x)
3
>>> x.value
1
```

## 15.2. 간단한 C Extension Module 작성

### 문제

Python의 extension API를 직접 사용하여 간단한 C extension module을 작성하고 싶다.

### 해결

간단한 C 코드의 경우, 수작업 extension module을 만드는 것은 간단하다. 먼저 적절한 헤더 파일이 있는지 확인한다:

```c
/* sample.h */
#include <math.h>

extern int gcd(int, int);
extern int in_mandel(double x0, double y0, int n);
extern int divide(int a, int b, int *remainder);
extern double avg(double *a, int n);

typedef struct Point {
    double x,y;
} Point;

extern double distance(Point *p1, Point *p2);
```

Extension 함수의 기본을 보여주는 샘플 extension module:

```c
#include "Python.h"
#include "sample.h"

/* int gcd(int, int) */
static PyObject *py_gcd(PyObject *self, PyObject *args) {
    int x, y, result;
    if (!PyArg_ParseTuple(args,"ii", &x, &y)) {
        return NULL;
    }
    result = gcd(x,y);
    return Py_BuildValue("i", result);
}

/* int in_mandel(double, double, int) */
static PyObject *py_in_mandel(PyObject *self, PyObject *args) {
    double x0, y0;
    int n;
    int result;
    if (!PyArg_ParseTuple(args, "ddi", &x0, &y0, &n)) {
        return NULL;
    }
    result = in_mandel(x0,y0,n);
    return Py_BuildValue("i", result);
}

/* int divide(int, int, int *) */
static PyObject *py_divide(PyObject *self, PyObject *args) {
    int a, b, quotient, remainder;
    if (!PyArg_ParseTuple(args, "ii", &a, &b)) {
        return NULL;
    }
    quotient = divide(a,b, &remainder);
    return Py_BuildValue("(ii)", quotient, remainder);
}

/* Module method table */
static PyMethodDef SampleMethods[] = {
    {"gcd", py_gcd, METH_VARARGS, "Greatest common divisor"},
    {"in_mandel", py_in_mandel, METH_VARARGS, "Mandelbrot test"},
    {"divide", py_divide, METH_VARARGS, "Integer division"},
    { NULL, NULL, 0, NULL}
};

/* Module structure */
static struct PyModuleDef samplemodule = {
    PyModuleDef_HEAD_INIT,
    "sample",           /* name of module */
    "A sample module",  /* Doc string (may be NULL) */
    -1,                 /* Size of per-interpreter state or -1 */
    SampleMethods       /* Method table */
};

/* Module initialization function */
PyMODINIT_FUNC
PyInit_sample(void) {
    return PyModule_Create(&samplemodule);
}
```

Extension module을 빌드하기 위한 setup.py 파일:

```python
# setup.py
from distutils.core import setup, Extension

setup(name='sample',
      ext_modules=[
          Extension('sample',
                    ['pysample.c'],
                    include_dirs = ['/some/dir'],
                    define_macros = [('FOO','1')],
                    undef_macros = ['BAR'],
                    library_dirs = ['/usr/local/lib'],
                    libraries = ['sample'])
      ])
```

빌드 명령:

```bash
bash % python3 setup.py build_ext --inplace
```

### 논의

Extension module에서 작성하는 함수는 모두 공통 프로토타입으로 작성된다:

```c
static PyObject *py_func(PyObject *self, PyObject *args) {
    ...
}
```

`PyArg_ParseTuple()` 함수는 Python에서 C 표현으로 값을 변환하는 데 사용된다. 입력으로 정수는 "i", double은 "d"와 같은 필요한 값을 나타내는 format string을 받는다.

`Py_BuildValue()` 함수는 C 데이터 타입에서 Python 객체를 생성하는 데 사용된다. 예제:

```c
return Py_BuildValue("i", 34);      // 정수 반환
return Py_BuildValue("d", 3.4);     // double 반환
return Py_BuildValue("s", "Hello"); // Null-terminated UTF-8 string
return Py_BuildValue("(ii)", 3, 4); // Tuple (3, 4)
```

## 15.3. 배열을 다루는 Extension Function 작성

### 문제

array module이나 NumPy 같은 라이브러리로 생성될 수 있는 연속 데이터 배열을 다루는 C extension function을 작성하고 싶다.

### 해결

이식 가능한 방식으로 배열을 받고 처리하려면 Buffer Protocol을 사용하는 코드를 작성해야 한다:

```c
/* Call double avg(double *, int) */
static PyObject *py_avg(PyObject *self, PyObject *args) {
    PyObject *bufobj;
    Py_buffer view;
    double result;
    
    /* Get the passed Python object */
    if (!PyArg_ParseTuple(args, "O", &bufobj)) {
        return NULL;
    }
    
    /* Attempt to extract buffer information from it */
    if (PyObject_GetBuffer(bufobj, &view,
                          PyBUF_ANY_CONTIGUOUS | PyBUF_FORMAT) == -1) {
        return NULL;
    }
    
    if (view.ndim != 1) {
        PyErr_SetString(PyExc_TypeError, "Expected a 1-dimensional array");
        PyBuffer_Release(&view);
        return NULL;
    }
    
    /* Check the type of items in the array */
    if (strcmp(view.format,"d") != 0) {
        PyErr_SetString(PyExc_TypeError, "Expected an array of doubles");
        PyBuffer_Release(&view);
        return NULL;
    }
    
    /* Pass the raw buffer and size to the C function */
    result = avg(view.buf, view.shape[0]);
    
    /* Indicate we're done working with the buffer */
    PyBuffer_Release(&view);
    return Py_BuildValue("d", result);
}
```

사용 예:

```python
>>> import array
>>> avg(array.array('d',[1,2,3]))
2.0
>>> import numpy
>>> avg(numpy.array([1.0,2.0,3.0]))
2.0
```

### 논의

`PyBuffer_GetBuffer()` 함수가 핵심이다. 임의의 Python 객체가 주어지면 기본 메모리 표현에 대한 정보를 얻으려고 시도한다.

`Py_buffer` 구조는 기본 메모리에 대한 정보로 채워진다:

```c
typedef struct bufferinfo {
    void *buf;              /* Pointer to buffer memory */
    PyObject *obj;          /* Python object that is the owner */
    Py_ssize_t len;         /* Total size in bytes */
    Py_ssize_t itemsize;    /* Size in bytes of a single item */
    int readonly;           /* Read-only access flag */
    int ndim;               /* Number of dimensions */
    char *format;           /* struct code of a single item */
    Py_ssize_t *shape;      /* Array containing dimensions */
    Py_ssize_t *strides;    /* Array containing strides */
    Py_ssize_t *suboffsets; /* Array containing suboffsets */
} Py_buffer;
```

## 15.4. C Extension Module에서 Opaque Pointer 관리

### 문제

C 데이터 구조에 대한 포인터를 처리해야 하는 extension module이 있지만, 구조의 내부 세부 사항을 Python에 노출하고 싶지 않다.

### 해결

Opaque 데이터 구조는 capsule 객체 안에 래핑하여 쉽게 처리할 수 있다:

```c
/* Destructor function for points */
static void del_Point(PyObject *obj) {
    free(PyCapsule_GetPointer(obj,"Point"));
}

/* Utility functions */
static Point *PyPoint_AsPoint(PyObject *obj) {
    return (Point *) PyCapsule_GetPointer(obj, "Point");
}

static PyObject *PyPoint_FromPoint(Point *p, int must_free) {
    return PyCapsule_New(p, "Point", must_free ? del_Point : NULL);
}

/* Create a new Point object */
static PyObject *py_Point(PyObject *self, PyObject *args) {
    Point *p;
    double x,y;
    if (!PyArg_ParseTuple(args,"dd",&x,&y)) {
        return NULL;
    }
    p = (Point *) malloc(sizeof(Point));
    p->x = x;
    p->y = y;
    return PyPoint_FromPoint(p, 1);
}

static PyObject *py_distance(PyObject *self, PyObject *args) {
    Point *p1, *p2;
    PyObject *py_p1, *py_p2;
    double result;
    
    if (!PyArg_ParseTuple(args,"OO",&py_p1, &py_p2)) {
        return NULL;
    }
    if (!(p1 = PyPoint_AsPoint(py_p1))) {
        return NULL;
    }
    if (!(p2 = PyPoint_AsPoint(py_p2))) {
        return NULL;
    }
    result = distance(p1,p2);
    return Py_BuildValue("d", result);
}
```

사용 예:

```python
>>> import sample
>>> p1 = sample.Point(2,3)
>>> p2 = sample.Point(4,5)
>>> p1
<capsule object "Point" at 0x1004ea330>
>>> sample.distance(p1,p2)
2.8284271247461903
```

### 논의

Capsule은 타입이 지정된 C 포인터와 유사하다. 내부적으로 식별 이름과 함께 일반 포인터를 보유하며 `PyCapsule_New()` 함수를 사용하여 쉽게 생성할 수 있다.

## 15.6. C에서 Python 호출

### 문제

C에서 Python callable을 안전하게 실행하고 결과를 C로 반환하고 싶다.

### 해결

C에서 Python을 호출하는 것은 대부분 간단하지만 몇 가지 까다로운 부분이 있다:

```c
#include <Python.h>

double call_func(PyObject *func, double x, double y) {
    PyObject *args;
    PyObject *kwargs;
    PyObject *result = 0;
    double retval;
    
    /* Make sure we own the GIL */
    PyGILState_STATE state = PyGILState_Ensure();
    
    /* Verify that func is a proper callable */
    if (!PyCallable_Check(func)) {
        fprintf(stderr,"call_func: expected a callable\n");
        goto fail;
    }
    
    /* Build arguments */
    args = Py_BuildValue("(dd)", x, y);
    kwargs = NULL;
    
    /* Call the function */
    result = PyObject_Call(func, args, kwargs);
    Py_DECREF(args);
    Py_XDECREF(kwargs);
    
    /* Check for Python exceptions (if any) */
    if (PyErr_Occurred()) {
        PyErr_Print();
        goto fail;
    }
    
    /* Verify the result is a float object */
    if (!PyFloat_Check(result)) {
        fprintf(stderr,"call_func: callable didn't return a float\n");
        goto fail;
    }
    
    /* Create the return value */
    retval = PyFloat_AsDouble(result);
    Py_DECREF(result);
    
    /* Restore previous GIL state and return */
    PyGILState_Release(state);
    return retval;

fail:
    Py_XDECREF(result);
    PyGILState_Release(state);
    abort();
}
```

### 논의

C에서 Python을 호출할 때 가장 중요한 점은 C가 일반적으로 담당한다는 것이다. 즉, C가 인수 생성, Python 함수 호출, 예외 확인, 타입 확인, 반환 값 추출 등의 책임이 있다.

Python의 global interpreter lock (GIL) 관리가 C에서 Python에 접근할 때의 까다로운 부분이다. `PyGILState_Ensure()`와 `PyGILState_Release()`
호출이 올바르게 수행되도록 한다.

## 15.7. C Extension에서 GIL 해제

### 문제

Python interpreter의 다른 스레드와 동시에 실행하고 싶은 C extension 코드가 있다. 이를 위해 global interpreter lock (GIL)을 해제하고 다시 획득해야 한다.

### 해결

C extension 코드에서 GIL은 다음 매크로를 삽입하여 해제하고 다시 획득할 수 있다:

```c
#include "Python.h"
...

PyObject *pyfunc(PyObject *self, PyObject *args) {
    ...
    Py_BEGIN_ALLOW_THREADS
    // Threaded C code. Must not use Python API functions
    ...
    Py_END_ALLOW_THREADS
    ...
    return result;
}
```

### 논의

GIL은 C 코드에서 Python C API 함수가 실행되지 않을 것을 보장할 수 있는 경우에만 안전하게 해제할 수 있다. 일반적인 예는 C 배열에 대한 계산을 수행하는 계산 집약적 코드(예: numpy와 같은
extension)나 블로킹 I/O 작업이 수행될 코드다.

## 15.9. Swig로 C 코드 래핑

### 문제

C extension module로 접근하고 싶은 기존 C 코드가 있다. Swig wrapper 생성기를 사용하여 이를 수행하고 싶다.

### 해결

Swig는 C 헤더 파일을 파싱하고 자동으로 extension 코드를 생성하여 작동한다. Swig "interface" 파일 예제:

```swig
// sample.i - Swig interface
%module sample
%{
#include "sample.h"
%}

/* Customizations */
%extend Point {
    /* Constructor for Point objects */
    Point(double x, double y) {
        Point *p = (Point *) malloc(sizeof(Point));
        p->x = x;
        p->y = y;
        return p;
    };
};

/* Map int *remainder as an output argument */
%include typemaps.i
%apply int *OUTPUT { int * remainder };

/* Map the argument pattern (double *a, int n) to arrays */
%typemap(in) (double *a, int n)(Py_buffer view) {
    view.obj = NULL;
    if (PyObject_GetBuffer($input, &view, 
                          PyBUF_ANY_CONTIGUOUS | PyBUF_FORMAT) == -1) {
        SWIG_fail;
    }
    if (strcmp(view.format,"d") != 0) {
        PyErr_SetString(PyExc_TypeError, "Expected an array of doubles");
        SWIG_fail;
    }
    $1 = (double *) view.buf;
    $2 = view.len / sizeof(double);
}

%typemap(freearg) (double *a, int n) {
    if (view$argnum.obj) {
        PyBuffer_Release(&view$argnum);
    }
}

/* C declarations */
extern int gcd(int, int);
extern int in_mandel(double x0, double y0, int n);
extern int divide(int a, int b, int *remainder);
extern double avg(double *a, int n);

typedef struct Point {
    double x,y;
} Point;

extern double distance(Point *p1, Point *p2);
```

Swig 실행:

```bash
bash % swig -python -py3 sample.i
```

### 논의

Swig는 extension module을 빌드하기 위한 가장 오래된 도구 중 하나다. 주요 사용자는 Python을 고급 제어 언어로 사용하여 접근하려는 대규모 기존 C 코드 베이스를 가진 사람들이다.

## 15.10. Cython으로 기존 C 코드 래핑

### 문제

기존 C 라이브러리를 래핑하는 Python extension module을 만들기 위해 Cython을 사용하고 싶다.

### 해결

Cython으로 extension module을 만드는 것은 손으로 작성한 extension을 작성하는 것과 다소 유사해 보이지만, C가 아닌 Python처럼 보이는 코드를 작성한다.

먼저 csample.pxd 파일 생성:

```cython
# csample.pxd
# Declarations of "external" C functions and structures
cdef extern from "sample.h":
    int gcd(int, int)
    bint in_mandel(double, double, int)
    int divide(int, int, int *)
    double avg(double *, int) nogil
    
    ctypedef struct Point:
        double x
        double y
    
    double distance(Point *, Point *)
```

다음으로 sample.pyx 파일 생성:

```cython
# sample.pyx
# Import the low-level C declarations
cimport csample

# Import some functionality from Python and the C stdlib
from cpython.pycapsule cimport *
from libc.stdlib cimport malloc, free

# Wrappers
def gcd(unsigned int x, unsigned int y):
    return csample.gcd(x, y)

def in_mandel(x, y, unsigned int n):
    return csample.in_mandel(x, y, n)

def divide(x, y):
    cdef int rem
    quot = csample.divide(x, y, &rem)
    return quot, rem

def avg(double[:] a):
    cdef:
        int sz
        double result
    
    sz = a.size
    with nogil:
        result = csample.avg(<double *> &a[0], sz)
    return result

# Point 객체 정리를 위한 Destructor
cdef del_Point(object obj):
    pt = <csample.Point *> PyCapsule_GetPointer(obj,"Point")
    free(<void *> pt)

# Point 객체 생성 및 capsule로 반환
def Point(double x, double y):
    cdef csample.Point *p
    p = <csample.Point *> malloc(sizeof(csample.Point))
    if p == NULL:
        raise MemoryError("No memory to make a Point")
    p.x = x
    p.y = y
    return PyCapsule_New(<void *>p,"Point",<PyCapsule_Destructor>del_Point)

def distance(p1, p2):
    pt1 = <csample.Point *> PyCapsule_GetPointer(p1,"Point")
    pt2 = <csample.Point *> PyCapsule_GetPointer(p2,"Point")
    return csample.distance(pt1,pt2)
```

빌드를 위한 setup.py:

```python
from distutils.core import setup
from distutils.extension import Extension
from Cython.Distutils import build_ext

ext_modules = [
    Extension('sample',
              ['sample.pyx'],
              libraries=['sample'],
              library_dirs=['.'])
]

setup(
    name = 'Sample extension module',
    cmdclass = {'build_ext': build_ext},
    ext_modules = ext_modules
)
```

### 논의

Cython 사용은 높은 수준에서 C를 모델로 한다. .pxd 파일은 C 정의(.h 파일과 유사)를 포함하고 .pyx 파일은 구현(.c 파일과 유사)을 포함한다.

`cimport` 문은 Cython에서 .pxd 파일에서 정의를 가져오는 데 사용된다. 이는 일반 Python module을 로드하는 일반 Python import 문과 다르다.

## 15.11. Cython으로 고성능 배열 연산 작성

### 문제

NumPy와 같은 라이브러리의 배열에서 작동하는 고성능 배열 처리 함수를 작성하고 싶다.

### 해결

간단한 1차원 double 배열의 값을 클리핑하는 Cython 함수 예제:

```cython
# sample.pyx (Cython)
cimport cython

@cython.boundscheck(False)
@cython.wraparound(False)
cpdef clip(double[:] a, double min, double max, double[:] out):
    '''
    Clip the values in a to be between min and max. Result in out
    '''
    if min > max:
        raise ValueError("min must be <= max")
    if a.shape[0] != out.shape[0]:
        raise ValueError("input and output arrays must be the same size")
    for i in range(a.shape[0]):
        if a[i] < min:
            out[i] = min
        elif a[i] > max:
            out[i] = max
        else:
            out[i] = a[i]
```

사용 예:

```python
>>> import sample
>>> import array
>>> a = array.array('d',[1,-3,4,7,2,0])
>>> sample.clip(a,1,4,a)
>>> a
array('d', [1.0, 1.0, 4.0, 4.0, 2.0, 1.0])

>>> import numpy
>>> b = numpy.random.uniform(-10,10,size=1000000)
>>> c = numpy.zeros_like(b)
>>> sample.clip(b,-5,5,c)
```

### 논의

이 레시피는 Cython의 typed memoryview를 활용하여 배열에서 작동하는 코드를 크게 단순화한다. `cpdef clip()` 선언은 clip()을 C-level과 Python-level 함수 모두로
선언한다.

typed parameter `double[:] a`와 `double[:] out`은 이러한 매개변수를 double의 1차원 배열로 선언한다. PEP 3118에 설명된 대로 memoryview 인터페이스를 제대로
구현하는 모든 배열 객체에 접근한다.

`@cython.boundscheck(False)`는 모든 배열 경계 검사를 제거하고, `@cython.wraparound(False)`는 배열 인덱스의 음수 처리를 제거한다. 이러한 decorator를 포함하면
코드가 훨씬 빠르게 실행될 수 있다.

## 15.13. NULL-Terminated String을 C 라이브러리로 전달

### 문제

NULL-terminated string을 C 라이브러리에 전달해야 하는 extension module을 작성 중이다.

### 해결

bytes에서만 작동하도록 제한하려면 `PyArg_ParseTuple()`에 "y" 변환 코드를 사용한다:

```c
static PyObject *py_print_chars(PyObject *self, PyObject *args) {
    char *s;
    if (!PyArg_ParseTuple(args, "y", &s)) {
        return NULL;
    }
    print_chars(s);
    Py_RETURN_NONE;
}
```

Unicode string을 전달하려면 "s" format 코드를 사용한다:

```c
static PyObject *py_print_chars(PyObject *self, PyObject *args) {
    char *s;
    if (!PyArg_ParseTuple(args, "s", &s)) {
        return NULL;
    }
    print_chars(s);
    Py_RETURN_NONE;
}
```

이렇게 하면 모든 문자열이 자동으로 NULL-terminated UTF-8 인코딩으로 변환된다.

### 논의

"s" format 코드를 사용하면 숨겨진 메모리 오버헤드가 있다. 이 변환을 사용하는 코드를 작성하면 UTF-8 문자열이 생성되어 원래 문자열 객체에 영구적으로 연결된다.

메모리 사용 증가가 우려된다면 `PyUnicode_AsUTF8String()` 함수를 사용하도록 C extension 코드를 다시 작성해야 한다:

```c
static PyObject *py_print_chars(PyObject *self, PyObject *args) {
    PyObject *o, *bytes;
    char *s;
    
    if (!PyArg_ParseTuple(args, "U", &o)) {
        return NULL;
    }
    bytes = PyUnicode_AsUTF8String(o);
    s = PyBytes_AsString(bytes);
    print_chars(s);
    Py_DECREF(bytes);
    Py_RETURN_NONE;
}
```

## 15.14. Unicode String을 C 라이브러리로 전달

### 문제

Python 문자열을 Unicode를 올바르게 처리할 수도 있고 아닐 수도 있는 C 라이브러리 함수에 전달해야 하는 extension module을 작성 중이다.

### 해결

byte 지향 함수의 경우, Python 문자열을 UTF-8과 같은 적절한 바이트 인코딩으로 변환해야 한다:

```c
static PyObject *py_print_chars(PyObject *self, PyObject *args) {
    char *s;
    Py_ssize_t len;
    
    if (!PyArg_ParseTuple(args, "s#", &s, &len)) {
        return NULL;
    }
    print_chars(s, len);
    Py_RETURN_NONE;
}
```

기계 네이티브 `wchar_t` 타입으로 작동하는 라이브러리 함수의 경우:

```c
static PyObject *py_print_wchars(PyObject *self, PyObject *args) {
    wchar_t *s;
    Py_ssize_t len;
    
    if (!PyArg_ParseTuple(args, "u#", &s, &len)) {
        return NULL;
    }
    print_wchars(s,len);
    Py_RETURN_NONE;
}
```

### 논의

Python 3는 표준 타입인 `char *`나 `wchar_t *`를 사용하는 C 라이브러리에 직접 매핑하기가 완전히 간단하지 않은 적응형 문자열 표현을 사용한다. 따라서 C에 문자열 데이터를 제공하려면 거의 항상
변환이 필요하다.

`s#`와 `u#` format 코드는 이러한 변환을 안전하게 수행한다. 한 가지 잠재적인 단점은 이러한 변환으로 인해 원래 문자열 객체의 크기가 영구적으로 증가한다는 것이다.

## 15.21. Segmentation Fault 진단

### 문제

interpreter가 segmentation fault, bus error, access violation 또는 기타 치명적인 오류로 심하게 충돌한다. 실패 시점에서 프로그램이 어디서 실행되고 있었는지 보여주는
Python traceback을 얻고 싶다.

### 해결

`faulthandler` 모듈을 사용하여 이 문제를 해결할 수 있다:

```python
import faulthandler
faulthandler.enable()
```

또는 `-Xfaulthandler` 옵션으로 Python을 실행한다:

```bash
bash % python3 -Xfaulthandler program.py
```

`faulthandler`가 활성화되면 C extension의 치명적인 오류가 Python traceback이 출력된다:

```
Fatal Python error: Segmentation fault

Current thread 0x00007fff71106cc0:
  File "example.py", line 6 in foo
  File "example.py", line 10 in bar
  File "example.py", line 14 in spam
  File "example.py", line 19 in <module>
Segmentation fault
```

### 논의

`faulthandler`는 실패 시점에 실행 중인 Python 코드의 stack traceback을 표시한다. 최소한 이것은 호출된 최상위 extension 함수를 보여줄 것이다.

`faulthandler`는 C의 실패에 대해서는 아무것도 알려주지 않는다. 이를 위해서는 gdb와 같은 전통적인 C debugger를 사용해야 한다.