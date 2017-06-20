## MySQL 备份：Percona XtraBackup的原理与实践

[TOC]

### 备份背景及类型

#### 按照备份方式划分

- 热备份

  - 在数据库运行过程中进行备份，对生产环境中的数据库运行没有任何影响。
    - mysqldump、xtrabackup等工具

- 冷备份

  - 在数据库关闭的情况下进行备份，这种备份非常简单，只需要关闭数据库，复制相关的物理文件即可。
  - 很少使用。

- 温备份

  - 也是在数据库运行的过程中进行备份，但是备份会对数据库操作有所影响。

  - 该备份利用锁表的原理备份数据库

  - 很少使用


#### 根据备份文件的种类


-   物理备份

    - 指复制数据库的物理文件。物理备份即可以在数据库运行的情况下进行备份（常见备份工具：MySQL Enterprise Backup、XtraBackup等），也可以在数据库关闭的情况下进行备份。
    - 备份速度快，恢复速度也很快。
    - 无法查看备份后的内容，只能等到恢复以后，才能校验备份出来的数据是否是正确的。
-   逻辑备份

    -   指备份文件的内容是可读的，该文本一般都是由一条条SQL语句或者表的实际数据组成。常见的逻辑备份方式由mysqldump、select * into outfile等方法。
    -   可以观察备份后的文件内容
    -   恢复时间很长

#### 根据备份内容

- 全量备份
  - 指对数据库进行一次完整的备份，备份所有的数据。
  - 可以用来快速恢复数据库，或者搭建从库。
  - 恢复时间最快。
  - 占用很多的磁盘空间，且备份时间较长。
- 增量备份
  - 指基于上一次完整备份或增量备份，对数据库新增的修改进行备份。
  - 备份时间短。
  - 恢复速度较慢，且操作相对复杂。
- 日志备份
  - 指对MySQL数据库的二进制日志的备份。
  - 该备份一般与上面的全量备份或增量备份结合使用，可以使数据库恢复到任意位置。



> 备份一般需要满足以下三个需求：
>
> ​	备份时，数据库需要提供服务，即需要热备份
>
> ​	生产环境中，需要随时恢复到某个时间点，以便于线上出现问题后的数据回滚。
>
> ​	可以用来快速搭建新的从库，建立复制关系，实现数据的高可用、负载均衡的目标。

​	一般都会选择物理备份为主，逻辑备份为辅，加上日志备份来满足线上使用数据库的需求。



### 认识Percona XtraBackup

XtraBackup是由Percona公司开发的一款开元的MySQL热备工具，在备份的过程中不会被所在数据库中。该工具能够备份InnoDB、XtraDb、MyISAM表，甚至可以备份Percona XtraDB Cluster。

支持Percona Server的所有版本，支持MySQL、支持MariaDB的备份。

#### 优点

- 无需停止数据库进行InnoDB热备。
- 增量备份MySQL
- 流压缩传输到其他服务器（异地备份）
- 在线移动表
- 能够比较容易地创建主从同步
- 备份MySQL时不会增大服务器负载

在这套工具集里最重要的程序有两个：innobackupex与xtrabackup，前者是perl脚本，后者是C/C++编译的二进制程序。

xtrabackup是用来备份InnoDB的，并不能备份非InnoDB表。它在内部实现了对InnoDB的热备份。innobackipex脚本是对xtrabackup的封装，通过调用xtrabackup命令来备份InnoDB表，同时也通过mysqldump等命令来备份非InnoDB表，并且会与MySQL数据库发送命令交互，例如获取Binlog位点、添加锁操作。

### XtraBackup的工作流程

在工作过程中，innobackupex会和xtrabackup协同工作，共同完成备份任务。

- innobackupex启动后，首先通过start_ibbackup()函数中fork方式创建xtrabackup进程并且启动。然后等待xtrabackup完成InnoDB相关文件的备份。
- Innobackup通过wait_for_ibbackup_suspend($suspend_file)检测`xtrabackup_suspended_2`文件是否存在，如果检测到该文件，innobackupex进程就会被唤醒。



- xtrabackup在备份InnoDB相关文件时，会开启如下两种线程
  - ibd复制线程，负责不知ibd文件
  - redo log复制线程，负责复制redo log信息。xtrabackup首先启动redo log复制线程，从最近的checkpoint点开始顺序复制redo log；然后启动ibd复制线程，在xtrabackup复制ibd的过程中，redo log复制线程一直工作，并且innobackupex进程处于等待状态，等待被xtrabackup进程唤醒
- xtrabackup复制ibd完成后，通知innobackupex进程（通过创建`xtrabackup_suspended_2`文件方式），同时自己进入等待状态，但是redo log复制线程依然在工作。
- innobackupex收到通知，会执行备份锁（LOCK TABLE FOR BACKUP），取到一致性的位点，然后开始复制非InnoDB文件。
- 当非InnDBm文件复制完成后，innobackupex开始执行LOCK BINLOG FOR BACKUP 开始通过 gat_mysql_master_status($con)函数来获取Binlog位置。
- 然后创建xtrabackup_binglog_info文件，通过write_to_backup_file($binlog_info,join("\t",@binlog_info_coneten)."\n")函数讲Binlog的位点信息写入文件中
- 接着发起一个通知给xtrabackup进程（通过删除xtrabackup_suspended_2文件的方式）同时自己进入等待状态。
- xtrabackup收到通知后，就会停止redo log复制线程，告知innobackupex， redo log复制完成，然后通知innobackupex（通过创建xtrabakcup_log_copied文件的方式）
- innobackupex收到通知后，就开始释放锁资源（UNLOCK BINLOG和UNLOCK TABLES）
- 随后，innobackupex和xtrabackup进行后期工作，比如资源的释放、备份元数据信息、打印备份目录、备份binlog的位置信息，以及写入xtrabackup_info文件信息等。
- 最后，innobackupex等待xtrabakcup进程结束后退出。

#### 总结

- 在备份过程中，需要复制redo log，是由于备份ibd文件的过程中该文件可能被修改，这样备份出来的文件就可能由脏数据。那么在恢复时，就需要通过redo log进行数据恢复，即应用已经提交的事物，回滚那些未提交的事物。
- 在备份过程中严重依赖两个通信文件，如果人为删除或创建了这些文件当中的任何一个文件，都会导致备份出现错误。
- 如果是增量复制，还存在xtrabackup_suspended_1文件，作用是一样。