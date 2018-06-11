## 从一个Button说开去

一个最基本的UIbutton的使用大概应该是这个样子的：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    UIButton *testButton = [UIButton buttonWithType:UIButtonTypeCustom];
    testButton.backgroundColor = [UIColor redColor];
    testButton.frame = CGRectMake(100, 100, 100, 100);
    [self.view addSubview:testButton];
    [testButton addTarget:self action:@selector(test:) forControlEvents:UIControlEventTouchUpInside];
}

- (void)test:(UIButton*)sender{
    NSLog(@"test");
}
```
那么问题来了，想一想，如果现在临时加一个需求，button响应事件之前要先获取相机视频权限，应该怎么做？（举个例子，同样的需求还有获取位置权限，检测网络连接，查看登录状态等等）

先不说button，权限获取的代码大概应该长这样：

```
AVAuthorizationStatus authStatus =  [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
    if (authStatus == AVAuthorizationStatusRestricted || authStatus ==AVAuthorizationStatusDenied){
        //啥也不干
    }else if(authStatus == AVAuthorizationStatusNotDetermined){
        [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo completionHandler:^(BOOL granted) {
            if(granted){
                dispatch_async(dispatch_get_main_queue(), ^{
                    //干正事儿
                });
            }
        }];
    }else{
        //干正事儿
    }
```

需要注意的就一点,同意获取权限的回调需要返回主线程。

还有你需要设置info.plist的Privacy。

说起来可能有些搞笑，很大一部分的工程中，上面那段代码就这堂而皇之的躺在`- (void)test:(UIButton*)sender`方法里面。

当然这个写法实在是太...，除了萌新之外很少真的有人这么写了，但是其实上面那段代码其实还有以下几个变种：
1. 知道将button原本的响应事件单独提出来，至少不用写两遍。
2. 将获取权限的代码封装起来，大概这样：


 
    [Util cameraAuth:^{
       //干点啥    
    } fail:^{
        
    }];
   
当然还有的大胸弟是以上两种方法互相结合着来写的。

好了，写到这里，50%的开发者已经躺枪了。

“没错我们就是这么写的！”

这时候有人知道我想要说什么吗？对！万恶的产品经理又来了。

“我需要你在这个button事件里，再加上照片权限获取，位置权限获取，音频权限获取！”

然后代码就变成了：


	[Util cameraAuth:^{
   		[Util audioAuth:^{
       	 [Util photoAuth:^{
          	 [Util locationAuth:^{
           
    } fail:^{
        
    }];    
    } fail:^{
        
    }];      
    } fail:^{
        
    }];    
    } fail:^{
        
    }];

就问你怕不怕？

别着急，饭一口一口吃，我们现在先来拯救一下这50%的小伙伴。

其实很简单，你需要的是一个UIButton的子类。（什么玩意儿？裤子都脱了你就给我看这个？？）

是的就是这样，一个UIbutton的子类，需要实现的方法大概如下：

```
-(void)sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event{
    AVAuthorizationStatus authStatus =  [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
    if (authStatus == AVAuthorizationStatusRestricted || authStatus ==AVAuthorizationStatusDenied){
        
    }else if(authStatus == AVAuthorizationStatusNotDetermined){
        [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo completionHandler:^(BOOL granted) {
            if(granted){
                dispatch_async(dispatch_get_main_queue(), ^{
                    [super sendAction:action to:target forEvent:event];
                });
            }
        }];
    }else{
         [super sendAction:action to:target forEvent:event];
    }
    
}
```
这样的话，你就可以肆无忌惮的和原生的UIbutton一样去使用它了，并不需要在原本的action中添加任何的代码。

当然有必要的话还应该对event做一下区分，以免影响到这个button的其他功能交互。

有没有人这么做？我想肯定有，而且不算少，初步估计应该有个10%左右吧。

如果你真的是这么做的，恭喜你掉坑了，这种写法有个弊端就是使用button子类替换很不方便，工作量大，而且降低了可读性，总感觉哪里不太对。

没关系，至少路子走对了，如果想要继续完善这个思路，下面的这个变种了解一下。

你知道 ~~安利~~ runtime吗？

如果你对runtime的了解和使用仅限于`Method Swizzling`和`objc_setAssociatedObject`的话，往下看一定会有收获。

新建一个UIbutton的类别，假设之前的button子类为`SKButton`,则添加方法如下：

```
- (void)setNeedsCameraPermission{
    object_setClass(self, [SKButton class]]);
}
```
是的你没有看错，只需要一句话，就可以把一个UIbutton，变成他的子类，不需要`#import`，不需要改类名，屠龙宝刀点击就送，是不是很方便？

but...

你以为完了吗？怎么可能。

上面的写法和直接替换一个UIbutton的子类一样，有一个共同的弊端，就是当工程内部使用的button控件本身就已经是一个写好的轮子了，也是UIbutton的子类，那你怎么办？

将`SKButton`的父类由`UIbutton`改为当前子类？呸！不要脸！

这个思路对吗?当然对！

但是作为一个轮子，别人拿去之后还没使用就要先补胎，你好意思吗？

所以在runtime中，不仅可以动态变更类，还可以动态创建类，你知道吗？

一个动态创建的支持获取相机权限的button的代码大概长这样：

```
- (void)setNeedsCameraPermission{
    NSString *className = [NSString stringWithFormat:@"CameraPermission_%@",self.class];
    Class kclass = objc_getClass([className UTF8String]);
    if (!kclass)
    {
        kclass = objc_allocateClassPair([self class], [className UTF8String], 0);
    }
    SEL setterSelector = NSSelectorFromString(@"sendAction:to:forEvent:");
    Method setterMethod = class_getInstanceMethod([self class], setterSelector);
    object_setClass(self, kclass);
    const char *types = method_getTypeEncoding(setterMethod);
    class_addMethod(kclass, setterSelector, (IMP)camerapermission_SendAction, types);
    objc_registerClassPair(kclass);
}

static void camerapermission_SendAction(id self, SEL _cmd, SEL action ,id target , UIEvent *event)
{
        struct objc_super superclass = {
        .receiver = self,
        .super_class = class_getSuperclass(object_getClass(self))
    };
    void (*objc_msgSendSuperCasted)(const void *, SEL, SEL, id, UIEvent*) = (void *)objc_msgSendSuper;
    AVAuthorizationStatus authStatus =  [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
    if (authStatus == AVAuthorizationStatusRestricted || authStatus ==AVAuthorizationStatusDenied){
        
    }else if(authStatus == AVAuthorizationStatusNotDetermined){
        [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo completionHandler:^(BOOL granted) {
            if(granted){
                dispatch_async(dispatch_get_main_queue(), ^{
                    objc_msgSendSuperCasted(&superclass, _cmd,action,target,event);
                });
            }
        }];
    }else{
        objc_msgSendSuperCasted(&superclass, _cmd,action,target,event);
    }
}
```
这样就动态创建并替换了一个叫做“CameraPermission_XXXXXX”的button子类，任何一个button，只需要调用`setNeedsCameraPermission`方法，就能够为button添加权限获取功能了。

能够写到这里的话，基本上就差不多了，不过真的有人有耐心把这么烂的文章看完吗？

如果你真的看到这的话，那一定是因为爱情了，你也一定发现了我似乎漏掉了什么东西，我当然是故意的！

好了下面给你留一个作业，如果让你动态创建一个可自由组合，同时获取多个权限的button子类，你会写吗？








