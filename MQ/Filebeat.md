### 配置

#### filebeat.inputs

- scan_frequency：Filebeat在指定的获取路径中检查新文件的频率；如果您需要近实时发送日志行，请不要使用非常低的扫描频率，而是将close_inactive设置一个高的值，以便文件处理程序保持打开状态并不断轮询您的文件，默认为10s；
- close_inactive：启用此选项后，如果在指定的持续时间内未获取文件，Filebeat将关闭文件句柄，当读取最后一行日志时，定义周期的计数器开始；默认5m（分钟），如果关闭的文件再次更改，则会启动一个新的句柄；将close_inactive设置为较低的值意味着文件句柄关闭得更快。然而，这有一个副作用，即如果文件句柄关闭，新的行数据不会近实时发送；
- backoff：检查文件中新行的频率，默认1s
- max_backoff：扫描到行尾后，再次检查文件之前等待的最长时间，默认10s；要求：backoff <= max_backoff <= scan_frequency；
- backoff_factor：等待时间增加的速度，值越大，到达max_backoff的速度越快；默认值为2；

#### output.elasticsearch

- bulk_max_size：Elasticsearch单个_bulk请求中要批量处理的最大事件数，默认值为50；
- max_retries：最大重试次数
- timeout：http请求超时（以秒为单位），默认值为90。
- worker：向Elasticsearch发布事件的每个配置主机（每个配置的host）的工作进程数。这最好在启用负载平衡模式时使用。示例：如果有2台主机和3个工作线程，则总共启动6个工作线程（每个主机3个）

### 场景分析

#### 数据重复

- Filebeat重传导致数据重复。重传是因为Filebeat要保证数据至少发送一次，进而避免数据丢失。具体来说就是每条event发送到output后都要等待ack，只有收到ack了才会认为数据发送成功，然后将状态记录到registry。当然实际操作的时候为了高效是批量发送，批量确认的。而造成重传的场景（也就是没有收到ack）非常多，而且很多都不可避免，比如后端不可达、网络传输失败、程序突然挂掉等等。
- 配置不当或操作不当导致文件重复收集。Filebeat感知文件有没有被收集过靠的是registry文件里面记录的状态，如果一个文件已经被收集过了，但因为各种原因它的状态从registry文件中被移除了，而恰巧这个文件还在收集范围内，那就会再收集一次。

对于第一类产生的数据重复一般不可避免，而第二类可以避免，但总的来说，Filebeat提供的是at least once的机制，所以我们在使用时要明白数据是可能重复的。如果业务上不能接受数据重复，那就要在Filebeat之后的流程中去重。



#### 数据截断和丢失

Filebeat这里保证的没有数据丢失也同样是有前提条件的，就是只保证数据传输时不丢失；Filebeat自身机制方面的一些缺陷，导致即使你的使用方式完全正确，数据在采集这一步就丢失的问题依旧是无法完全避免的；Filebeat处理文件时会维护一个状态，这个状态里面记录了收集过的每一个带绝对路径的文件名，文件的inode值，文件内容上次收集的位置（即offset）以及其它一些信息；这个机制上的缺陷主要和Filebeat判断文件是否truncate的方式有关系。我们先看下判断的代码：

```go
// harvestExistingFile continues harvesting a file with a known state if needed
func (p *Input) harvestExistingFile(newState file.State, oldState file.State) {
  // 省略部分代码

    // File size was reduced -> truncated file
    if oldState.Finished && newState.Fileinfo.Size() < oldState.Offset {
        logp.Debug("input", "Old file was truncated. Starting from the beginning: %s, offset: %d, new size: %d ", newState.Source, newState.Fileinfo.Size())
        err := p.startHarvester(newState, 0)
        if err != nil {
            logp.Err("Harvester could not be started on truncated file: %s, Err: %s", newState.Source, err)
        }

        filesTruncated.Add(1)
        return
    }

  // 省略部分代码
}
```

registry文件里面虽然记录了文件名，但Filebeat唯一标识一个文件使用的是里面的inode值，而非文件名，但操作系统的**inode值是会复用**的。比如原来有一个文件A，Filebeat处理过之后将其inode，以及处理的offset（假设为n）记录到了registry文件中。后来这个文件删除了，但registry里面记录的状态还没有自动删除，此时如果有另外一个文件B正好复用了之前A的inode，那Filebeat就会认为这个文件之前处理过，且已经处理到了offset为n处。如果B的文件比A小，即文件的end offset都小于n，那Filebeat就会认为是原来的A文件被truncate掉了，此时会从头开始收集，没有问题。但如果B的end offset大于等于n，那Filebeat就认为是A文件有更新，然后就会从offset为n处开始处理，于是B的前n个字节的数据就丢失了，这样我们就会看到数据有被截断。**数据截断**属于数据采集时丢失的一种情况。

还有一些其它情况，比如文件数太多，Filebeat的处理能力有限，在还没来得及处理的时候这些文件就被删掉了（比如rotate给老化掉了）也会造成数据丢失。还有就是后端不可用，所以Filebeat还在重试，但源文件被删了，那数据也就丢了。因为Filebeat的重试并非一直发送已经收集到内存里面的event，必要的时候会重新从源文件读，比如程序重启。这些情况的话，只要不限制Filebeat的收集能力，同时保证后端的可用性，网络的可用性，一般问题不大。