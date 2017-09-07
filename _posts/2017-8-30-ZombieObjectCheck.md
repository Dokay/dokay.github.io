---
layout: post
comments: true
title:  "Release 模式下的Zombie检查工具"
date:   2017-8-30 21:03:03 +0800
categories: iOS
---

不想看原理的可以直接去使用，传送门：[DJZombieCheck](https://github.com/Dokay/DJZombieCheck)

一般我们对Zombie对象检查使用Xcode自带的功能：

![](/images/posts/zombie_object_check/xcode_zombie_object.png)

但是该功能有个缺陷，即只能在Debug模式下使用，大部分Zombie是偶然发生的，并没有必现的路径。如果QA手中的版本（Release模式打包）中有类似的Zombie检查功能将会大大提高我们找到Zombie对象的效率。

### 尝试使用系统自带的
在源码(这里使用[objc4-706](https://opensource.apple.com/tarballs/objc4/objc4-706.tar.gz))中搜索zombie，很容易搜索到下面代码：
```
void environ_init(void) 
{
 	...
	
    for (char **p = *_NSGetEnviron(); *p != nil; p++) {
        if (0 == strncmp(*p, "Malloc", 6)  ||  0 == strncmp(*p, "DYLD", 4)  ||  
            0 == strncmp(*p, "NSZombiesEnabled", 16))
        {
            maybeMallocDebugging = true;
        }
	}
	
	...
	
	if (maybeMallocDebugging) {
	        const char *insert = getenv("DYLD_INSERT_LIBRARIES");
	        const char *zombie = getenv("NSZombiesEnabled");
	        const char *pooldebug = getenv("OBJC_DEBUG_POOL_ALLOCATION");
	        if ((getenv("MallocStackLogging")
	             || getenv("MallocStackLoggingNoCompact")
	             || (zombie && (*zombie == 'Y' || *zombie == 'y'))
	             || (insert && strstr(insert, "libgmalloc")))
	            &&
	            (!pooldebug || 0 == strcmp(pooldebug, "YES")))
	        {
	            DebugPoolAllocation = true;
	        }
	    }
}

```
从源码中很明显的看出在Xcode中设置ZombieObject其实就是设置了环境变量NSZombiesEnabled的值为Y或y,并且OBJC_DEBUG_POOL_ALLOCATION环境变量为YES。自己可以打开Zombie通过代码验证一下：
```
    const char *zombie = getenv("NSZombiesEnabled");
    char *pool = getenv("OBJC_DEBUG_POOL_ALLOCATION");
```
两个变量的输出都是NULL，难道猜错了。再验证一下：
```
    for (char **p = *_NSGetEnviron(); *p != nil; p++) {
        printf("\n env:%s",*p);
    }
```
输出：
```
 ...
 env:NSZombieEnabled=YES
 env:CLASSIC=0
 env:SIMULATOR_RUNTIME_BUILD_VERSION=14E8301
 ...
```
明显是设置了NSZombieEnabled环境变量为YES，与源码有出入，但是思路是对的。
设置Zombie Object前后环境变量的改变只有一处，见下图红色部分：
![](/images/posts/zombie_object_check/env_compare.png)
能否直接代码设置环境变量NSZombieEnabled为YES达到目的呢：
```
+ (void)load
{
    setenv("NSZombieEnabled", "YES", 1);
    
    const char *zombie = getenv("NSZombieEnabled");
    printf("\n env:%s",zombie);
}
```
输出：
```
env:YES
```
尽管环境变量已经设置YES，并不起作用，猜测环境变量起作用的时机在+load()方法调用之前。
回过头看看源码，发现设置环境变量后控制的是DebugPoolAllocation变量，尝试将其置为true也不行。
既然这里通过环境变量开控制开关，Apple总归要有实现的地方，查看CF的源码(这里使用的是[CF-855.17](https://opensource.apple.com/tarballs/CF/CF-855.17.tar.gz))
在CFRuntime.c文件中看到两个函数__CFZombifyNSObject,_CFEnableZombies：
```
#if defined(DEBUG) || defined(ENABLE_ZOMBIES)

CF_PRIVATE uint8_t __CFZombieEnabled = 0;
CF_PRIVATE uint8_t __CFDeallocateZombies = 0;

extern void __CFZombifyNSObject(void);  // from NSObject.m

void _CFEnableZombies(void) {
}
```
Xcode打卡Zombie Object分别添加符号断点，__CFZombifyNSObject汇编大致如下：
```
CoreFoundation`__CFZombifyNSObject:
->  0x10b417e10 <+0>:  pushq  %rbp
    0x10b417e11 <+1>:  movq   %rsp, %rbp
    0x10b417e14 <+4>:  pushq  %r15
    0x10b417e16 <+6>:  pushq  %r14
    0x10b417e18 <+8>:  pushq  %rbx
    0x10b417e19 <+9>:  pushq  %rax
    0x10b417e1a <+10>: leaq   0x1f3470(%rip), %rdi      ; "NSObject"
    0x10b417e21 <+17>: callq  0x10b4628ca               ; symbol stub for: objc_lookUpClass
    0x10b417e26 <+22>: movq   %rax, %rbx
    0x10b417e29 <+25>: movq   0x246f88(%rip), %rsi      ; "dealloc"
    0x10b417e30 <+32>: movq   0x2481c9(%rip), %r14      ; "__dealloc_zombie"
    0x10b417e37 <+39>: movq   %rbx, %rdi
    0x10b417e3a <+42>: callq  0x10b4624d4               ; symbol stub for: class_getInstanceMethod
    0x10b417e3f <+47>: movq   %rax, %r15
    0x10b417e42 <+50>: movq   %rbx, %rdi
    0x10b417e45 <+53>: movq   %r14, %rsi
    0x10b417e48 <+56>: callq  0x10b4624d4               ; symbol stub for: class_getInstanceMethod
    0x10b417e4d <+61>: movq   %r15, %rdi
    0x10b417e50 <+64>: movq   %rax, %rsi
    0x10b417e53 <+67>: addq   $0x8, %rsp
    0x10b417e57 <+71>: popq   %rbx
    0x10b417e58 <+72>: popq   %r14
    0x10b417e5a <+74>: popq   %r15
    0x10b417e5c <+76>: popq   %rbp
    0x10b417e5d <+77>: jmp    0x10b462810               ; symbol stub for: method_exchangeImplementations
    0x10b417e62 <+82>: nopw   %cs:(%rax,%rax)

```
看到这个，经常做Swizzle的同学已经看出来了，Apple直接Swizzle了NSObject的dealloc方法，最后实际的实现函数是 __dealloc_zombie，
对__dealloc_zombie添加符号断点：
```
CoreFoundation`-[NSObject(NSObject) __dealloc_zombie]:
->  0x10b418250 <+0>:   pushq  %rbp
    0x10b418251 <+1>:   movq   %rsp, %rbp
    0x10b418254 <+4>:   pushq  %r14
    0x10b418256 <+6>:   pushq  %rbx
    0x10b418257 <+7>:   subq   $0x10, %rsp
    0x10b41825b <+11>:  movq   %rdi, %rbx
    0x10b41825e <+14>:  testq  %rbx, %rbx
    0x10b418261 <+17>:  js     0x10b418305               ; <+181>
    0x10b418267 <+23>:  leaq   0x25a1aa(%rip), %rax      ; __CFZombieEnabled
    0x10b41826e <+30>:  cmpb   $0x0, (%rax)
    0x10b418271 <+33>:  je     0x10b41830e               ; <+190>
    0x10b418277 <+39>:  movq   %rbx, %rdi
    0x10b41827a <+42>:  callq  0x10b462936               ; symbol stub for: object_getClass
    0x10b41827f <+47>:  movq   $0x0, -0x18(%rbp)
    0x10b418287 <+55>:  movq   %rax, %rdi
    0x10b41828a <+58>:  callq  0x10b4624e0               ; symbol stub for: class_getName
    0x10b41828f <+63>:  movq   %rax, %rcx
    0x10b418292 <+66>:  leaq   0x1e9285(%rip), %rsi      ; "_NSZombie_%s"
    0x10b418299 <+73>:  leaq   -0x18(%rbp), %rdi
    0x10b41829d <+77>:  xorl   %eax, %eax
    0x10b41829f <+79>:  movq   %rcx, %rdx
    0x10b4182a2 <+82>:  callq  0x10b462402               ; symbol stub for: asprintf
    0x10b4182a7 <+87>:  movq   -0x18(%rbp), %rdi
    0x10b4182ab <+91>:  callq  0x10b4628ca               ; symbol stub for: objc_lookUpClass
    0x10b4182b0 <+96>:  movq   %rax, %r14
    0x10b4182b3 <+99>:  testq  %r14, %r14
    0x10b4182b6 <+102>: jne    0x10b4182d5               ; <+133>
    0x10b4182b8 <+104>: leaq   0x1e8c7d(%rip), %rdi      ; "_NSZombie_"
    0x10b4182bf <+111>: callq  0x10b4628ca               ; symbol stub for: objc_lookUpClass
    0x10b4182c4 <+116>: movq   -0x18(%rbp), %rsi
    0x10b4182c8 <+120>: xorl   %edx, %edx
    0x10b4182ca <+122>: movq   %rax, %rdi
    0x10b4182cd <+125>: callq  0x10b462888               ; symbol stub for: objc_duplicateClass
    0x10b4182d2 <+130>: movq   %rax, %r14
    0x10b4182d5 <+133>: movq   -0x18(%rbp), %rdi
    0x10b4182d9 <+137>: callq  0x10b462690               ; symbol stub for: free
    0x10b4182de <+142>: movq   %rbx, %rdi
    0x10b4182e1 <+145>: callq  0x10b462882               ; symbol stub for: objc_destructInstance
    0x10b4182e6 <+150>: movq   %rbx, %rdi
    0x10b4182e9 <+153>: movq   %r14, %rsi
    0x10b4182ec <+156>: callq  0x10b46294e               ; symbol stub for: object_setClass
    0x10b4182f1 <+161>: leaq   0x25a121(%rip), %rax      ; __CFDeallocateZombies
    0x10b4182f8 <+168>: cmpb   $0x0, (%rax)
    0x10b4182fb <+171>: je     0x10b418305               ; <+181>
    0x10b4182fd <+173>: movq   %rbx, %rdi
    0x10b418300 <+176>: callq  0x10b462690               ; symbol stub for: free
    0x10b418305 <+181>: addq   $0x10, %rsp
    0x10b418309 <+185>: popq   %rbx
    0x10b41830a <+186>: popq   %r14
    0x10b41830c <+188>: popq   %rbp
    0x10b41830d <+189>: retq   
    0x10b41830e <+190>: movq   %rbx, %rdi
    0x10b418311 <+193>: addq   $0x10, %rsp
    0x10b418315 <+197>: popq   %rbx
    0x10b418316 <+198>: popq   %r14
    0x10b418318 <+200>: popq   %rbp
    0x10b418319 <+201>: jmp    0x10b46237e               ; symbol stub for: _objc_rootDealloc
    0x10b41831e <+206>: nop    
```
Zombie 检查的实现一览无余了。
上面汇编大致的翻译成伪码后：
```
if(__CFZombieEnabled){
    class selfClass = object_getClass(self);
    char *className = class_getName(selfClass);//获取类名
    char *classNameWithZombie;
    asprintf(&classNameWithZombie,"_NSZombie_%s",className);//加_NSZombie_前缀
    class zombieClass = objc_lookUpClass(classNameWithZombie);//返回一个classNameWithZombie类名的class
    if(zombieClass == nil){
        //返回一个“_NSZombie_”类名的class,这个类比较特殊，里面没有任何的方法，
        //所以给这个类发任何消息都会触发消息转发，最后因找不到方法而崩溃。
        class defaultZombieClass = objc_lookUpClass("_NSZombie_");
        //根据defaultZombieClass复制一个类名为classNameWithZombie的class
        zombieClass = objc_duplicateClass(defaultZombieClass,classNameWithZombie,0);
    }
    free(classNameWithZombie);//释放字符串内存
    objc_destructInstance(self);//销毁self，后面会看到该方法的实现。
    object_setClass(self,zombieClass);//设置当前类的内容是zombieClass，即一个没有任何方法的类
    if(__CFDeallocateZombies){
        free(self);//设置NSDeallocateZombies环境变量时真正释放Zombie对象的内存
    }
}
```
objc_destructInstance源码如下：
```
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}
```
一个正常的对象dealloc方法实际是调用object_dispose：
```
id 
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);    
    free(obj);

    return nil;
}
```
看到上面的源码就清晰了，objc_destructInstance并没有释放class的内存，释放内存放生在object_dispose，这里并没有调用。

### 小结
Apple的Zombie 检查功能实际是Swizzle NSObject的dealloc方法，创建一个不包哈任何方法的class将原来的class内存内容替换掉，这样当已经调用dealloc的对象再被发送任何消息（调用函数）时都会出发OC的消息转发机制,只需在添加消息转发过程中的函数，并在函数里处理输出log即可。由于__CFZombieEnabled是私有变量，自己的工程中并不能修改__CFZombieEnabled变量的值，即使自己Swizzle dealloc 到 __dealloc_zombie ，因为__CFZombieEnabled的判断过不去也不能正常的使用。

### 自己实现
因为已经看到完整的汇编自己实现并不难。
自己的dj_zombie_dealloc方法：
```
- (void)dj_zombie_dealloc
{
    BOOL zombieClassHasResigted = YES;
    NSString *className = NSStringFromClass(self.class);
    NSString *zombieClassName = [kDJZombieLong stringByAppendingString:className];
    const char *zombieClassNameUtf8 = zombieClassName.UTF8String;
    
    Class zombieClass = objc_lookUpClass(zombieClassNameUtf8);
    if (!zombieClass) {
        zombieClassHasResigted = NO;
        Class tmpClass = objc_lookUpClass(kDJZombieShort);
        DumpObjcMethods(tmpClass);
        zombieClass = objc_duplicateClass(tmpClass,zombieClassNameUtf8,0);
        DumpObjcMethods(zombieClass);
    }
    
    objc_destructInstance((id)self);
    object_setClass((id)self,zombieClass);
    
    if (zombieClassHasResigted == NO) {
        SEL originalSignatureSelector = @selector(methodSignatureForSelector:);
        SEL swizzledSignatureSelector = @selector(dj_methodSignatureForSelector:);
        //_NSZombie_ has no method ,just add it.
        Method swizzledSignatureMethod = class_getInstanceMethod(NSObject.class, swizzledSignatureSelector);
        class_addMethod(zombieClass, originalSignatureSelector, method_getImplementation(swizzledSignatureMethod), method_getTypeEncoding(swizzledSignatureMethod));
        
        SEL originalInvocationSelector = @selector(forwardInvocation:);
        SEL swizzledInvocationSelector = @selector(dj_forwardInvocation:);
        Method swizzledInvocationMethod = class_getInstanceMethod(NSObject.class, swizzledInvocationSelector);
        class_addMethod(zombieClass, originalInvocationSelector, method_getImplementation(swizzledInvocationMethod), method_getTypeEncoding(swizzledInvocationMethod));
        
        DumpObjcMethods(zombieClass);
    }
}
```
DJZombieCheck为了获取到给Zombie对象发消息时的参数，使用了消息转发机制的methodSignatureForSelector:和forwardInvocation:,所以给空的class添加了两个方法来接收ZombieObject的消息，其中处理完后自己抛出包含详细信息的异常:
```
void dj_throwExceptionForSelector(id selfInstance,SEL selector,NSArray *paramList)
{
    Class selfClass = object_getClass(selfInstance);
    NSString *zombieClassName = [NSString stringWithUTF8String:class_getName(selfClass)];
    NSString *originalClassName = [zombieClassName stringByReplacingOccurrencesOfString:kDJZombieLong withString:@""];
    
    id paramLog = paramList ? paramList : @"empty";//empty string is also param,so use "empty" to replace
    NSString *zombieLog = [NSString stringWithFormat:@"Find zombie,class:%@ address:%p selector:%@ param:%@\r\n",originalClassName,selfInstance,NSStringFromSelector(selector),paramLog];
    NSLog(@"%@", zombieLog);
    
    if ([DJZombieCheckHanlder sharedInstance].zombieHandler) {
        [DJZombieCheckHanlder sharedInstance].zombieHandler(originalClassName, selector, paramList);
    }
    
    NSException *zombieException = [NSException exceptionWithName:@"DJZombieException" reason:zombieLog userInfo:nil];
    [zombieException raise];
}

```
修改全局变量DJZombieCheckEnable的值来关闭DJZombieCheck:
```
BOOL DJZombieCheckEnable = NO;
```
如果需要上传Zombie信息到服务端：
```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    //read last crash log and send it to server.
    
    [[DJZombieCheckHanlder sharedInstance] setZombieHandler:^(NSString *className, SEL selector, NSArray *paramList){
        //save crash log
    }];
    return YES;
}
```
### 注意：
* 由于Zombie检查一直在申请内存，长时间使用后会有内存压力问题，注意OOM，这个Xcode自带的也有同样问题。
* 如果Xcode自带的和DJZombieCheck同时打开，DJZombieCheck会自动关闭。

完整的实现以及使用方法见：[DJZombieCheck](https://github.com/Dokay/DJZombieCheck)

原创文章，转载请注明出处。

