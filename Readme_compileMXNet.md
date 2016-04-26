[iOS] mxnet 的 iOS 版本编译
 skylook 2015年12月10日 1 Comment
mxnet2

0、编译环境：
Mac OSX 10.11 Capitan
Xcode 7.1
mxnet 0.5.0

0、下载 mxnet：
参考 sqlite 的方式，mxnet 也提供了一个 Makefile 文件用来生成单文件的版本。这样只需要一个文件加上 BLAS 依赖库就可以运行 predict 预测部分。这一文件移植到任何平台上都会比较容易。
下载 mxnet 版本：

1、生成 mxnet 单文件版：
修改 amalgamation 目录下的 Makefile 文件：

1）修改 OPENBLAS_ROOT 路径1：
找到：


export OPENBLAS_ROOT=`pwd`/OpenBLAS
1
export OPENBLAS_ROOT=`pwd`/OpenBLAS
修改成：


export OPENBLAS_ROOT=/usr/local/opt/openblas
1
export OPENBLAS_ROOT=/usr/local/opt/openblas
注意这里面 /usr/local/opt/openblas/include 是我的 openblas 库安装位置，如果你的不是安装在这里请修改为你自己的路径。如果你没有安装 openblas 请参照 这篇文章 里面的进行安装。

2）修改 OPENBLAS_ROOT 路径2：
找到：


ifneq ($(MIN), 1)
	CFLAGS+= -I${OPENBLAS_ROOT}
	LDFLAGS+=-L${OPENBLAS_ROOT} -lopenblas
endif
1
2
3
4
ifneq ($(MIN), 1)
	CFLAGS+= -I${OPENBLAS_ROOT}
	LDFLAGS+=-L${OPENBLAS_ROOT} -lopenblas
endif
修改成：


ifneq ($(MIN), 1)
	CFLAGS+= -I${OPENBLAS_ROOT}/include
	LDFLAGS+=-L${OPENBLAS_ROOT}/lib -lopenblas
endif
1
2
3
4
ifneq ($(MIN), 1)
	CFLAGS+= -I${OPENBLAS_ROOT}/include
	LDFLAGS+=-L${OPENBLAS_ROOT}/lib -lopenblas
endif
3）去除 lrt 库链接：
找到：


ifneq ($(ANDROID), 1)
        LDFLAGS+= -lrt
        android:
else
1
2
3
4
ifneq ($(ANDROID), 1)
        LDFLAGS+= -lrt
        android:
else
修改成：


ifneq ($(ANDROID), 1)
        android:
else 
1
2
3
ifneq ($(ANDROID), 1)
        android:
else 
4）生成单文件和库：
在 amalgamation 目录下运行 make：


make
1
make
如果生成过程顺利会看到如下输出：
ar rcs libmxnet_predict.a mxnet_predict-all.o
c++ -std=c++11 -Wno-unknown-pragmas -Wall -I/usr/local/opt/openblas/include -shared -o pwd/../lib/libmxnet_predict.so mxnet_predict-all.o -L/usr/local/opt/openblas/lib -lopenblas
ls -alh pwd/../lib/libmxnet_predict.so
-rwxr-xr-x 1 valiantliu staff 4.1M Dec 8 17:47 /Users/valiantliu/Documents/Develop/mxnet/mxnet/amalgamation/../lib/libmxnet_predict.so

这时应该目录下会生成如下几个文件：
D386FED1-BB9B-4C22-A6BE-55C7D18DE19E

这里面编译的自然都是 Mac 上面的库，如果你只是需要一个 OSX 版本，那这里的已经可以用了。不过编译 iOS 版本我们还需要继续做一些工作。

附注：如果不想手工修改，你可以直接下载我修改好的这个 Makefile 文件替换到 amalgamation 目录下面：
Makefile

2、编译 iOS 版本：
1）添加源码：
我们使用 Xcode 新建一个工程命名为 mxnet_predict，然后将刚才生成的 mxnet_predict-all.cc 文件和 include/mxnet/c_predict_api.h 文件放入工程里面。

2）添加 Accelerate 框架：
mxnet 使用 openblas 库，直接把这个库编进来也可以，但我们更希望其使用 Apple 专门开发的加速框架 vecLib。这个框架在 Accelerate.framework 里面。

我们在 Target -> Build Phases -> Link Binary With Libraries 下面添加 Accelerate.framework 框架，如图所示：
FE60438B-7FF2-48DB-838D-5CCD4F573B94

3）修改 mxnet_predict-all.cc 源码：
找到：


#include <cblas.h>
1
#include <cblas.h>
修改成：


#include <Accelerate/Accelerate.h>
1
#include <Accelerate/Accelerate.h>
找到：


#if !defined(__ANDROID__) && (!defined(MSHADOW_USE_SSE) || MSHADOW_USE_SSE == 1)
#include <emmintrin.h>
#endif
1
2
3
#if !defined(__ANDROID__) && (!defined(MSHADOW_USE_SSE) || MSHADOW_USE_SSE == 1)
#include <emmintrin.h>
#endif
修改成：


#if !defined(__ANDROID__) && (!defined(MSHADOW_USE_SSE) || MSHADOW_USE_SSE == 1)
//#include <emmintrin.h>
#endif
1
2
3
#if !defined(__ANDROID__) && (!defined(MSHADOW_USE_SSE) || MSHADOW_USE_SSE == 1)
//#include <emmintrin.h>
#endif
找到：


#include <deque>
#include <emmintrin.h>
#include <functional>
1
2
3
#include <deque>
#include <emmintrin.h>
#include <functional>
修改成：


#include <deque>
//#include <emmintrin.h>
#include <functional>
1
2
3
#include <deque>
//#include <emmintrin.h>
#include <functional>
找到：


#ifndef MX_TREAD_LOCAL
#message("Warning: Threadlocal is not enabled");
#endif
1
2
3
#ifndef MX_TREAD_LOCAL
#message("Warning: Threadlocal is not enabled");
#endif
修改成：


#define MX_TREAD_LOCAL __declspec(thread)
    
#ifndef MX_TREAD_LOCAL
#message("Warning: Threadlocal is not enabled");
#endif
1
2
3
4
5
#define MX_TREAD_LOCAL __declspec(thread)
    
#ifndef MX_TREAD_LOCAL
#message("Warning: Threadlocal is not enabled");
#endif
如果您不想做上面的修改，也可以直接使用我修改好的文件：
mxnet_predict.zip

4）直接编译生成 libmxnet_predict.a 库文件即可。

下方是我编译好的 iOS 版本 mxnet 库，编译选项为 libc++ 和 enable bitcode：
libmxnet_0.5.0_iOS

常见错误：
1、错误：’cblas.h’ file not found
请按照步骤 1 中 2）小节的提示配置 openblas 目录或者安装 openblas。

2、错误：ld: library not found for -lrt
请按照步骤 1 中 3）小节的提示删除 lrt 链接库。

3、错误：Thread-local storage is not supported for the current target
OSX 和 iOS 因为某种原因禁止了 TLS 的支持，因此需要参照文献 [2] 进行修改才能使用。

参考文献：
[1] https://github.com/dmlc/mxnet/issues/546
[2] http://www.darlinghq.org/for-developers/thread-local-storage-on-os-x
