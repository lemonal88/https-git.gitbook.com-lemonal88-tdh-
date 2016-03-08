# 附录三：HDFS Permission Guide（U.S version） 
####(注：后附中文版，强烈建议先看英文文档)

- HDFS Permissions Guide

  - Overview
  - User Identity
  - Group Mapping
  - Understanding the Implementation
  - Changes to the File System API
  - Changes to the Application Shell
  - The Super-User
  - The Web Server
  - ACLs (Access Control Lists)
  - ACLs File System API
  - ACLs Shell Commands
  - Configuration Parameters
   
   
**Overview**

The Hadoop Distributed File System (HDFS) implements a permissions model for files and directories that shares much of the POSIX model. Each file and directory is associated with an owner and a group. The file or directory has separate permissions for the user that is the owner, for other users that are members of the group, and for all other users. For files, the r permission is required to read the file, and the w permission is required to write or append to the file. For directories, the r permission is required to list the contents of the directory, the w permission is required to create or delete files or directories, and the x permission is required to access a child of the directory.

In contrast to the POSIX model, there are no setuid or setgid bits for files as there is no notion of executable files. For directories, there are no setuid or setgid bits directory as a simplification. The Sticky bit can be set on directories, preventing anyone except the superuser, directory owner or file owner from deleting or moving the files within the directory. Setting the sticky bit for a file has no effect. Collectively, the permissions of a file or directory are its mode. In general, Unix customs for representing and displaying modes will be used, including the use of octal numbers in this description. When a file or directory is created, its owner is the user identity of the client process, and its group is the group of the parent directory (the BSD rule).

HDFS also provides optional support for POSIX ACLs (Access Control Lists) to augment file permissions with finer-grained rules for specific named users or named groups. ACLs are discussed in greater detail later in this document.

Each client process that accesses HDFS has a two-part identity composed of the user name, and groups list. Whenever HDFS must do a permissions check for a file or directory foo accessed by a client process,

- If the user name matches the owner of foo, then the owner permissions are tested;

- Else if the group of foo matches any of member of the groups list, then the group permissions are tested;

- Otherwise the other permissions of foo are tested.
 
If a permissions check fails, the client operation fails.

**User Identity**

As of Hadoop 0.22, Hadoop supports two different modes of operation to determine the user’s identity, specified by the hadoop.security.authentication property:

- simple

In this mode of operation, the identity of a client process is determined by the host operating system. On Unix-like systems, the user name is the equivalent of `whoami`.

- kerberos

In Kerberized operation, the identity of a client process is determined by its Kerberos credentials. For example, in a Kerberized environment, a user may use the kinit utility to obtain a Kerberos ticket-granting-ticket (TGT) and use klist to determine their current principal. When mapping a Kerberos principal to an HDFS username, all components except for the primary are dropped. For example, a principal todd/foobar@CORP.COMPANY.COM will act as the simple username todd on HDFS.

Regardless of the mode of operation, the user identity mechanism is extrinsic to HDFS itself. There is no provision within HDFS for creating user identities, establishing groups, or processing user credentials.

**Group Mapping**

Once a username has been determined as described above, the list of groups is determined by a group mapping service, configured by the hadoop.security.group.mapping property. The default implementation, org.apache.hadoop.security.JniBasedUnixGroupsMappingWithFallback, will determine if the Java Native Interface (JNI) is available. If JNI is available, the implementation will use the API within hadoop to resolve a list of groups for a user. If JNI is not available then the shell implementation, org.apache.hadoop.security.ShellBasedUnixGroupsMapping, is used. This implementation shells out with the bash -c groups command (for a Linux/Unix environment) or the net group command (for a Windows environment) to resolve a list of groups for a user.

An alternate implementation, which connects directly to an LDAP server to resolve the list of groups, is available via org.apache.hadoop.security.LdapGroupsMapping. However, this provider should only be used if the required groups reside exclusively in LDAP, and are not materialized on the Unix servers. More information on configuring the group mapping service is available in the Javadocs.

For HDFS, the mapping of users to groups is performed on the NameNode. Thus, the host system configuration of the NameNode determines the group mappings for the users.

Note that HDFS stores the user and group of a file or directory as strings; there is no conversion from user and group identity numbers as is conventional in Unix.

**Understanding the Implementation**

Each file or directory operation passes the full path name to the name node, and the permissions checks are applied along the path for each operation. The client framework will implicitly associate the user identity with the connection to the name node, reducing the need for changes to the existing client API. It has always been the case that when one operation on a file succeeds, the operation might fail when repeated because the file, or some directory on the path, no longer exists. For instance, when the client first begins reading a file, it makes a first request to the name node to discover the location of the first blocks of the file. A second request made to find additional blocks may fail. On the other hand, deleting a file does not revoke access by a client that already knows the blocks of the file. With the addition of permissions, a client’s access to a file may be withdrawn between requests. Again, changing permissions does not revoke the access of a client that already knows the file’s blocks.

**Changes to the File System API**

All methods that use a path parameter will throw AccessControlException if permission checking fails.

New methods:

- public FSDataOutputStream create(Path f, FsPermission permission, boolean overwrite, int bufferSize, short replication, long blockSize, Progressable progress) throws IOException;

- public boolean mkdirs(Path f, FsPermission permission) throws IOException;

- public void setPermission(Path p, FsPermission permission) throws IOException;

- public void setOwner(Path p, String username, String groupname) throws IOException;

- public FileStatus getFileStatus(Path f) throws IOException;will additionally return the user, group and mode associated with the path.

The mode of a new file or directory is restricted my the umask set as a configuration parameter. When the existing create(path, …) method (without the permission parameter) is used, the mode of the new file is 0666 & ^umask. When the new create(path, permission, …) method (with the permission parameter P) is used, the mode of the new file is P & ^umask & 0666. When a new directory is created with the existing mkdirs(path) method (without the permission parameter), the mode of the new directory is 0777 & ^umask. When the new mkdirs(path, permission) method (with the permission parameter P) is used, the mode of new directory is P & ^umask & 0777.

**Changes to the Application Shell**

New operations:

- chmod [-R] mode file ...

Only the owner of a file or the super-user is permitted to change the mode of a file.

- chgrp [-R] group file ...

The user invoking chgrp must belong to the specified group and be the owner of the file, or be the super-user.

- chown [-R] [owner][:[group]] file ...

The owner of a file may only be altered by a super-user.

- ls file ...

- lsr file ...

The output is reformatted to display the owner, group and mode.

**The Super-User**

The super-user is the user with the same identity as name node process itself. Loosely, if you started the name node, then you are the super-user. The super-user can do anything in that permissions checks never fail for the super-user. There is no persistent notion of who was the super-user; when the name node is started the process identity determines who is the super-user for now. The HDFS super-user does not have to be the super-user of the name node host, nor is it necessary that all clusters have the same super-user. Also, an experimenter running HDFS on a personal workstation, conveniently becomes that installation’s super-user without any configuration.

In addition, the administrator my identify a distinguished group using a configuration parameter. If set, members of this group are also super-users.

**The Web Server**

By default, the identity of the web server is a configuration parameter. That is, the name node has no notion of the identity of the real user, but the web server behaves as if it has the identity (user and groups) of a user chosen by the administrator. Unless the chosen identity matches the super-user, parts of the name space may be inaccessible to the web server.

**ACLs (Access Control Lists)**

In addition to the traditional POSIX permissions model, HDFS also supports POSIX ACLs (Access Control Lists). ACLs are useful for implementing permission requirements that differ from the natural organizational hierarchy of users and groups. An ACL provides a way to set different permissions for specific named users or named groups, not only the file’s owner and the file’s group.

By default, support for ACLs is disabled, and the NameNode disallows creation of ACLs. To enable support for ACLs, set dfs.namenode.acls.enabled to true in the NameNode configuration.

An ACL consists of a set of ACL entries. Each ACL entry names a specific user or group and grants or denies read, write and execute permissions for that specific user or group. For example:
```
   user::rw-
   user:bruce:rwx                  #effective:r--
   group::r-x                      #effective:r--
   group:sales:rwx                 #effective:r--
   mask::r--
   other::r--
```
   
ACL entries consist of a type, an optional name and a permission string. For display purposes, ‘:’ is used as the delimiter between each field. In this example ACL, the file owner has read-write access, the file group has read-execute access and others have read access. So far, this is equivalent to setting the file’s permission bits to 654.

Additionally, there are 2 extended ACL entries for the named user bruce and the named group sales, both granted full access. The mask is a special ACL entry that filters the permissions granted to all named user entries and named group entries, and also the unnamed group entry. In the example, the mask has only read permissions, and we can see that the effective permissions of several ACL entries have been filtered accordingly.

Every ACL must have a mask. If the user doesn’t supply a mask while setting an ACL, then a mask is inserted automatically by calculating the union of permissions on all entries that would be filtered by the mask.

Running chmod on a file that has an ACL actually changes the permissions of the mask. Since the mask acts as a filter, this effectively constrains the permissions of all extended ACL entries instead of changing just the group entry and possibly missing other extended ACL entries.

The model also differentiates between an “access ACL”, which defines the rules to enforce during permission checks, and a “default ACL”, which defines the ACL entries that new child files or sub-directories receive automatically during creation. For example:
```
   user::rwx
   group::r-x
   other::r-x
   default:user::rwx
   default:user:bruce:rwx          #effective:r-x
   default:group::r-x
   default:group:sales:rwx         #effective:r-x
   default:mask::r-x
   default:other::r-x
```
   
Only directories may have a default ACL. When a new file or sub-directory is created, it automatically copies the default ACL of its parent into its own access ACL. A new sub-directory also copies it to its own default ACL. In this way, the default ACL will be copied down through arbitrarily deep levels of the file system tree as new sub-directories get created.

The exact permission values in the new child’s access ACL are subject to filtering by the mode parameter. Considering the default umask of 022, this is typically 755 for new directories and 644 for new files. The mode parameter filters the copied permission values for the unnamed user (file owner), the mask and other. Using this particular example ACL, and creating a new sub-directory with 755 for the mode, this mode filtering has no effect on the final result. However, if we consider creation of a file with 644 for the mode, then mode filtering causes the new file’s ACL to receive read-write for the unnamed user (file owner), read for the mask and read for others. This mask also means that effective permissions for named user bruce and named group sales are only read.

Note that the copy occurs at time of creation of the new file or sub-directory. Subsequent changes to the parent’s default ACL do not change existing children.

The default ACL must have all minimum required ACL entries, including the unnamed user (file owner), unnamed group (file group) and other entries. If the user doesn’t supply one of these entries while setting a default ACL, then the entries are inserted automatically by copying the corresponding permissions from the access ACL, or permission bits if there is no access ACL. The default ACL also must have mask. As described above, if the mask is unspecified, then a mask is inserted automatically by calculating the union of permissions on all entries that would be filtered by the mask.

When considering a file that has an ACL, the algorithm for permission checks changes to:

- If the user name matches the owner of file, then the owner permissions are tested;

- Else if the user name matches the name in one of the named user entries, then these permissions are tested, filtered by the mask permissions;

- Else if the group of file matches any member of the groups list, and if these permissions filtered by the mask grant access, then these permissions are used;

- Else if there is a named group entry matching a member of the groups list, and if these permissions filtered by the mask grant access, then these permissions are used;

- Else if the file group or any named group entry matches a member of the groups list, but access was not granted by any of those permissions, then access is denied;

- Otherwise the other permissions of file are tested.

Best practice is to rely on traditional permission bits to implement most permission requirements, and define a smaller number of ACLs to augment the permission bits with a few exceptional rules. A file with an ACL incurs an additional cost in memory in the NameNode compared to a file that has only permission bits.

**ACLs File System API**

New methods:

- public void modifyAclEntries(Path path, List<AclEntry> aclSpec) throws IOException;
- public void removeAclEntries(Path path, List<AclEntry> aclSpec) throws IOException;
- public void public void removeDefaultAcl(Path path) throws IOException;
- public void removeAcl(Path path) throws IOException;
- public void setAcl(Path path, List<AclEntry> aclSpec) throws IOException;
- public AclStatus getAclStatus(Path path) throws IOException;

**ACLs Shell Commands**
```
hdfs dfs -getfacl [-R] <path>
```

Displays the Access Control Lists (ACLs) of files and directories. If a directory has a default ACL, then getfacl also displays the default ACL.

```
hdfs dfs -setfacl [-R] [-b |-k -m |-x <acl_spec> <path>] |[--set <acl_spec> <path>]
```

Sets Access Control Lists (ACLs) of files and directories.

```
hdfs dfs -ls <args>
```
The output of ls will append a ‘+’ character to the permissions string of any file or directory that has an ACL.

See the File System Shell documentation for full coverage of these commands.

**Configuration Parameters**

```
dfs.permissions.enabled = true
```
If yes use the permissions system as described here. If no, permission checking is turned off, but all other behavior is unchanged. Switching from one parameter value to the other does not change the mode, owner or group of files or directories. Regardless of whether permissions are on or off, chmod, chgrp, chown and setfacl always check permissions. These functions are only useful in the permissions context, and so there is no backwards compatibility issue. Furthermore, this allows administrators to reliably set owners and permissions in advance of turning on regular permissions checking.

```
dfs.web.ugi = webuser,webgroup
```

The user name to be used by the web server. Setting this to the name of the super-user allows any web client to see everything. Changing this to an otherwise unused identity allows web clients to see only those things visible using “other” permissions. Additional groups may be added to the comma-separated list.

```
dfs.permissions.superusergroup = supergroup
```


The name of the group of super-users.

```
fs.permissions.umask-mode = 0022
```


The umask used when creating files and directories. For configuration files, the decimal value 18 may be used.

```
dfs.cluster.administrators = ACL-for-admins
```
The administrators for the cluster specified as an ACL. This controls who can access the default servlets, etc. in the HDFS.

```
dfs.namenode.acls.enabled = true
```
Set to true to enable support for HDFS ACLs (Access Control Lists). By default, ACLs are disabled. When ACLs are disabled, the NameNode rejects all attempts to set an ACL.



#HDFS Permission Guide（中文版）
**概述：**

HDFS实现了一个文件和目录权限模型，拥有很多POXIS模型的影子。每个文件和目录与一个所有者和一个用户组相关联。文件或目录有各自的用户权限，用户包括所有者，所有者同组的其他用户，所有其他的用户。对于文件来说，r权限代表读文件，w权限代表写或者追加数据到文件。对于目录，r表示可以列出目录的内容，w权限代表可以创建或者删除文件或目录，x权限代表可以访问目录的子目录。

与POSIX模型对比，文件没有setuid or setgid的位置，因为没有可执行文件的概念。目录也没有setuid or setgid的位置，以简化设计。Sticky位可以被设置在目录上，以防止除了超级用户，目录所有者和文件所有者的任何人删除或者移动目录内的文件。对一个文件设置Sticky位没有影响。一个文件或者目录都是那种模式。通常，使用Unix风格来代表和显示这个访问模式，包括在描述中使用八进制。当一个文件或者目录被创建，它的所有者是客户端进程的用户标识，它的用户组是父目录的用户组（BSD规则）。

HDFS也对使用POSIX ACL增加特定用户和用户组的细粒度的文件权限提供可选的支持。ACL在之后的文档详细讨论。

每一个访问HDFS的客户端进程包含两部分标识，用户名和组列表（一个用户可以属于多个组）。不管HDFS什么时候对一个被客户端访问的文件或目录foo做权限检查，

1.如果用户名匹配foo的所有者，所有者权限测试通过；

2.否则，如果foo的组匹配组列表中的任意一个，组权限测试通过；

3.否则，foo的其他用户的权限测试通过。

如果权限检测失败，客户端操作失败。

**User Identity**

自Hadoop0.22起，Hadoop支持两个不同的操作模型来判定用户的标识，通过hadoop.security.authentication属性指定：

1.Simple

在这种操作模式中，一个客户进程的标识通过操作系统判定。在类Unix的系统上，用户名等于”whoami”。

2.Kerberos

在使用Kerberos的操作时，一个客户端进程的标识通过它的Kerberos证书来判定。例如，在一个Kerberos的环境中，一个用户可能使用kinit工具获得一个Kerberos TGT，然后用Klist来确定它们当前的实体（Principal）。当映射一个Kerberos Principal到一个HDFS用户名时， all components except for the primary are dropped。例如，一个Principaltodd/foobar@CORP.COMPANY.COM将会作为HDFS上的用户todd。

不管操作模型是哪种，用户认证机制对HDFS来说是外在的。HDFS中没有命令创建一个用户标识，建立一个用户组，或者处理用户的证书。

**Group Mapping**

一旦一个用户名被上述过程确定，文件所属的一个或者多个用户组就会通过hadoop.security.group.mapping属性中配置的用户组映射服务来确定。默认的实现是org.apache.hadoop.security.ShellBasedUnixGroupsMapping，此实现将会使用Unixbash –c 用户组命令来解析一个用户所属的一个或者多个用户组。

另一个实现，直接连接到一个LDAP服务器来解析用户组的列表，可通过配置org.apache.hadoop.security.LdapGroupsMapping使用。但是，LdapGroupsMapping应该只在所需的用户组专门放在LDAP中时才使用，are not materialized on the Unix servers。更多的配置用户组映射服务的信息可在JAVA doc中得到。

对于HDFS来说，用户到用户组的映射在NameNode中实现。因此，NameNode宿主系统的配置决定了用户到用户组的映射。

注意，HDFS将一个文件或者目录的用户和用户组存为一个字符串；不像在Unix中可以从一个用户和用户组标识号转换。

**Understanding the Implementation**

每一个文件或者目录操作都会将全路径传到NameNode，权限检查伴随这个目录的每一个操作。客户端框架将暗中使用到NameNode的连接连接用户标识，减少对已经存在的客户端API的改变。总是存在这样的情况，当对一个文件上的操作成功，再重复执行时这个操作可能会失败，因为路径中的文件或者一些目录，不在存在。例如，当一个客户端开始读一个文件，它发送第一次请求到NameNode以获取文件的第一个Block的位置。获取上下的Block的第二个请求可能会失败。另一方面，删除一个文件不会撤销已经知道这个文件的Block的客户端的访问。通过附加的权限配置（ACL），一个客户端到一个文件的访问权限可能在两次请求间被收回。再说一遍，权限的改变不会撤销已经知道这个文件的Block的客户端的访问。

**Changes to the File System API**

所有使用路径参数的方法在权限检查失败是都会抛出AccessControlException 。

新的方法：
```
public FSDataOutputStream create(Path f, FsPermission permission, boolean overwrite, int bufferSize, short replication, long blockSize, Progressable progress) throws IOException;

public boolean mkdirs(Path f, FsPermission permission) throws IOException;

public void setPermission(Path p, FsPermission permission) throws IOException;

public void setOwner(Path p, String username, String groupname) throws IOException;

public FileStatus getFileStatus(Path f) throws IOException;
```
将会额外的返回用户，用户组的和路径的访问模式。

一个新建立的文件或者目录的访问模式通过umask配置参数设置限制。当使用已经存在的create(path, …)方法（没有权限参数）时，新文件的访问模式时0666&^umask。当使用create(path, permission, …)创建文件（有权限参数）时，新文件的访问模式是P & ^umask & 0666。当一个新目录被已经存在的mkdirs(path)方法（没有权限参数）创建时，新目录的访问模式是0777 & ^umask。当使用新的mkdirs(path, permission)方法（有权限参数）创建一个新的目录时，这个目录的模式是 P & ^umask& 0777。

**Changes to the Application Shell**

新的操作：
```
chmod [-R] mode file…
```
只有文件所有者和超级用户被允许改变文件的访问模式。
```
chgrp [-R] group file…
```
只有属于指定的用户组和文件的所有者或者超级用户可以调用此命令。
```
chown [-R][owner][:[group]] file …
```
一个文件的拥有者只有超级用户可以更改。
```
ls file …
lsr file …
```
输出被重新格式化后显示文件所有者，用户组和访问模式。

**The Super-User**

超级用户是与NameNode进程相同标识的用户。如果你启动NameNode，你就是超级用户。超级用户可以做任何事情，因为对于超级用户来说，权限检查从不失败。没有永久的超级用户的概念；启动NameNode进程的系统用户就是当前的超级用户。HDFS超级用户不需要必须是NameNode宿主系统的超级用户，也没必要所有的集群有相同的超级用户。一个在个人工作站上运行的HDFS，不需要任何设置，便利地变成这个安装实例的超级用户。

此外，可以配置一个管理员用户组。如果配置了，所有这个用户组的用户都是超级用户。

**The Web Server**

默认情况下，Web Server的标识是一个配置参数。也就是说，NameNode没有真实用户的标识的概念，但是Web Server表现就像是有管理员选择的一个用户的标识。除非被选择的用户标识匹配超级用户，部分命名空间可能对Web Server不可访问。

**ACLs（Access Control Lists）**

除了传统了POSIX的权限模型，HDFS也支持POSIX ACLs（Access Control Lists）。ACLs对于实现不同于用户和用户组的自然组织分层结构的权限需求是非常有用的。一个ACL提供一个为特殊的用户和用户组设置不同权限的方式，不仅仅是文件的所有者和文件的所属的用户组。

默认ACLs的支持是关闭的，NameNode不允许ACLs的创建。为了开启ACLs的支持，设置NameNode配置dfs.namenode.acls.enabled为true。

一个ACL包含一系列的ACL Entry。每个ACL Entry包括一个指定的用户或者用户组，授予或者拒绝指定的用户和用户组的读，写和执行权限。例如：
```
[html] view plain copy
user::rw-  
user:bruce:rwx                  #effective:r--  
group::r-x                      #effective:r--  
group:sales:rwx                 #effective:r--  
mask::r--  
other::r-- 
```
一个Entry包含一个类型，一个可选的名字和一个权限字符串。为了显示目的，每一个字段之间用“：”分隔。在这个ACL例子中，文件所有者拥有读写权限，文件用户组有读和可执行权限，其他用户只有读权限。到目前为止，这等于设置文件的权限为654。

此外，还有两个为用户bruce和用户组sales设置的ACL Entry。Mask是特别的ACL条目，它过滤授权给所有命名的用户的Entry，命名的用户组条目和没有命名的用户组条目的权限。在这个例子中，mask只有读权限，我们可以看到几个ACL条目的有效权限已被相应的过滤了。

每一个ACL必须有一个mask。如果用户在设置一个ACL时不提供一个mask，通过计算所有条目的权限的并（数学概念上的并）作为mask自动插入（Mask实际的值实际上就是它过滤的几个权限Entry的最大值）。

在一个有ACL改变了mask的权限的文件上运行chmod命令。因为mask作为过滤器，实际上约束了所有ACL条目的权限而不是仅仅改变了用户组条目和其他的ACL条目的权限。

这个权限模型区分一个“Access ACL”和一个“default ACL”，“Access ACL”定义了在权限检查时强制执行的规则，“default ACl”定义了新的子文件或者子目录在创建期间自动从父目录继承的ACL Entry。例如：

```
[html] view plain copy 
user::rwx  
group::r-x  
other::r-x  
default:user::rwx  
default:user:bruce:rwx          #effective:r-x  
default:group::r-x  
default:group:sales:rwx         #effective:r-x  
default:mask::r-x  
default:other::r-x 
```
只有目录可能有默认的ACL。当一个文件或者一个子目录被创建时，它自动复制它的父目录的default ACL作为它的Access ACL。然后新的子目录复制这个Access ACl 为它的default Access。在这个方式下，当新的子目录被创建时，默认的ACL将被应用到任意深度的文件系统树。

新的子文件或目录的Access ACL的确切的权限值通过访问模式参数被过滤。考虑默认的umask为022，这通常是对目录是755，文件是644。访问模式过滤ACL中没有名字的用户（文件所有者），被mask的用户和其他的用户拷贝的权限值。一个特别的ACL例子，用755访问模式创建一个新的子目录，这个访问模式的过滤对最终结果没有影响。但是，如果我们用644访问模式创建一个文件，这个访问模式的过滤会导致新文件的ACL拥有对ACL中未命名用户（文件所有者）的读写，对mask的用户的读和其他用户的读权限。这个mask也意味着用户bruce和用户组sales有效的权限为只读。

注意，复制发生在新文件或目录创建的时候。随后的到父目录default ACL的更改不会改变已存在的子文件或目录的default ACL。

Default ACl必须有所有ACL条目的最小需求，包括未命名的用户（文件所有者），未命名的组（文件所属组）和其他的Entry。如果用户在配置default ACL时不提供这3个条目中的任何一个，这些条目将会通过从Access ACL复制相应的权限条目（如果没有Access ACL，就复制permission bits）自动被插入。Default ACL也必须有mask。就像上边描述的那样，如果mask没有被指定，将计算所有条目的权限的并作为权限值自动插入。

当判定一个有ACL的文件的权限时，权限检查算法更改为：

1.如果用户名匹配文件所有者，所有者权限测试通过

2.否则，如果用户名匹配命名用户条目中的一个，这些权限测试通过，被mask权限过滤

3.否则，如果文件组匹配任何组列表中的一个，而且，这些被mask过滤的权限授予访问权限，然后，这些权限被使用。

4.否则，如果一个命名的组条目匹配组列表中的一个，而且这些被mask过滤的权限同意访问，这些权限被使用。

5.否则，如果文件组或者任何一个命名的组条目匹配组列表中的一个，但是这些权限没有被授予访问权限，访问被拒绝。

6.否则，文件的其他用户的权限测试通过

最佳实践是依靠传统permission bits来实现大多数权限需求，然后定义少量的ACLs来给permission bits增加额外的规则。相比于只有权限位的文件，一个有ACL的文件导致NameNode中额外的内存消耗。

**ACLs File System API**

新的方法：

```
public void modifyAclEntries(Path path, List<AclEntry> aclSpec) throws IOException;

public void removeAclEntries(Path path, List<AclEntry> aclSpec) throws IOException;

public void public void removeDefaultAcl(Path path) throws IOException;

public void removeAcl(Path path) throws IOException;

public void setAcl(Path path, List<AclEntry> aclSpec) throws IOException;

public AclStatus getAclStatus(Path path) throws IOException;
```
**ACLs Shell Commands**

```
hdfs dfs -getfacl[-R] <path>
```
显示一个文件和目录的ACLs。如果一个目录有一个defaultACL，getfacl也会显示默认的ACL。

```
hdfs dfs -setfacl [-R][-b|-k -m|-x <acl_spec> <path>]|[--set <acl_spec><path>]
```
设置文件和目录的ACLs。
```
hdfs dfs -ls<args>
```
ls的输出将会附加“+”号到任何一个有ACL的权限字符串。

看File System Shell 文档查看这些命令的所有用法。

**Configuration Parameters**
```
dfs.permissions.enabled= true
```
如果你像上边描述的一样使用权限系统。如果不是，权限检查被关闭，但是所有其他的表现都是不变的。从一个参数值切换到另一个，不会改变文件和目录的访问模式，所有者和用户组。不管权限是否关闭，chmod，chgrp和chown总是检查权限。这些功能只在有权限的环境中有用，所以没有向后兼容的问题。而且，这允许管理员在打开常规的权限检查之前可靠的设置所有者和权限。
```
dfs.web.ugi =webuser,webgroup
```
Web Server使用的用户名。设置这个值为超级用户的名字将允许任何Web客户端查看任何东西。将其改为其他的未使用的身份，将允许客户端值看到对其可见的内容。附加的用户组可被增加到逗号分隔的列表。
```
dfs.permissions.superusergroup= supergroup
```
超级用户所在的组。
```
fs.permissions.umask-mode= 0022
```
创建文件或目录时使用的umask值。对于配置文件，可用十进制18代替。
```
dfs.cluster.administrators= ACL-for-admins
```
集群的管理员指定的ACL。这将控制访问HDFS中默认的servlet等。
```
dfs.namenode.acls.enabled= true
```
设置为TRUE以使HDFS支持ACLs。默认情况下，ACLs是禁用的。当ACLs被禁用时，NameNode拒绝所有设置一个ACL的尝试。
