```
1.  //用户必须实现的方法: 用于封装新引擎连接器实例启动命令

2.  def buildApplication(protocol:Protocol):ApplicationRequest  
```





```
1.  //用户必须实现的方法：用于调用底层引擎提交执行计算任务

2.  def executeLine(context: EngineConnContext,code: String): ExecuteResponse  
```

 