
# 附录二： HDFS Quotas Guide（English version）

####(注：后附中文版，强烈建议先看英文文档)

- HDFS Quotas Guide
  - Name Quotas
  - Space Quotas
  - Administrative Commands
  - Reporting Command
   
**overview**

The Hadoop Distributed File System (HDFS) allows the administrator to set quotas for the number of names used and the amount of space used for individual directories. Name quotas and space quotas operate independently, but the administration and implementation of the two types of quotas are closely parallel.

**Name Quotas**

The name quota is a hard limit on the number of file and directory names in the tree rooted at that directory. File and directory creations fail if the quota would be exceeded. Quotas stick with renamed directories; the rename operation fails if operation would result in a quota violation. The attempt to set a quota will still succeed even if the directory would be in violation of the new quota. A newly created directory has no associated quota. The largest quota is Long.Max_Value. A quota of one forces a directory to remain empty. (Yes, a directory counts against its own quota!)

Quotas are persistent with the fsimage. When starting, if the fsimage is immediately in violation of a quota (perhaps the fsimage was surreptitiously modified), a warning is printed for each of such violations. Setting or removing a quota creates a journal entry.

**Space Quotas**

The space quota is a hard limit on the number of bytes used by files in the tree rooted at that directory. Block allocations fail if the quota would not allow a full block to be written. Each replica of a block counts against the quota. Quotas stick with renamed directories; the rename operation fails if the operation would result in a quota violation. A newly created directory has no associated quota. The largest quota is Long.Max_Value. A quota of zero still permits files to be created, but no blocks can be added to the files. Directories don’t use host file system space and don’t count against the space quota. The host file system space used to save the file meta data is not counted against the quota. Quotas are charged at the intended replication factor for the file; changing the replication factor for a file will credit or debit quotas.

Quotas are persistent with the fsimage. When starting, if the fsimage is immediately in violation of a quota (perhaps the fsimage was surreptitiously modified), a warning is printed for each of such violations. Setting or removing a quota creates a journal entry.

**Administrative Commands**

Quotas are managed by a set of commands available only to the administrator.
```
hdfs dfsadmin -setQuota <N> <directory>...<directory>
```
Set the name quota to be N for each directory. Best effort for each directory, with faults reported if N is not a positive long integer, the directory does not exist or it is a file, or the directory would immediately exceed the new quota.

```
hdfs dfsadmin -clrQuota <directory>...<directory>
```

Remove any name quota for each directory. Best effort for each directory, with faults reported if the directory does not exist or it is a file. It is not a fault if the directory has no quota.
```
hdfs dfsadmin -setSpaceQuota <N> <directory>...<directory>
```
Set the space quota to be N bytes for each directory. This is a hard limit on total size of all the files under the directory tree. The space quota takes replication also into account, i.e. one GB of data with replication of 3 consumes 3GB of quota. N can also be specified with a binary prefix for convenience, for e.g. 50g for 50 gigabytes and 2t for 2 terabytes etc. Best effort for each directory, with faults reported if N is neither zero nor a positive integer, the directory does not exist or it is a file, or the directory would immediately exceed the new quota.
```
hdfs dfsadmin -clrSpaceQuota <directory>...<directory>
```
Remove any space quota for each directory. Best effort for each directory, with faults reported if the directory does not exist or it is a file. It is not a fault if the directory has no quota.

**Reporting Command**

An an extension to the count command of the HDFS shell reports quota values and the current count of names and bytes in use.
```
hadoop fs -count -q [-h] [-v] <directory>...<directory>
```
With the -q option, also report the name quota value set for each directory, the available name quota remaining, the space quota value set, and the available space quota remaining. If the directory does not have a quota set, the reported values are none and inf. The -h option shows sizes in human readable format. The -v option displays a header line.


**原文请参考Apache官方网站：**
[http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsQuotaAdminGuide.html]()


#HDFS Quotas Guide（中文版）
**概述**

HDFS允许管理员为使用的命名和每个文件夹设置配额。命名配额和空间配额独立操作，但是这两种陪管理和实现是连接紧密的。

**命名配额**

命名配额是一个在这个文件夹下文件和文件夹的数目。如果超过限额那么文件和文件夹的创建会失败，重命名后命名配额仍然起作用。如果重命名操作违反配额的限制，那么重命名会失败。新创建的目录中没有配额的限制。Long.Max_Value表示最大限额。如果配额为1那么这个文件夹会强制为空。(一个目录也占用自己的配额)。
配额被持久化在fsimage中，当启动后，如果fsimage 马上违反了配额限制（由于fsimage偷偷的改变），这是会打印警告。设置或删除配额会创建一个空的日志。

**空间配额**

空间配额是设置一个文件夹的大小。如果超过那么块写入会失败。副本也算配额中的一部分。重命名文件夹后配额还是起作用，如果已经违反了配额，那么重命名操作会失败。新创建的文件夹不会有配额的限制，Long.Max_Value可以设置最大的配额。配额设置为0还是运行文件创建，但是不能向文件中写入块。文件夹不使用主机文件系统不计算在空间配额里面，主机文件系统用来记录文件源数据的数据不算在配额中。配额被持久化在fsimage中，当启动后，如果fsimage 马上违反了配额限制（由于fsimage偷偷的改变），这是会打印警告。设置或删除配额会创建一个空的日志。

**管理命令**

只有管理员可以设置配额。
```
dfsadmin -setQuota <N> <directory>...<directory>
```

为文件夹设置命名配额为N，当N不是正整数，文件夹不存在或者是一个文件，或者是文件已经超过了新的配额的时候会报错，
```
dfsadmin -clrQuota <directory>...<directory>
```

删除命名配额。如果文件夹不存在或者是一个文件会报错。
```
dfsadmin -setSpaceQuota <N> <directory>...<directory>
```
设置每个文件夹的空间配额为Nbytes。这是在此文件夹下所有文件的大小和的限制。空间配额需要考虑复制，比如1GB的数据并且一共三个副本（包含原文件）那么会消耗3GB的空间配额。N可以指定一个前缀，比如50g 就是50GB 2t就是2TB等。如果N是负数，文件夹不存在或者是一个文件，或者文件夹已经超出了新的配额会抛异常
```
dfsadmin -clrSpaceQuota <directory>...<director>
```
为每一个文件夹删除空间配额

**报告命令**

HDFS的shell可以使用count命令去查询配额的值和当前的name和字节被用了。

```
fs -count -q <directory>...<directory>
```
使用-q选项，可以报告文件夹的命名配额，剩余命名配额，空间配额，可用空间配额。如果文件夹没有设置配额，那么会报告值为none和inf