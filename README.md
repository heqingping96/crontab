# Golang实现分布式任务调度

**谈到任务调度，我们就不得不提到linux下的crontab任务调度，我们使用到crontab的时候，往往会遇到几类问题：**

1. **在配置任务时，需要ssh登录到服务器上进行操作，我们的任务可能就是一个脚本，定时执行，一般情况下并没有什么问题，但是当我们的任务变大变多之后，就不可避免得把任务分散到很多服务器上去，这时候如果我们需要管理我们的任务就变得很麻烦了，我们需要精确的知道我们的服务器部署在哪台服务器上，然后连接上去**
2. **crontab是一个单机的调度，任务部署在服务器上时，如果我们的服务器宕机了，任务就停止调度了，我们的业务就会受到影响，我们需要进行人工的调度，把它迁移到健康的服务器上去，比较麻烦**
3. **排查问题低效，假如我们的业务已经挂掉了，但是我们还不知道其挂掉的具体原因，crontab没有提供查看脚本运行出错原因的日志，无法方便的查看任务状态与错误输出。**
4. **通过上述的描述可以看出，传统的crontab存在很多的问题，所以我们来实现一个分布式的任务调度。**

## 需要解决的问题
1. **可视化web,方便任务管理，通过简单的点击鼠标的方式就可以管理任务。**
2. **解决服务器宕机所带来的影响，我们进行分布式，集群化的调度，不存在单点故障**
3. **追踪任务执行状态，采集任务输出，可视化log查看。**

## 分布式架构
1. **本项目我们主要采用master-worker分布式架构，集群中有两种角色，我们主要是依托etcd这个分布式协调服务，etcd在本项目的主要作用是会实现集群当中的任务分发，在项目中，我们会有一个强杀的按钮，这是为了避免任务长时间没有得到调度，这时候我们就会把这个任务进行强杀，传统的做法我们是登录到服务器上执行kill,但是在本系统当中我们可以通过简单的强杀按钮来进行把它杀死，这个就相当于是一个事件广播，也是基于etcd来操作的，当我们点击按钮时，这个时间就会下发到集群当中，任务就会立马死掉。在任务在集群当中并发调度的时候，我们就会基于etcd实现一个分布式锁，来防止任务并发得调度，包括服务注册与发现，这些都是基于etcd来实现的。**
2. **任务所产生的日志，会通过异步传输的方式传输到mongodb,然后展示到web后台上。**

## 系统架构
![image](https://note.youdao.com/yws/api/group/109236362/noteresource/88C25B98C768422F9A584C058728B32C/version/278?method=get-resource)

1. **项目包含两种角色，master和worker，WebConsole是前端页面，前端页面会前后端分离调用master后端的APIserver,调用一些接口做任务的管理。**
2. **任务会包存到etcd中，任务一旦保存到etcd中，就会实时通过etcd的监听机制同步全量任务列表到worker节点，这时所有的worker都有同样的任务列表，收到任务就会进入到任务调度Scheduler模块,基于cron表达式每个worker独立做多任务的并发调度，无需与master产生直接的RPC，当某个任务到期之后就会交给Executor模块并发得执行，在任务执行的过程中会基于etcd上分布式乐观锁，来进行调度互斥，保证一个任务在集群当中不会并发执行。**
3. **任务执行完成之后的日志会异步的写入MongoDB当中，然后通过master的日志管理给web后台提供一个展现。**

## 项目实现效果
1. ![image](https://note.youdao.com/yws/api/group/109236362/noteresource/0794C3D00CE14404980B3A558C3DF28B/version/282?method=get-resource)
 **新建任务**
2. ![image](https://note.youdao.com/yws/api/group/109236362/noteresource/B34C51DE14544B80AE60D9965ABD8A43/version/284?method=get-resource)**编辑任**务
3. ![image](https://note.youdao.com/yws/api/group/109236362/noteresource/23C0BED029554C6EA652ED709DD9DDAD/version/286?method=get-resource)**查看集群当中的健康节点**
4. ![image](https://note.youdao.com/yws/api/group/109236362/noteresource/D175A57C9856495BB97080F3C21FA2DC/version/288?method=get-resource)**查看任务执行日志**
5. ![image](https://note.youdao.com/yws/api/group/109236362/noteresource/3F1BFA6253B64899BED181257890DF27/version/289?method=get-resource)**点击强杀按钮任务日志发生变化**


