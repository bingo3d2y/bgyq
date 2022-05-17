# 【LGTM】Python3 cpython 优化 支持解释器并行

Original 谢俊逸 [字节跳动终端技术](javascript:void(0);) *2022-02-22 18:07*

## 背景

在业务场景中，我们通过cpython执行算法包，由于cpython的实现，在一个进程内，无法利用CPU的多个核心去同时执行算法包。对此，我们决定优化cpython，**目标是让cpython高完成度的支持并行，大幅度提高单个进程内Python算法包的执行效率。**

在2020年，我们完成了对cpython的并行执行改造，是目前业界首个cpython3的高完成度同时兼容Python C API的并行实现。

- 性能

- - 单线程性能劣化7.7%
  - 多线程基本无锁抢占，多开一个线程减少44%的执行时间。
  - 并行执行对总执行时间有大幅度的优化

- 通过了cpython的单元测试
- 在线上已经全量使用

cpython痛, GIL

cpython是python官方的解释器实现。在cpython中，GIL，用于保护对Python对象的访问，从而防止多个线程同时执行Python字节码。GIL防止出现竞争情况并确保线程安全。因为GIL的存在，cpython 是无法真正的并行执行python字节码的. GIL虽然限制了python的并行，但是因为cpython的代码没有考虑到并行执行的场景，充满着各种各样的共享变量，改动复杂度太高，官方一直没有移除GIL。

## 挑战

在Python开源的20年里，Python 因为GIL（全局锁）不能并行。目前主流实现Python并行的两种技术路线，但是一直没有高完成度的解决方案（高性能，兼容所有开源feature, API稳定）。主要是因为：

**1. 直接去除GIL 解释器需要加许多细粒度的锁，影响单线程的执行性能，慢两倍。**

> Back in the days of Python 1.5, Greg Stein actually implemented a comprehensive patch set (the “free threading” patches) that removed the GIL and replaced it with fine-grained locking. Unfortunately, even on Windows (where locks are very efficient) this ran ordinary Python code about twice as slow as the interpreter using the GIL. On Linux the performance loss was even worse because pthread locks aren’t as efficient.

**2. 解释器状态隔离 解释器内部的实现充满了各种全局状态，改造繁琐，工作量大。**

> It has been suggested that the GIL should be a per-interpreter-state lock rather than truly global; interpreters then wouldn’t be able to share objects. Unfortunately, this isn’t likely to happen either. It would be a tremendous amount of work, because many object implementations currently have global state. For example, small integers and short strings are cached; these caches would have to be moved to the interpreter state. Other object types have their own free list; these free lists would have to be moved to the interpreter state. And so on.

这个思路开源有一个项目在做 multi-core-python[1],但是目前已经搁置了。目前只能运行非常简单的算术运算的demo。对Type和许多模块的并行执行问题并没有处理，无法在实际场景中使用。

## 新架构-多解释器架构

为了实现最佳的执行性能，我们参考multi-core-python[1]，在cpython3.10实现了一个高完成度的并行实现。

- 从全局解释器状态 转换为 每个解释器结构持有自己的运行状态（独立的GIL，各种执行状态）。
- 支持并行，解释器状态隔离，并行执行性能不受解释器个数的影响（解释器间基本没有锁相互抢占）
- 通过线程的Thread Specific Data获取Python解释器状态。

在这套新架构下，Python的解释器相互隔离，不共享GIL，可以并行执行。充分利用现代CPU的多核性能。大大减少了业务算法代码的执行时间。

### 共享变量的隔离

解释器执行中使用了很多共享的变量，他们普遍以全局变量的形式存在.多个解释器运行时，会同时对这些共享变量进行读写操作，线程不安全。

cpython内部的主要共享变量：3.10待处理的共享变量[2]。大概有1000个...需要处理，工作量非常之大。

- free lists

- - MemoryError
  - asynchronous generator
  - context
  - dict
  - float
  - frame
  - list
  - slice

- singletons

- - small integer ([-5; 256] range)
  - empty bytes string singleton
  - empty Unicode string singleton
  - empty tuple singleton
  - single byte character (b’\x00’ to b’\xFF’)
  - single Unicode character (U+0000-U+00FF range)

- cache

- - slide cache
  - method cache
  - bigint cache
  - ...

- interned strings

- `PyUnicode_FromId` static strings

- ....

**如何让每个解释器独有这些变量呢？**

cpython是c语言实现的，在c中，我们一般会通过 参数中传递 interpreter_state 结构体指针来保存属于一个解释器的成员变量。这种改法也是性能上最好的改法。但是如果这样改，那么所有使用interpreter_state的函数都需要修改函数签名。从工程角度上是几乎无法实现的。

只能换种方法，我们可以将interpreter_state存放到thread specific data中。interpreter执行时，通过thread specific key获取到 interpreter_state.这样就可以通过thread specific的API，获取到执行状态，并且不用修改函数的签名。

```c
static inline PyInterpreterState* _PyInterpreterState_GET(void) {
    PyThreadState *tstate = _PyThreadState_GET();
#ifdef Py_DEBUG
    _Py_EnsureTstateNotNULL(tstate);
#endif
    return tstate->interp;
}
```

**共享变量变为解释器单独持有** 我们将所有的共享变量存放到 interpreter_state里。

```c
    /* Small integers are preallocated in this array so that they
       can be shared.
       The integers that are preallocated are those in the range
       -_PY_NSMALLNEGINTS (inclusive) to _PY_NSMALLPOSINTS (not inclusive).
    */
    PyLongObject* small_ints[_PY_NSMALLNEGINTS + _PY_NSMALLPOSINTS];
    struct _Py_bytes_state bytes;
    struct _Py_unicode_state unicode;
    struct _Py_float_state float_state;
    /* Using a cache is very effective since typically only a single slice is
       created and then deleted again. */
    PySliceObject *slice_cache;

    struct _Py_tuple_state tuple;
    struct _Py_list_state list;
    struct _Py_dict_state dict_state;
    struct _Py_frame_state frame;
    struct _Py_async_gen_state async_gen;
    struct _Py_context_state context;
    struct _Py_exc_state exc_state;

    struct ast_state ast;
    struct type_cache type_cache;
#ifndef PY_NO_SHORT_FLOAT_REPR
    struct _PyDtoa_Bigint *dtoa_freelist[_PyDtoa_Kmax + 1];
#endif
```

通过 _PyInterpreterState_GET 快速访问。例如

```c
/* Get Bigint freelist from interpreter  */
static Bigint **
get_freelist(void) {
    PyInterpreterState *interp = _PyInterpreterState_GET();
    return interp->dtoa_freelist;
} 
```

注意，将全局变量改为thread specific data是有性能影响的，不过只要控制该API调用的次数，性能影响还是可以接受的。我们在cpython3.10已有改动的的基础上，解决了各种各样的共享变量问题，3.10待处理的共享变量[2]。

### Type变量共享的处理，API兼容性及解决方案

目前cpython3.x 暴露了PyType_xxx 类型变量在API中。这些全局类型变量被第三方扩展代码以&PyType_xxx的方式引用。如果将Type隔离到子解释器中，势必造成不兼容的问题。这也是官方改动停滞的原因，这个问题无法以合理改动的方式出现在python3中。只能等到python4修改API之后改掉。

我们通过另外一种方式快速地改掉了这个问题。

Type是共享变量会导致以下的问题

\1. Type Object的 Ref count被频繁修改，线程不安全

\2. Type Object 成员变量被修改，线程不安全。

改法：

\1. immortal type object.

\2. 使用频率低的不安全处加锁。

\3. 高频使用的场景，使用的成员变量设置为immortal object.

- 针对python的描述符机制，对实际使用时，类型的property,函数，classmethod,staticmethod,doc生成的描述符也设置成immortal object.

这样会导致Type和成员变量会内存泄漏。不过由于cpython有module的缓存机制，不清理缓存时，便没有问题。

### pymalloc内存池共享处理

我们使用了mimalloc[3] 替代pymalloc内存池，在优化1%-2%性能的同时，也不需要额外处理pymalloc。

## subinterperter 能力补全

官方master最新代码 subinterpreter 模块只提供了interp_run_string可以执行code_string. 出于体积和安全方面的考虑，我们已经删除了python动态执行code_string的功能。我们给subinterpreter模块添加了两个额外的能力

\1. interp_call_file 调用执行python pyc文件

\2. interp_call_function 执行任意函数

## subinterpreter 执行模型

python中，我们执行代码默认运行的是main interpreter, 我们也可以创建的sub interpreter执行代码，

```c
interp = _xxsubinterpreters.create()
result = _xxsubinterpreters.interp_call_function(*args, **kwargs)
```

这里值得注意的是，我们是在 main interpreter 创建 sub interpreter，随后在sub interpreter 执行，最后把结果返回到main interpreter. 这里看似简单，但是做了很多事情。

\1. main interpreter 将参数传递到 sub interpreter

\2. 线程切换到 sub interpreter的 interpreter_state。获取并转换参数

\3. sub interpreter 解释执行代码

\4. 获取返回值，切换到main interpreter

\5. 转换返回值

\6. 异常处理

这里有两个复杂的地方：

\1. interpreter state 状态的切换

\2. interpreter 数据的传递

### interpreter state 状态的切换

```c
interp = _xxsubinterpreters.create()
result = _xxsubinterpreters.interp_call_function(*args, **kwargs)
```

我们可以分解为

```c
# Running In thread 11:
# main interpreter:
# 现在 thread specific 设置的 interpreter state 是 main interpreter的
do some things ... 
create subinterpreter ...
interp_call_function ...
# thread specific 设置 interpreter state 为 sub interpreter state
# sub interpreter: 
do some thins ...
call function ...
get result ...
# 现在 thread specific 设置 interpreter state 为 main interpreter state
get return result ..
```

interpreter 数据的传递

因为我们解释器的执行状态是隔离的，在main interpreter 中创建的 Python Object是无法在 sub interpreter 使用的. 我们需要:

\1. 获取 main interpreter 的 PyObject 关键数据

\2. 存放在 一块内存中

\3. 在sub interpreter 中根据该数据重新创建 PyObject

interpreter 状态的切换 & 数据的传递 的实现可以参考以下示例 ...

```c
static PyObject *
_call_function_in_interpreter(PyObject *self, PyInterpreterState *interp, _sharedns *args_shared, _sharedns *kwargs_shared)
{
    PyObject *result = NULL;
    PyObject *exctype = NULL;
    PyObject *excval = NULL;
    PyObject *tb = NULL;
    _sharedns *result_shread = _sharedns_new(1);

#ifdef EXPERIMENTAL_ISOLATED_SUBINTERPRETERS
    // Switch to interpreter.
    PyThreadState *new_tstate = PyInterpreterState_ThreadHead(interp);
    PyThreadState *save1 = PyEval_SaveThread();

    (void)PyThreadState_Swap(new_tstate);
#else
    // Switch to interpreter.
    PyThreadState *save_tstate = NULL;
    if (interp != PyInterpreterState_Get()) {
        // XXX Using the  head  thread isn't strictly correct.
        PyThreadState *tstate = PyInterpreterState_ThreadHead(interp);
        // XXX Possible GILState issues?
        save_tstate = PyThreadState_Swap(tstate);
    }
#endif
    
    PyObject *module_name = _PyCrossInterpreterData_NewObject(&args_shared->items[0].data);
    PyObject *function_name = _PyCrossInterpreterData_NewObject(&args_shared->items[1].data);

    ...
    
    PyObject *module = PyImport_ImportModule(PyUnicode_AsUTF8(module_name));
    PyObject *function = PyObject_GetAttr(module, function_name);
    
    result = PyObject_Call(function, args, kwargs);

    ...

#ifdef EXPERIMENTAL_ISOLATED_SUBINTERPRETERS
    // Switch back.
    PyEval_RestoreThread(save1);
#else
    // Switch back.
    if (save_tstate != NULL) {
        PyThreadState_Swap(save_tstate);
    }
#endif
    
    if (result) {
        result = _PyCrossInterpreterData_NewObject(&result_shread->items[0].data);
        _sharedns_free(result_shread);
    }
    
    return result;
}
```

## 实现子解释器池

我们已经实现了内部的隔离执行环境，但是这是API比较低级，需要封装一些高度抽象的API，提高子解释器并行的易用能力。

```c
interp = _xxsubinterpreters.create()
result = _xxsubinterpreters.interp_call_function(*args, **kwargs)
```

这里我们参考了，python concurrent库提供的 thread pool, process pool, futures 的实现，自己实现了 subinterpreter pool. 通过concurrent.futures 模块提供异步执行回调高层接口。

```
executer = concurrent.futures.SubInterpreterPoolExecutor(max_workers)
future = executer.submit(_xxsubinterpreters.call_function, module_name, func_name, *args, **kwargs)
future.context = context
future.add_done_callback(executeDoneCallBack)
```

我们内部是这样实现的: 继承 concurrent 提供的 Executor 基类

```
class SubInterpreterPoolExecutor(_base.Executor):
```

SubInterpreterPool 初始化时创建线程，并且每个线程创建一个 sub interpreter

```
interp = _xxsubinterpreters.create()
t = threading.Thread(name=thread_name, target=_worker,
                     args=(interp, 
                           weakref.ref(self, weakref_cb),
                           self._work_queue,
                           self._initializer,
                           self._initargs))
```

线程 worker 接收参数，并使用 interp 执行

```
result = self.fn(self.interp ,*self.args, **self.kwargs)
```

## 实现外部调度模块

针对sub interpreter的改动较大，存在两个隐患

\1. 代码可能存在兼容性问题，第三方C/C++ Extension 实现存在全局状态变量，非线程安全。

\2. python存在着极少的一些模块.sub interpreter无法使用。例如process

我们希望能统一对外的接口，让使用者不需要关注这些细节，我们自动的切换调用方式。自动选择在主解释器使用（兼容性好,稳定）还是子解释器（支持并行，性能佳）

我们提供了C和python的实现，方便业务方在各种场景使用，这里介绍下python实现的简化版代码。

在bddispatch.py 中，抽象了调用方式，提供统一的执行接口，统一处理异常和返回结果。bddispatch.py

```
def executeFunc(module_name, func_name, context=None, use_main_interp=True, *args, **kwargs):
    print( submit call  , module_name,  . , func_name)
    if use_main_interp == True:
        result = None
        exception = None
        try:
            m = __import__(module_name)
            f = getattr(m, func_name)
            r = f(*args, **kwargs)
            result = r
        except:
            exception = traceback.format_exc()
        singletonExecutorCallback(result, exception, context)

    else:
        future = singletonExecutor.submit(_xxsubinterpreters.call_function, module_name, func_name, *args, **kwargs)
        future.context = context
        future.add_done_callback(executeDoneCallBack)


def executeDoneCallBack(future):
    r = future.result()
    e = future.exception()
    singletonExecutorCallback(r, e, future.context)
```

## 直接绑定到子解释器执行

对于性能要求高的场景，通过上述的方式，由主解释器调用子解释器去执行任务会增加性能损耗。这里我们提供了一些CAPI, 让直接内嵌cpython的使用方通过C API直接绑定某个解释器执行。

```
class GILGuard {
public:
    GILGuard() {
        inter_ = BDPythonVMDispatchGetInterperter();
        if (inter_ == PyInterpreterState_Main()) {
            printf( Ensure on main interpreter: %p\n , inter_);
        } else {
            printf( Ensure on sub interpreter: %p\n , inter_);
        }
        gil_ = PyGILState_EnsureWithInterpreterState(inter_);
        
    }
    
    ~GILGuard() {
        if (inter_ == PyInterpreterState_Main()) {
            printf( Release on main interpreter: %p\n , inter_);
        } else {
            printf( Release on sub interpreter: %p\n , inter_);
        }
        PyGILState_Release(gil_);
    }
    
private:
    PyInterpreterState *inter_;
    PyGILState_STATE gil_;
};

// 这样就可以自动绑定到一个解释器直接执行
- (void)testNumpy {
    GILGuard gil_guard;
    BDPythonVMRun(....);
}
```

### **参考文献**

\1. multi-core-python[1]：

https://github.com/ericsnowcurrently/multi-core-python

\2. 3.10待处理的共享变量[2]：

https://github.com/python/cpython/blob/3.10/Tools/c-analyzer/TODO

\3. mimalloc[3]：

https://github.com/microsoft/mimalloc

\# 关于字节终端技术团队

字节跳动终端技术团队(Client Infrastructure)是大前端基础技术的全球化研发团队（分别在北京、上海、杭州、深圳、广州、新加坡和美国山景城设有研发团队），负责整个字节跳动的大前端基础设施建设，提升公司全产品线的性能、稳定性和工程效率；支持的产品包括但不限于抖音、今日头条、西瓜视频、飞书、番茄小说等，在移动端、Web、Desktop等各终端都有深入研究。



