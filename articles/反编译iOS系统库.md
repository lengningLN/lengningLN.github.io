# dyld-extractior-lib
了解苹果动态库的加载过程，拆分苹果共享动态库，并逆向苹果动态库，查看关心的方法的实现。

## 介绍
- 环境介绍: 越狱设备是iPhone 5S，系统为iOS 9。系统分析工具使用的是iFunBox,反编译工具是Hopper Disassembler v4，代码查看工具是Xcode 9.4 

- 我们知道苹果为了方便程序员开发，提供了很多封装好的动态库，比如UIKit、AVFoundation、ImageIO等，从iOS3.1开始，为了提高性能，绝大部分的系统动态库都会被打包到一个缓存文件中。这样所有的应用程序都共享一份动态库的缓存。
- 查看共享动态库的路径/System/Library/Caches/com.apple.dyld/
![](img/dyldlib.png)

## dyld
- 在Mac/iOS中，使用dyld程序来加载动态库，路径如下
![](img/dyld_command.png)
- 在Xcode中加入的动态库（通过Target -> Build Phases -> Link Binary With Libraries）或者在程序运行期间我们手动加入的动态库(通过[NSBundle bundleWithPath:@"/System/Library/Frameworks/UIKit.framework"])，操作系统会从中拿到需要加载的库的名称，然后去动态库共享缓存中查找与之对应的动态库。

- 由于系统所有的动态库文件都被放到一个文件中，有600M左右，我们可能只关心指定库中的某一文件的某一方法是如何现实的。比如：UIKit库中的UIView的init方法是如何实现的。所以应该想办法把不同的动态库抽离出来，方便以后查找。

## 从动态库共享缓存中抽离独立的动态库
1. 通过Xcode打开dyld的源码
- [dyld的源码地址](https://opensource.apple.com/tarballs/dyld/)中已经下载下来最新的源码，放在本工程中的[这个位置](/dyld-519.2.2)
2. 找到launch-cache/dsc_extractor.cpp文件，拷贝最下面的main函数的代码，如下
```
#include <stdio.h>
#include <stddef.h>
#include <dlfcn.h>
typedef int (*extractor_proc)(const char* shared_cache_file_path, const char* extraction_root_path,
													void (^progress)(unsigned current, unsigned total));
int main(int argc, const char* argv[])
{
	if ( argc != 3 ) {
		fprintf(stderr, "usage: dsc_extractor <path-to-cache-file> <path-to-device-dir>\n");
		return 1;
	}
	
	//void* handle = dlopen("/Volumes/my/src/dyld/build/Debug/dsc_extractor.bundle", RTLD_LAZY);
	void* handle = dlopen("/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle", RTLD_LAZY);
	if ( handle == NULL ) {
		fprintf(stderr, "dsc_extractor.bundle could not be loaded\n");
		return 1;
	}
	
	extractor_proc proc = (extractor_proc)dlsym(handle, "dyld_shared_cache_extract_dylibs_progress");
	if ( proc == NULL ) {
		fprintf(stderr, "dsc_extractor.bundle did not have dyld_shared_cache_extract_dylibs_progress symbol\n");
		return 1;
	}
	
	int result = (*proc)(argv[1], argv[2], ^(unsigned c, unsigned total) { printf("%d/%d\n", c, total); } );
	fprintf(stderr, "dyld_shared_cache_extract_dylibs_progress() => %d\n", result);
	return 0;
}
```

3. 在桌面上新建一个文件夹dinamicLibs，在文件夹中新建一个dsc_extractor.cpp文件放，把上述代码拷贝进去。
4. 使用如下命令编译dsc_extractor.cpp
```
- clang++ -o dsc_extractor dsc_extractor1.cpp 
```
- 在当前路径下会出现一个可执行程序</br>
![](img/clangfile.png)
- 把想要抽离的共享动态库文件（比如dyld_shared_cache_armv7s）拷贝到当前路径下，然后就可以执行如下命令。
```
./dsc_extractor dyld_shared_cache_armv7s armv7s
```
这个命令的第一个参数dyld_shared_cache_armv7s表示动态库共享文件的路径，第二个参数armv7s表示存放抽取结果的文件夹
- 执行结束后，效果如下
- ![](img/extractor.png)
- 查看armv7s文件夹，效果如下,到此就已经抽离了所有单独的系统动态库
- ![](img/singleLibs.png)

### 查看系统UIKit库中UIView的init方法的实现
1. 打开反编译工具是Hopper Disassembler v4
2. 把UIKit库拖到工具中
3. 找到指定方法，如下
![](img/demo.png)
- 由此看出，在调用UIView的init方法是，系统调用了initWithFrame:方法

## 总结
以上就是抽离系统动态库并反编译系统动态库的过程。从反编译的代码可以看出，虽然和真实的代码有很大出入，但是依然能够窥视苹果实现的关键代码，从而获得苹果实现的思路。对我们更好的使用苹果api提供理论支持。

### more about  【更多】
1. 如果有什么问题，请在[issues](https://github.com/lengningLN/LNSwipeCellDemo/issues)区域提问，我会抽时间改进。
2. [我的博客](https://www.jianshu.com/u/dbd52f0e4f1c)
### 打赏
![](http://m.qpic.cn/psb?/V11R4JcH0fAdbu/h4vWrizoOlby*zntVMiu.1F9CMMMx2T9BOWUjSEnCE8!/b/dDUBAAAAAAAA&bo=nALQAgAAAAADB24!&rf=viewer_4)







