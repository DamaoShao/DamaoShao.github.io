# SCVMM的远程管理实践

最近公司新签了一个客户，他们有在用Hyper-V，所以需要我们平台接入SCVMM，纳管他们的Hyper-V机器，开始我以为这事类似接入OpenStack、VSphere一样呢，结果一上手才发现，这是天坑，SCVMM居然没有自带API服务，更别提SDK了。客户需求大于天，没有API也得硬着头皮上。本篇blog就讲讲我目前碰到的坑。

##  SPF还是PowerShell

SCVMM本身没有API服务，但是可以安装外挂API服务，即Service Provider Foundation（简称SPF）。SPF本质就是网络服务器，需要一些组件的支撑（SQLServer，.Net等）。操作SCVMM的另外一种方式PowerShell。

最后，我们选择使用PowerShell，这绝不是因为PowerShell好用，而且SPF本身的缺陷，逼着我们选了PowerShell：

+ SPF本质就是网络服务器，需要一些组件的支撑（SQLServer，.Net等），并且对组件版本有要求。在客户环境上安装这么多东西，客户可能会不同意，即使同意了，还有兼容性的问题，不符合我们产品化的原则。
+ SPF并不是专门给SCVMM做API服务，测试后发现，有些SCVMM操作SPF居然没有对应的API。

## 自建Agent还是使用WinRM

既然确定了使用PowerShell，那么下面的问题就是如何远程执行PowerShell命令。又是两条路，在客户机器上面搭Agent（HTTP服务），或者使用WinRM远程执行，经过比较，自搭HTTP服务的Agent（用Flask起的服务）比使用WinRM要快得多，平均一个命令自搭Agent要比Winrm能快出1到2秒。但是使用Agent有两个问题，一个是Agent要自己写，这是工作量，后期维护这又是工作量。另外一个是就是不安全，毕竟在客户环境上面开端口起服务，而且涉及管理权限。因为自搭Agent这两个缺点，我们最后决定选用WinRM远程执行方案（即使它真的很慢orz）。

## WinRM的编码(字符集)问题

跨平台总逃不出编码问题，这次是在Linux上面远程执行Windows的PowerShell命令，编码就是头号拦路虎。

首先是字符集。使用pywinrm库执行PowerShell命令的输出中，如果有中文，就会显示成`?`。看了一圈源码，我发现，pywinrm库暴露出来的执行PowerShell命令的`run_ps`方法默认使用的字符集是437，这是Windows中美国字符集编码（中国字符集编码是936），为了统一使用`utf-8`写代码，这里最好指定成`utf8`的字符集编码`65001`。字符集在`Protocol`类中的`open_shell`方法中可以指定。

然后是编码问题。指定了`65001`的字符集后，一旦远程执行的命令有问题，标准错误中含有中文，代码就报无法编码的错误。报错位置在`session`类的`_clean_error_msg`中。上下文看一下，就很容易发现根本原因pywinrm解码方式是ascii码，对，你没有看错，9102年了，pywinrm居然还在用字符集`437`和ascii解码。没有办法，我只能重写了输出的方法，将将解码改成`utf8`。

```python
# 为了编码过程中统一使用utf-8，重写下面两个方法，修改了字符集和解码方式。
# set codepage to 65001(the powershell utf-8 charset code is 65001)
# use utf-8 to decode
class SessionUTF8(Session):
    def run_cmd(self, command, args=()):
        shell_id = self.protocol.open_shell(codepage=65001)
        command_id = self.protocol.run_command(shell_id, command, args)
        rs = Response(self.protocol.get_command_output(shell_id, command_id))
        self.protocol.cleanup_command(shell_id, command_id)
        self.protocol.close_shell(shell_id)
        return rs

    def run_ps(self, script):
        encoded_ps = b64encode(script.encode('utf_16_le')).decode('utf-8')
        rs = self.run_cmd('powershell -encodedcommand {0}'.format(encoded_ps))
        if len(rs.std_err):
            rs.std_err = self._clean_error_msg(rs.std_err.decode('utf-8'))
        return rs
```

## PowerShell的输出正则问题

SCVMM的Powershell输出就是一行行的`:`隔开的键值对，如果有多个对象，每个对象之间是空行隔开。最坑是，每行最多80字符，超过80字符就会换行。为了方便前端使用，我需要把这格式转成Json。

```yaml
# PowerShell的输出格式
ServerConnection                  : Microsoft.SystemCenter.VirtualMachineManage
                                    r.Remoting.ServerConnection
ID                                : c755cbc0-d481-49db-b0a5-484c8f930767
MarkedForDeletion                 : False
ComputerTierTemplate              :
CapabilityProfile                 :
ApplicationProfile                :
SQLProfile                        :
ServerFeatures                    : {}
TotalVHDCapacity                  : 42949672960
VirtualizationPlatform            : HyperV
```
```text
# pywinrm的输出的字符串。记为output。
ServerConnection                  : Microsoft.SystemCenter.VirtualMachineManage\r\n                                    r.Remoting.ServerConnection\r\nID                                : afafac26-4b05-4458-9eb8-529a589ec0cb\r\nMarkedForDeletion                 : False\r\nComputerTierTemplate              : \r\nCapabilityProfile                 : \r\nApplicationProfile                : \r\nSQLProfile                        : \r\nServerFeatures                    : {}\r\nTotalVHDCapacity                  : 42949672960\r\nVirtualizationPlatform            : Hyper
```

一开始我准备用正则解决这个问题的，但是作为一个正则菜鸡，实在想不出健壮的正则来解决换行问题。所以我首先想到的是，让输出不要换行。经过一番尝试，我发现在本地将PowerShell的输出重定向到文本中，那么文件里面的输出是不换行的。但是这仅仅在于本地，一旦使用pywinrm执行，哪怕重定向到文本中也换行orz

这条路走不通，我只能想办法把换行合并回去了。分析输出结构，我发现，每个对象间的分隔是`\r\n\r\n`，每行的分隔是`\r\n`，键和值的分隔是`:`，这直接用字符串操作就能解决啦，出现`\r\n`带着一大串空格的就是`值`换行，直接替换成`''`，等于就把换成合并回去了，然后在按对象（`\r\n\r\n`）分隔，每个对象按`:`分隔，就把键值对分解出来啦。仅仅需要两个列表推导，很舒服。

```python
EMPTY_STRING = ''
ONE_BLANK_SPACE = ' '
KV_DEFAULT_INTERVAL = ': '
NEWLINE = '\r\n'
KV_OFFSET = 2


def _preprocess_merge_newlines(text):
    """
    change string to list, merge the newlines
    text: str
    return: list
    """
    components = [component for component in text.split(NEWLINE * 2) if component != EMPTY_STRING]
    interval = NEWLINE + (int(components[0].find(':')) + KV_OFFSET) * ONE_BLANK_SPACE
    return map(lambda component: component.replace(interval, EMPTY_STRING), components)


def _preprocess_regex(components):
    """
    change string to json in the list
    :param components: [string,string,string]
    :return: [json,json,json]
    """

    return [dict([(key_value.split(KV_DEFAULT_INTERVAL)[0].strip(),
                   key_value.split(KV_DEFAULT_INTERVAL)[1].strip())
                  for key_value in component.split(NEWLINE)]) for component in components]


def common_regex(text):
    """
    convert the output of powershell into json.
    :param text: str
    :return: [json,json,json] --> every json is instance property set
    """
    components = _preprocess_merge_newlines(text)
    return _preprocess_regex(components)
```
经过转换，`output`变为：
```text
[{'ServerConnection': 'Microsoft.SystemCenter.VirtualMachineManager.Remoting.ServerConnection',
  'ID': 'afafac26-4b05-4458-9eb8-529a589ec0cb',
  'MarkedForDeletion': 'False',
  'ComputerTierTemplate': '',
  'CapabilityProfile': '',
  'ApplicationProfile': '',
  'SQLProfile': '',
  'ServerFeatures': '{}',
  'TotalVHDCapacity': '42949672960',
  'VirtualizationPlatform': 'Hyper'}]
```


