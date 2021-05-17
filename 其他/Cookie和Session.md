#### Cookie与Session有什么区别？

- session保存在服务器端，cookie在客户端（浏览器）
- session的运行依赖session id，而session id是存在cookie中的，也就是说，如果浏览器禁用的cookie，同时session也会失效，这时要使用url重写的技术来进行会话跟踪，即每次HTTP交互，URL后面被附加上一个诸如sid=xxx的参数。
- session是在服务端保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群、数据库、文件中。
- cookie是客户端保存用户信息的一种机制，用来记录用户的一些信息，也是实现session的一种方式。
- session cookie针对某一次会话而言，会话结束session cookie也就随着消失了，而persistent cookie只是存在于客户端硬盘上的一段文本（通常是加密的），而且可能会遭到cookie欺骗以及针对cookie的跨站脚本攻击，自然不如 session cookie安全了。