阅读AFNetworking杂记

AFURLSessionManager

- 通过队列管理多个task的任务
- 通过swizzle去在已有的函数中添加一些自己的东西，例如通知等等
- block和nsstring以及nsurl等都使用copy属性 (block使用copy协议是因为block本身是创建在栈中，属于域内变量，很多时候为了在域外也使用block，程序内部会把block拷贝一份到堆中，所以最好直接使用copy协议)
- 使用锁(nslock)去保证多个线程一起去维护同一张task的key-value表格时使其能够是线程安全操作
- 通过大量的block把多个代理的方法包装起来，实现接口统一

AFHTTPSessionManager

- 是AFURLSessionManager的子类，提供便捷常用的HTTP请求接口，可以配置baseURL，request编码器，和response解析器，以及安全协议的设置

