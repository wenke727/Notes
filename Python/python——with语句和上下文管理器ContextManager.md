<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. 前言</a></li>
<li><a href="#sec-2">2. 上下文管理器（ContextManager）</a></li>
<li><a href="#sec-3">3. With语法</a></li>
<li><a href="#sec-4">4. 实例：自己实现打开文件操作</a></li>
<li><a href="#sec-5">5. 内置库contextlib的使用</a></li>
</ul>
</div>
</div>

本文转载自[简书](https://zhuanlan.zhihu.com/p/24709718)

# 前言<a id="sec-1" name="sec-1"></a>

Python中的 `with` 语句和上下文管理器，是从2.5版本开始加入到语法中的。

# 上下文管理器（ContextManager）<a id="sec-2" name="sec-2"></a>

上下文管理器是指在一段代码执行之前执行一段代码，用于一些预处理工作；执行之后再执行一段代码，用于一些清理工作。比如打开文件进行读写，读写完之后需要将文件关闭。又比如在数据库操作中，操作之前需要连接数据库，操作之后需要关闭数据库。在上下文管理协议中，有两个方法\_<sub>enter</sub>\_<sub>和</sub>\_<sub>exit</sub>\_<sub>，分别实现上述两个功能。</sub>

# With语法<a id="sec-3" name="sec-3"></a>

讲到上下文管理器，就不得不说到python的with语法。基本语法格式为：

    with EXPR as VAR:
        BLOCK

这里就是一个标准的上下文管理器的使用逻辑，稍微解释一下其中的运行逻辑：

1.  执行 `EXPR` 语句，获取上下文管理器（Context Manager）
2.  调用上下文管理器中的 `__enter__` 方法，该方法执行一些预处理工作。
3.  这里的 `as VAR` 可以省略，如果不省略，则将\_<sub>enter</sub>\_<sub>方法的返回值赋值给VAR。</sub>
4.  执行代码块 `BLOCK` ，这里的VAR可以当做普通变量使用。
5.  最后调用上下文管理器中的的 `__exit__` 方法。
6.  `__exit__` 方法有三个参数： `exc_type`, `exc_val`, `exc_tb` 。如果代码块 `BLOCK` 发生异常并退出，那么分别对应异常的 `type` 、 `value` 和 `traceback` 。否则三个参数全为 `None` 。
7.  `__exit__` 方法的返回值可以为 `True` 或者 `False` 。如果为 `True` ，那么表示异常被忽视，相当于进行了 `try-except` 操作；如果为 `False` ，则该异常会被重新 `raise` 。

# 实例：自己实现打开文件操作<a id="sec-4" name="sec-4"></a>

    # 自定义打开文件操作 
    class MyOpen(object): 
    
        def __init__(self, file_name): 
            """初始化方法""" 
            self.file_name = file_name 
            self.file_handler = None 
            return 
    
        def __enter__(self): 
            """enter方法，返回file_handler""" 
            print("enter:", self.file_name) 
            self.file_handler = open(self.file_name, "r") 
            return self.file_handler 
    
        def __exit__(self, exc_type, exc_val, exc_tb): 
            """exit方法，关闭文件并返回True""" 
            print("exit:", exc_type, exc_val, exc_tb) 
            if self.file_handler: 
                self.file_handler.close() 
            return True 
    
    # 使用实例 
    with MyOpen("python_base.py") as file_in: 
        for line in file_in: 
            print(line) 
            raise ZeroDivisionError 
    # 代码块中主动引发一个除零异常，但整个程序不会引发异常

代码很简单，也很容易理解，这里不做过多解释。

# 内置库contextlib的使用<a id="sec-5" name="sec-5"></a>

Python提供内置的contextlib库，使得上线文管理器更加容易使用。其中包含如下功能：

1.  装饰器contextmanager。该装饰器将一个函数中yield语句之前的代码当做\_<sub>enter</sub>\_<sub>方法执行，yield语句之后的代码当做</sub>\_<sub>exit</sub>\_<sub>方法执行。同时yield返回值赋值给as后的变量。</sub>
    
    ```
    @contextlib.contextmanager 
    def open_func(file_name): 
        # __enter__方法 
        print('open file:', file_name, 'in __enter__') 
        file_handler = open(file_name, 'r') 
    
        yield file_handler 
    
        # __exit__方法 print('close file:', file_name, 'in __exit__') 
        file_handler.close() 
        return 
    
    with open_func('python_base.py') as file_in: 
        for line in file_in: 
            print(line)
    ```

1.  closing类。该类会自动调用传入对象的 `close` 方法。使用实例如下：

    ```
    class MyOpen2(object): 
    
        def __init__(self, file_name): 
            """初始化方法""" 
            self.file_handler = open(file_name, "r") 
            return 
    
        def close(self): 
            """关闭文件，会被自动调用""" 
            print("call close in MyOpen2") 
            if self.file_handler: 
                self.file_handler.close() 
            return 
    
    with contextlib.closing(MyOpen2("python_base.py")) as file_in: 
        pass
    ```

    这里会自动调用MyOpen2的 `close` 方法。我们查看 `contextlib.closing` 方法，内部实现为：

    ```
    class closing(object): 
        """Context to automatically close something at the end of a block.""" 
        def __init__(self, thing): 
            self.thing = thing 
    
        def __enter__(self): 
            return self.thing 
    
        def __exit__(self, *exc_info): 
            self.thing.close()
    ```

    closing类的 `__exit__` 方法自动调用传入的thing的 `close` 方法。

1.  nested类。该类在Python2.7之后就删除了。原本该类的作用是减少嵌套，但是Python2.7之后允许如下的写法：

    ```
    with open("aaa", "r") as file_in, open("bbb", "w") as file_out:
        pass
    ```

参考：<https://www.python.org/dev/peps/pep-0343/>
