---
layout: post
title:  "2 Determine and Set Sizing Parameters for Database Structures"
date:   2016-08-05 10:48:57 +0000
comments: true
categories: Server_Configuration
---


1. oracle 官方文档 ->  Masters Book List -> Administrator’s Guide -> 2 Creating and Configuring an Oracle Database -> [Specifying Initialization Parameters](http://docs.oracle.com/cd/E11882_01/server.112/e25494/create.htm#CIAFAFFG)

2. 查看和定位数据库的各种参数
		
		登录Oracle
		[oracle@oracle ~]$ sqlplus / as sysdba

		SQL*Plus: Release 11.2.0.3.0 Production on Fri Aug 5 03:11:11 2016
		
		Copyright (c) 1982, 2011, Oracle.  All rights reserved.
		
		Connected to:
		Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
		With the Partitioning, OLAP, Data Mining and Real Application Testing options
		
		查看当前会话的参数
		SQL> show parameter
		
		查看指定的参数
		SQL> show parameter db_recovery
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_recovery_file_dest		     string	 /u01/app/oracle/flash_recovery_area
		db_recovery_file_dest_size	     big integer 12G

		
		这个命令等同于以下查询
		SQL> COL NAME FORMAT A35
		SQL> COL VALUE FORMAT A40
		SQL> SELECT NAME, VALUE FROM V$PARAMETER WHERE LOWER(NAME) LIKE '%db_recovery%';
		NAME				    VALUE
		----------------------------------- ----------------------------------------
		db_recovery_file_dest		    /u01/app/oracle/flash_recovery_area
		db_recovery_file_dest_size	    5368709120
		
		查看启动实例使用的是SPFILE还是PFILE
		可以查看SPFILE的位置，如果是通过PFILE启动的，则为空
		SQL> show parameter spfile
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		spfile				     string	 /u01/app/oracle/product/11.2.0/dbhome_1/dbs/spfileORCL.ora

		如果是用SPFILE启动的，可以用以下方式查询
		SQL> show spparameters db_recovery
		SID	 NAME			       TYPE	   VALUE
		-------- ----------------------------- ----------- ----------------------------
		*	 db_recovery_file_dest	       string	   /u01/app/oracle/flash_recovery_area
		*	 db_recovery_file_dest_size    big integer 5G


		此查询使用的是V$SPPARAMETER，因此可以将此查询改写为以下SQL
		SQL> COL SID FORMAT A5
		SQL> COL NAME FORMAT A30
		SQL> SELECT SID,NAME, VALUE FROM V$SPPARAMETER WHERE LOWER(NAME) LIKE '%db_recovery%';
				
		SID   NAME			     VALUE
		----- ------------------------------ ----------------------------------------
		*     db_recovery_file_dest	     /u01/app/oracle/flash_recovery_area
		*     db_recovery_file_dest_size     5368709120


3. Global Database Name

	全局数据库名称（Global Database Name）由用户指定的本地数据库名称和数据库的网络结构中的位置的。该DB_NAME初始化参数决定数据库名称的本地名称部分，而DB_DOMAIN参数，它是可选的，表示网络结构中的域（逻辑位置）。对于这两个参数的设置的组合必须形成一个数据库名称是在网络内是唯一的。  
		
	DB_NAME Initialization Parameter

		DB_NAME must be set to a text string of no more than eight characters. During database creation, the name provided for DB_NAME is recorded in the data files, redo log files, and control file of the database. If during database instance startup the value of the DB_NAME parameter (in the parameter file) and the database name in the control file are different, the database does not start.
	
	DB_DOMAIN Initialization Parameter

		DB_DOMAIN is a text string that specifies the network domain where the database is created. If the database you are about to create will ever be part of a distributed database system, then give special attention to this initialization parameter before database creation. This parameter is optional.
	
    查看db_name和db_domain的值
		
		SQL> show parameter db_name
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_name 			     string	 ORCL

		SQL> show parameter db_domain		
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_domain			     string	 oracle.com

		GLOBAL_DB_NAME等于db_name.db_domain
		SQL> COL PROPERTY_VALUE FORMAT A40
		SQL> SELECT PROPERTY_NAME, PROPERTY_VALUE FROM DATABASE_PROPERTIES WHERE PROPERTY_NAME = 'GLOBAL_DB_NAME';
		PROPERTY_NAME		       PROPERTY_VALUE
		------------------------------ ----------------------------------------
		GLOBAL_DB_NAME		       ORCL.ORACLE.COM


4. 指定一个Fast Recovery Area

	Fast Recovery Area是oracle DB存储并管理备份和恢复相关文件的位置。它与数据库中当前数据库文件（数据文件，控制文件和redo log）的位置不同。
	
	指定Fast Recovery Area 主要用到以下两个参数：

		DB_RECOVERY_FILE_DEST: Location of the Fast Recovery Area. This can be a directory, file system, or 
		Automatic Storage Management (Oracle ASM) disk group. It cannot be a raw file system.

	在RAC环境中，这个位置必须在集群文件系统中，OracleASM磁盘组，或者通过NFS的共享目录

		DB_RECOVERY_FILE_DEST_SIZE: Specifies the maximum total bytes to be used by the Fast Recovery Area. This initialization parameter must be specified before DB_RECOVERY_FILE_DEST is enabled. 
	在RAC环境下面，所有实例下面的这两个参数的设置必须一样
	如果为LOG_ARCHIVE_DEST和LOG_ARCHIVE_DUPLEX_DEST参数的设值，你不能启用这些参数。建立Fast Recovery Area之前，您必须禁用这些参数。您可以用LOG_ARCHIVE_DEST_n参数的值替代。如果本地存档位置尚未配置和LOG_ARCHIVE_DEST_1价值尚未设置则LOG_ARCHIVE_DEST_1参数被隐式设置指向快速恢复区。
	Oracle建议使用快速恢复区，因为它可以简化备份和恢复操作的数据库。
		
		SQL> show parameter db_recovery
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_recovery_file_dest		     string	 /u01/app/oracle/flash_recovery_area
		db_recovery_file_dest_size	     big integer 12G
		
		修改这些参数不需要重启实例
		SQL> alter system set db_recovery_file_dest_size=5G scope=both;
		System altered.

		查看变化
		SQL> show parameter db_recovery_file_dest_size
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_recovery_file_dest_size	     big integer 5G
		
		Oracle自动管理这个位置的使用情况
		查看占用情况
		SQL> SELECT * FROM V$FLASH_RECOVERY_AREA_USAGE;

		FILE_TYPE	     	 PERCENT_SPACE_USED PERCENT_SPACE_RECLAIMABLE NUMBER_OF_FILES
		-------------------- ------------------ ------------------------- ---------------
		CONTROL FILE		0		0		0
		REDO LOG		0		0		0
		ARCHIVED LOG		0.69		0.63		8
		BACKUP PIECE		19.79		8.31		12
		IMAGE COPY		0		0		0
		FLASHBACK LOG		3.91		0		2
		FOREIGN ARCHIVED LOG		0		0		0

5. 指定控制文件
	
	参数文件中CONTROL_FILES定义一个或多个控制文件的名称和位置，在执行 CREATE DATABASE语句的时候，CONTROL_FILES后面列出的控制文件将会被创建。
	
	如果初始化参数文件不包括CONTROL_FILES，那么Oracle数据库会在初始化参数文件同一个目录的控制文件，使用默认的操作系统有关的文件名。如果已经启用了OMF，数据库将会创建的Oracle管理的控制文件。

	如果你想在数据库中创建数据库的控制文件时，在CONTROL_FILES参数中列出的文件名不是必须匹配当前系统中已经存在的文件名。如果你希望数据库重用或覆盖数据库的控制文件时，确保在CONTROL_FILES参数中列出的文件名和需要重复使用的文件名相同的，并包含在CREATE DATABASE语句的CONTROLFILE REUSE子句中。
	
	Oracle强烈建议您在每个数据库至少有两个控制文件，且存储在单独的磁盘上。
		
		查看控制文件
		SQL> show parameter control_files
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		control_files			     string	 /u01/app/oracle/oradata/orcl/control01.ctl,/u01/app/oracle/oradata/orcl/control02.ctl

		也可以通过视图V$CONTROLFILE查看
		SQL> select * from V$CONTROLFILE;
		STATUS	NAME			       IS_ BLOCK_SIZE FILE_SIZE_BLKS
		------- ------------------------------ --- ---------- --------------
			/u01/app/oracle/oradata/orcl/c NO	16384		 580
			ontrol01.ctl
		
			/u01/app/oracle/oradata/orcl/c NO	16384		 580
			ontrol02.ctl


		增加一个控制文件的镜像
		1. 关闭数据库
		SQL> shutdown immediate
		Database closed.
		Database dismounted.
		ORACLE instance shut down.
		2. 将控制文件拷贝到一个新的位置
		[oracle@oracle ~]$ cp /u01/app/oracle/oradata/orcl/control01.ctl /u01/control03.ctl
		3. 将数据库启动到 NOMOUNT 状态，然后修改参数 control_files
		SQL> startup nomount;
		ORACLE instance started.
		
		Total System Global Area  417546240 bytes
		Fixed Size		    2228944 bytes
		Variable Size		  327159088 bytes
		Database Buffers	   83886080 bytes
		Redo Buffers		    4272128 bytes
		SQL> ALTER SYSTEM SET CONTROL_FILES='/u01/app/oracle/oradata/orcl/control01.ctl','/u01/app/oracle/oradata/orcl/control02.ctl','/u01/control03.ctl' SCOPE=SPFILE;
		
		System altered.

		4. 启动数据库，然后查看变化
		SQL> alter database mount; 

		Database altered.
		
		SQL> alter database open;
		
		Database altered.
		
		SQL> show parameter control_files;
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		control_files			     string	 /u01/app/oracle/oradata/orcl/control01.ctl, /u01/app/oracle/oradata/orcl/control02.ctl

		立即启动以后，没有任何变化，重启数据库，重新读取参数文件
		SQL> shutdown immediate
		Database closed.
		Database dismounted.
		ORACLE instance shut down.
		
		再次重启，发现两处的控制文件已经不一致了，无法启动
		SQL> STARTUP
		ORACLE instance started.
		
		Total System Global Area  417546240 bytes
		Fixed Size		    2228944 bytes
		Variable Size		  327159088 bytes
		Database Buffers	   83886080 bytes
		Redo Buffers		    4272128 bytes
		ORA-00214: control file '/u01/app/oracle/oradata/orcl/control01.ctl' version
		1125 inconsistent with file '/u01/control03.ctl' version 1105
		
		SQL> shutdown immediate
		ORA-01507: database not mounted
		
		
		ORACLE instance shut down.
		关闭数据库后，重新拷贝控制文件，再次打开
		SQL> startup 
		ORACLE instance started.
		
		Total System Global Area  417546240 bytes
		Fixed Size		    2228944 bytes
		Variable Size		  327159088 bytes
		Database Buffers	   83886080 bytes
		Redo Buffers		    4272128 bytes
		Database mounted.
		Database opened.
		SQL> show parameter control_files;
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		control_files			     string	 /u01/app/oracle/oradata/orcl/control01.ctl, /u01/app/oracle/
								 				oradata/orcl/control02.ctl, /u01/control03.ctl


		5. 删除新的控制文件
		SQL> shutdown immediate   
		Database closed.
		Database dismounted.
		ORACLE instance shut down.
		SQL> startup nomount
		ORACLE instance started.
		
		Total System Global Area  417546240 bytes
		Fixed Size		    2228944 bytes
		Variable Size		  318770480 bytes
		Database Buffers	   92274688 bytes
		Redo Buffers		    4272128 bytes
		SQL> alter system set control_files='/u01/app/oracle/oradata/orcl/control01.ctl','/u01/app/oracle/oradata/orcl/control02.ctl' scope=spfile;
		
		System altered.

		6. 重启数据库
		SQL> shutdown immediate
		ORA-01507: database not mounted
		
		
		ORACLE instance shut down.
		SQL> startup
		ORACLE instance started.
		
		Total System Global Area  417546240 bytes
		Fixed Size		    2228944 bytes
		Variable Size		  318770480 bytes
		Database Buffers	   92274688 bytes
		Redo Buffers		    4272128 bytes
		Database mounted.
		Database opened.
		SQL> show parameter control_files;
		
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		control_files			     string	 /u01/app/oracle/oradata/orcl/control01.ctl, /u01/app/oracle/oradata/orcl/control02.ctl

6. 数据库块大小

	DB_BLOCK_SIZE 初始化参数指定数据库的标准的块大小。是系统表空间和其他表空间的默认值。Oracle 数据库可以支持最多四个额外的非标准块大小。

	最常用的块大小应该选为标准的块大小。在许多情况下，这是你必须指定唯一的块大小。通常情况下，DB_BLOCK_SIZE 设置为 4k 或 8k。如果不设置此参数的值，然后默认数据块大小是操作系统指定，大体上已足够。

	创建数据库后不能更改的块大小，除了通过重新创建数据库。如果数据库块大小的不同操作系统块大小，那就要确保数据库块大小是操作系统块大小的倍数。例如，如果您的操作系统的块大小是 2 K （2048 字节），DB_BLOCK_SIZE 初始化参数可以设置为 4096.
		
		查看 DB_BLOCK_SIZE 的值
		SQL> show parameter db_block_size
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_block_size			     integer	 8192
		
		使用不同的块大小创建表空间，可以用于：
		1. 在不同块大小的数据库之前传输表空间
		2. 可以用于数据仓库的事实表
		
		1. 创建16K的 CACHE BUFFER
		SQL> show parameter DB_16
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_16k_cache_size		     big integer 0
		SQL> alter system set db_16k_cache_size=50M scope=both;

		System altered.
		
		SQL> show parameter db_16			
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		db_16k_cache_size		     big integer 52M
		额。。。。为什么会变成52M。。。
		2. 创建一个测试数据仓库用的表空间，
		用52M创建成功了
		SQL>CREATE TABLESPACE TESTDW DATAFILE '/u01/app/oracle/oradata/orcl/testdw01.dbf' SIZE 52M BLOCKSIZE 16K;

		Tablespace created.

		SQL> SELECT TABLESPACE_NAME, BLOCK_SIZE FROM DBA_TABLESPACES;
		TABLESPACE_NAME 	       BLOCK_SIZE
		------------------------------ ----------
		SYSTEM				     8192
		SYSAUX				     8192
		UNDOTBS1			     8192
		TEMPTS1 			     8192
		USERS				     8192
		EXAMPLE 			     8192
		TESTDW				    16384
		3. 删除测试表空间
		SQL> drop tablespace TESTDW including contents and datafiles;
		drop tablespace TESTDW including contents and datafiles
		*
		ERROR at line 1:
		ORA-38881: Cannot drop tablespace TESTDW on primary database due to guaranteed
		restore points.
		
		涉及到可靠还原点，需要先删除restore point才能删除表空间
		Database: 11g Release 2
		Error code: ORA-38881
		Description: Cannot drop tablespace string on primary database due to guaranteed restore points.
		Cause: An attempt was made to drop a tablespace on a primary database while there are guaranteed restore points. You cannot do this because Flashback database cannot undo dropping of a tablespace.
		Action: Drop all guaranteed restore points first and retry, or delay dropping the tablespace until all guaranteed restore points are removed.

		SQL> SELECT NAME, SCN, TIME, DATABASE_INCARNATION#,GUARANTEE_FLASHBACK_DATABASE,STORAGE_SIZE FROM V$RESTORE_POINT; 
		NAME
		--------------------------------------------------------------------------------
		       SCN
		----------
		TIME
		---------------------------------------------------------------------------
		DATABASE_INCARNATION# GUA STORAGE_SIZE
		--------------------- --- ------------
		BEFORE_TRUNCATE
		    436709
		24-MAR-16 06.52.25.000000000 AM
				    2 YES    314572800
		
		SQL> drop restore point BEFORE_TRUNCATE;
		Restore point dropped.
		
		SQL> DROP TABLESPACE TESTDW INCLUDING CONTENTS AND DATAFILES;
		Tablespace dropped.
		
		恢复初始化参数
		SQL> alter system set db_16k_cache_size=0 scope=both;
		System altered.
		
		或者
		SQL> alter system set db_16k_cache_size=0 scope=memory;
		System altered.

		SQL> alter system reset db_16k_cache_size scope=spfile;
		System altered.

7. PROCESSES
	 1. PROCESSES决定当前Oracle数据库可以连接的最大用户数。
	 2. 这个参数的值必须是至少每个后台进程算一个，再加上用户进程的数量
	 3. 后台进程是数量根据数据库的功能变化
	 
	 DBCA创建数据库的时候，默认是150
		
		查看当前进程数量
		SQL> show parameter processes
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		aq_tm_processes 		     integer	 1
		db_writer_processes		     integer	 1
		gcs_server_processes		     integer	 0
		global_txn_processes		     integer	 1
		job_queue_processes		     integer	 1000
		log_archive_max_processes	     integer	 4
		processes			     integer	 150
		
		将进程数量调整到200
		SQL> alter system set processes=200 scope=spfile;
		System altered.
		重启后生效，查看值
		SQL> startup force
		ORACLE instance started.
		
		Total System Global Area  417546240 bytes
		Fixed Size		    2228944 bytes
		Variable Size		  310381872 bytes
		Database Buffers	  100663296 bytes
		Redo Buffers		    4272128 bytes
		Database mounted.
		Database opened.
		SQL> show parameter processes
		NAME				     TYPE	 VALUE
		------------------------------------ ----------- ------------------------------
		aq_tm_processes 		     integer	 1
		db_writer_processes		     integer	 1
		gcs_server_processes		     integer	 0
		global_txn_processes		     integer	 1
		job_queue_processes		     integer	 1000
		log_archive_max_processes	     integer	 4
		processes			     integer	 200

		
8. DDL Lock Timeout