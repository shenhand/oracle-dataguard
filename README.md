oracle dataguard 
目錄
1.	目的與預期優點	5
2.	適用範圍	6
3.	Introducing Data Guard	7
3.1	Oracle Data Guard Architecture	7
3.2	Type of Standby Databases	7
3.3	Data Protection Modes	8
4.	Create physical standby database using Oracle DataGuard	9
4.1	環境資訊	9
4.2	Prerequisites	9
4.2.1	Primary Database is in ARCHIVELOG-Mode	9
4.2.2	FORCE LOGGING is enabled on the Primary Database	11
4.3	Preparing the Environment	12
4.3.1	建立TNS-Alias on Primary	12
4.3.2	建立所需目錄於Standby Server	12
4.3.3	Copy SPFILE from Primary to Standby	13
4.4	備份Primary資料庫	14
4.5	備份controlfile for standby	15
4.6	傳送備份至Standby Server	15
4.7	restore database on standby	15
4.7.1	startup nomount on standby	15
4.7.2	設定DBID	16
4.7.3	restore standby controlfile	16
4.7.4	mount database	17
4.7.5	restore database	17
4.7.6	recover database	18
4.7.7	Create Standby logfile	19
4.8	修改及啟動listener on standby server	19
4.9	調整Primary Database參數	21
4.10	啟動real-time apply	21
4.10.1	啟動real-time apply	21
4.10.2	Primary Database switch logfile	22
4.10.3	確認log傳送的狀態	22
4.10.4	確認Data Guard Process狀態	22
4.10.5	開啟檔案自動管理	23
4.11	停止Standby database apply	24
5.	啟動Cold Standby Database	25
5.1	Primary Database Status確認	25
5.2	確認Archive log都傳送到Standby	26
5.3	確認archive log no gaps	26
5.4	Stop Redo Apply	27
5.5	完成Apply所有redo data	27
5.6	確認Standby Database可切換成Primary Database	27
5.7	將standby切換為primary	28
5.8	開啟資料庫	28
6.	Appendix – Trouble Shooting	29
6.1	新增datafile 時standby FILE_MANAGEMENT未開啟造成同步失敗	29
6.2	primary的log無法傳送到standby(其一)	32
6.3	若發現dataguard有gap為備份檔案過舊，請重新備份一次	35
6.4	清除MV 的安全作法(避免過大塞爆Arch log)	36
6.5	同步的監控優化作法	38
6.6	Active Dataguard作法	39
    6.7    Standby Archive log Housekeeping機制……………………………………..……...40

 

1.	目的與預期優點
資料庫在加值服務扮演著重要的角色，加值服務的提供往往與資料庫有緊密的依存性，若資料庫異常，將導致服務無法正常提供，而若資料庫嚴重損毀，復原的時間往往較為漫長，進而影響服務長時間無法使用。本SOP的目的在於透過SOP來建置Cold Standby Database，以預防資料庫損毀，可縮短資料庫回復所需時間，並能減少服務影響時間，讓Standby database可接手提供服務，提升資料庫可用性。
 

2.	適用範圍
本文件適用於Oracle 10g/11g資料庫，以使用Oracle Data Guard建置Cold Standby Database；因內容包含Oracle基礎概念及RMAN Backup & Recovery的相關操作，請恕在本文件無加以贅述。
 

3.	Introducing Data Guard
Oracle Data Guard Architecture
 

Type of Standby Databases
Oracle Standby Database 有下列三種型態，本文件所建置的為Physical standby database。
	Physical standby database
	Logical standby database
	Snapshot standby database
Data Protection Modes
Oracle Data Guard 資料保護有下列三種模式，本文件所建置的Standby database為Maximum performance。
	Maximum protection
	Maximum availability
	Maximum performance
 

4.	Create physical standby database using Oracle DataGuard
環境資訊
OS：RHEL 5.6
Database Version：11.2.0.3
Primary hostname/IP：Primary/192.168.100.88
Standby hostname/IP：Standby/192.168.100.89
SID/DB_UNIQUE_NAME ( Primary and Standby )：primary
Listener Port ( Primary and Standby )：1521
Database Files Location ( Primary and Standby )： /u01/oradata/primary
Database archive log file location ( Primary and Standby )： /u01/oradata/primary/archive
Prerequisites
4.1.1	Primary Database is in ARCHIVELOG-Mode
4.1.1.1	檢查資料庫歸檔模式
使用sqlplus 以sysdba登入，執行下列語法檢查資料庫是否為Archivelog mode。
SQL> SELECT LOG_MODE FROM V$DATABASE;
 	
4.1.1.2	開啟Archive log mode
若上一步驟執行結果為NOARCHIVELOG，則透過下列步驟開啟Archive log mode，若執行結果為ARCHIVELOG，則跳過此步驟。
1.	設定archive log destination
使用sqlplus 以sysdba登入，執行下列語法設定archive log destination，紅字部份請視環境進行調整。

SQL> alter system set log_archive_dest_1='LOCATION=/u01/oradata/primary/archive' scope=both;
 
2.	重啟資料庫於mount
SQL> shutdown immeidate;
 
SQL> startup mount;
 
3.	開啟archive log mode
SQL> alter database archivelog;
 
4.	確認資料庫歸檔模式
 
4.1.2	FORCE LOGGING is enabled on the Primary Database
4.1.2.1	檢查資料庫是否啟用force logging
使用sqlplus 以sysdba登入，執行下列語法檢查資料庫是否啟用force logging。

SQL> select force_logging from v$database;
 
4.1.2.2	enable force logging
若上一步驟執行結果為”NO”，則使用sqlplus 以sysdba登入，執行下列語法enable force logging。

SQL> alter database force logging;
 
4.1.2.3	確認資料庫已啟用force logging
SQL> select force_logging from v$database;
 
Preparing the Environment
4.1.3	建立TNS-Alias on Primary
新增下列內容於tnsnames.ora，紅字部份需視環境調整。

STANDBY =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.100.89)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SID = PRIMARY)
    )
  )
4.1.4	建立所需目錄於Standby Server
使用下列指令建立Standby Server所需目錄。
[oracle@Standby admin]$ cd $ORACLE_BASE/admin
[oracle@Standby admin]$ mkdir primary
[oracle@Standby admin]$ cd primary/
[oracle@Standby primary]$ mkdir adump dpdump
 
使用下列指令建立Standby Server放置datafile and archive log 路徑，紅字部份請依環境進行調整。
 [oracle@Standby oradata]$ mkdir -p /u01/oradata/primary/archive
 
使用root權限建立rman備份所需目錄，紅字部份請依環境進行調整。
[root@Standby ~]# mkdir /rmanbackup
[root@Standby ~]# chown oracle:dba /rmanbackup/
4.1.5	Copy SPFILE from Primary to Standby
使用scp指令將Primary Database的spfile和password file傳送到Standby Server。

[oracle@Primary dbs]$ cd $ORACLE_HOME/dbs
[oracle@Primary dbs]$ scp spfileprimary.ora orapwprimary oracle@192.168.100.89:$ORACLE_HOME/dbs/.

 

於Standby Server確認檔案已收到
 

備份Primary資料庫
使用下列rman指令執行Primary資料庫備份，紅字部份請依環境進行調整。

[oracle@Primary dbs]$ rman target / nocatalog

RMAN> backup database format '/rman2/backup/PRMY_%U.bk' plus archivelog format '/rman2/backup/PRMY_%U.bk';
 
 

備份controlfile for standby
使用下列rman指令執行Primary資料庫control file備份，紅字部份請依環境進行調整。

RMAN> backup current controlfile for standby format '/rmanbackup/backup/standby.ctl';
 
傳送備份至Standby Server
使用scp指令將Primary Database的備份傳送到Standby Server，紅字部份請依環境調整。

[oracle@Primary rmanbackup]$ scp /rmanbackup/* oracle@192.168.100.89:/rmanbackup/.
 

restore database on standby
4.1.6	startup nomount on standby
使用下列步驟透過rman將資料庫開在nomount狀態，紅字部份請依環境調整。

[oracle@Standby rmanbackup]$ export ORACLE_SID=primary
[oracle@Standby rmanbackup]$ rman target / nocatalog
RMAN> startup nomount;
 
4.1.7	設定DBID
使用下列指令設定DBID，DBID可由2.4 備份Primary資料庫時得知，紅字部份請依環境進行調整。

RMAN> set dbid=1678890199
 

4.1.8	restore standby controlfile
使用下列指令restore standby controlfile，紅字部份請依環境進行調整。

RMAN> restore standby controlfile from '/rmanbackup/standby.ctl';
 

4.1.9	mount database
使用下列語法mount database。

RMAN> alter database mount;
 

4.1.10	restore database
使用下列指令執行restore database。

RMAN> restore database;
 
4.1.11	recover database
使用下列指令執行recover database，底下出現的Error Message表示需要sequence=24的archive log，但因2.4備份Primary Database時，當時archive log只到sequence 23，故此Error Message為正常的。

RMAN> recover database;
 
4.1.12	Create Standby logfile
先使用下列語法查詢，Primary有幾個online redo log，以及online redo log file的size。

SQL> select group#,bytes from v$log;
 

使用下列語法新增stnadby logfile，所需standby logfile數量為Primary logfile 數+1，size大小與Primary一致，因上一步驟查詢Primary logfile數為3，size大小為50M，故需新增standby logfile數量為4，size大小為50M，紅字部份請依環境進行調整。

SQL> alter database add standby logfile '/u01/oradata/primary/redo01_std.log' size 50M;
SQL> alter database add standby logfile '/u01/oradata/primary/redo02_std.log' size 50M;
SQL> alter database add standby logfile '/u01/oradata/primary/redo03_std.log' size 50M;
SQL> alter database add standby logfile '/u01/oradata/primary/redo04_std.log' size 50M;
 
修改及啟動listener on standby server

先修改oracle   listener.ora檔案

[oracle@SDMPDBN01 admin]$ vim listener.ora
SID_LIST_SDMPLSNR =
  (SID_LIST =
    (SID_DESC =
      (ORACLE_HOME = /oracle/app/oracle/product/11.2.0.3 )
      (SID_NAME = sdmp)
    )
  )

SDMPLSNR =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 172.30.11.9)(PORT = 1521))
    )
  )
 [oracle@Standby ~]$ lsnrctl start listener_name
 
調整Primary Database參數
執行下列語法，指定archive log傳送至Cold Stnadby不使用壓縮傳輸，紅字部份請依各環境不同進行調整。

SQL> alter system set log_archive_dest_2='SERVICE=STANDBY compression=disable ' scope=both;
 
啟動real-time apply
4.1.13	啟動real-time apply
使用下列語法於standby database執行，啟動real-time apply。

SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
 
4.1.14	Primary Database switch logfile
使用下列語法於Primary Database switch logfile
SQL> alter system switch logfile;
 

4.1.15	確認log傳送的狀態
使用下列語法於Primary Database確認Standby Database apply機制是否為real-time apply，紅字部份請依各環境不同進行調整。

SQL> SELECT DEST_NAME,STATUS,TYPE,RECOVERY_MODE,GAP_STATUS FROM V$ARCHIVE_DEST_STATUS WHERE DEST_ID=2;
 

4.1.16	確認Data Guard Process狀態
使用下列語法於Primary Database查詢，確認LNS Process的狀態為WRITING，表示正在傳送redo log至Standby Database。

SQL> SELECT PROCESS,STATUS,CLIENT_PROCESS,SEQUENCE#, BLOCK#,BLOCKS FROM V$MANAGED_STANDBY;
 

使用下列語法於Standby Database查詢，確認MRP Process的狀態為APPLYING_LOG，表示目前正在同步，並且SEQUENCE的值與Primary database相同。

SQL> SELECT PROCESS,STATUS,CLIENT_PROCESS,SEQUENCE#, BLOCK#,BLOCKS FROM V$MANAGED_STANDBY;
 
4.1.17	開啟檔案自動管理
SQL> alter system set standby_file_management = auto;

當primary使用增加datafile以擴充tablespace空間時，standby預設值是不會同步此動作的(standby_file_management = MANUAL)，這時就必須以人工方式，於standby重複primary執行的add datafile指令。為避免上述loading，將standby的DB參數standby_file_management設為auto，standby就會接收primary增加datafile的指令，自動增加對應的datafile。
 
停止Standby database apply
使用下列語法於standby database執行，停止同步機制。

SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
 
 

5.	啟動Cold Standby Database

Primary Database Status確認
實務上啟動Cold Standby Database的時機可能有以下幾種:
	計劃性:
	例行性災難演練
	資料庫Migration
	非計畫性:
	發生災難的資料庫切換

    若是透過Data Guard來做資料庫Migration時,在啟動Cold Standby Database之前,需在Primary Database端,確認以下幾點,以確保切換後的資料ㄧ致性與完整性,以及Recplication在切換至Cold standby Database後,能持續正常Refresh
	停止Listener
Lsnrctl stop

	停止所有Replication的refresh
SQL>select * from dba_jobs;
SQL>EXEC DBMS_JOB.BROKEN(<JOB_ID>,TRUE);

P.S. 完成migration後,請執行以下指令,重新啟動replication refresh
SQL>EXEC DBMS_JOB.BROKEN(<JOB_ID>,FALSE);

	所有AP User的連線皆已中斷
SQL>select * from v$sessioin;

	執行Alter system switch logfile後shutdown db
SQL>alter system switch logfile;
SQL>shutdown immediate;
確認Archive log都傳送到Standby
使用下列語法於standby database執行，確認Primary Database Archive log 都傳送到Standby Database。

SQL> SELECT UNIQUE THREAD# AS THREAD, MAX(SEQUENCE#) OVER (PARTITION BY thread#) AS LAST from V$ARCHIVED_LOG;
 
如果發現Standby Database沒有收到Primary Database 所有Archive log，則將有差異的archive log 傳送到Standby Database，並執行以下指令手動register archive log，紅色部份請依環境進行調整。
	
SQL> ALTER DATABASE REGISTER PHYSICAL LOGFILE '檔案路徑';

確認archive log no gaps

SQL> SELECT THREAD#, LOW_SEQUENCE#, HIGH_SEQUENCE# FROM V$ARCHIVE_GAP;
 
如果發現Standby Database與Primary Database Archive log有差異的話，則將有差異的archive log 傳送到Standby Database，並執行以下指令手動register archive log，紅色部份請依環境進行調整。

SQL> ALTER DATABASE REGISTER PHYSICAL LOGFILE '檔案路徑';

Stop Redo Apply
使用下列語法於standby database執行，停止同步機制。
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
 
完成Apply所有redo data
使用下列語法於standby database執行，完成apply所有收到的redo data。
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;
 

若執行以上語法出現錯誤，並且無法解決該錯誤，則使用下列語法開啟Standby Database，執行完直接跳至5.8，使用此語法則表示會有data lost。

SQL> ALTER DATABASE ACTIVATE PHYSICAL STANDBY DATABASE;

確認Standby Database可切換成Primary Database

使用下列語法於Standby Database執行，確認Standby Database可切換為Primary Databaes。

SQL> SELECT SWITCHOVER_STATUS FROM V$DATABASE;
 

將standby切換為primary
使用下列語法將Standby Database 切換為Primary Database。

SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
 
開啟資料庫
使用下列語法於Standby執行，開啟資料庫。

SQL> ALTER DATABASE OPEN;
 
 
6.	Appendix – Trouble Shooting
新增datafile 時standby FILE_MANAGEMENT未開啟造成同步失敗
	觸發錯誤原因：兩DB使用DATAGUARD作同步，並未啟用STANDBY_FILE_MANAGEMENT，
於新建立DB Table Space File造成同步異常Apply Lag。

	Error log查詢
從StandbyDB的log.xml看到錯誤log如下：
<txt>File #55 added to control file as &apos;UNNAMED00055&apos; because
<txt>the parameter STANDBY_FILE_MANAGEMENT is set to MANUAL
<txt>The file should be manually created to continue.
<txt>MRP0: Background Media Recovery terminated with error 1274
<txt>Errors in file /oracle/app/oracle/diag/rdbms/sdmp/sdmp/trace/sdmp_pr00_7456.trc:
ORA-01274: cannot add datafile &apos;/u02/oradata/sdmp/sdmp_vtxm_backup10.dbf&apos; - file could not be created
<txt>Managed Standby Recovery not using Real Time Apply
<txt>Recovery interrupted!
<txt>Recovery stopped due to failure in applying recovery marker (opcode 17.30)
<txt>Datafiles are recovered to a consistent state at change 9890354310728 but controlfile could be ahead of datafiles.
<txt>MRP0: Background Media Recovery process shutdown (sdmp)

從Primary上查詢
select DEST_NAME,STATUS,TYPE,RECOVERY_MODE,GAP_STATUS from V$ARCHIVE_DEST_STATUS where DEST_ID=2;
RECOVERY_MODE會顯示為IDEL
 

從standby上查詢
select PROCESS,STATUS,Client_PROCESS,SEQUENCE#,BLOCK#,BLOCKS from v$MANAGED_STANDBY;
會沒有看到apply log的同步資訊

參考網址為：
http://docs.oracle.com/cd/B19306_01/server.102/b14239/manage_ps.htm#i1010512


	解決方式：
1.	從standby查詢 select NAME from v$DATAFILE;
/u02/oradata/sdmp/sdmp_vtxm_backup09.dbf
/u02/oradata/sdmp/sdmp2_sdk_bak_ts04.dbf
/u02/oradata/sdmp/sdmp_sdk_ts2_01.dbf
/u02/oradata/sdmp/sdmp2_sdk_idx03.dbf
/oracle/app/oracle/product/11.2.0.3/dbs/UNNAMED00055 錯誤應該為正確名稱

2.	重新修正檔案
SQL> ALTER DATABASE CREATE DATAFILE
       '/oracle/app/oracle/product/11.2.0.3/dbs/UNNAMED00055'
       AS
       '/u02/oradata/sdmp/sdmp_vtxm_backup10.dbf'; 這邊要指名為正確的檔案路徑名稱

3.	從standby查詢 select NAME from v$DATAFILE; 確認檔案名稱是否正確
 
4.	重新啟動複寫功能
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;

5.	查詢Primary DB RECOVERY_MODE是否正確為MANAGED REAL TIME APPLY
elect DEST_NAME,STATUS,TYPE,RECOVERY_MODE,GAP_STATUS from V$ARCHIVE_DEST_STATUS where DEST_ID=2;

 
6.	查詢dataguard stats是否延遲
select trim(substr(value,2,2)* 86400 + substr(value,5,2)* 3600 + substr(value,8,2)* 60 + substr(value,-2)) "sec" 
from v$dataguard_stats where name='apply lag';
 
 

primary的log無法傳送到standby(其一)
狀況：依本文件建置，在完成primary切換動作後，於primary檢查時發現v$archive_dest_status.status=ERROR。
檢查：
1.	SQL> select status, error from v$archive_dest where dest_id = 2;應該會看到status = ERROR，而error = ORA-12154.........。
2.	alert log會出現如下的明確error code：
 
解法：
1.	$ ps -ef|grep ora_arc
找出archive background process的PID：
 
2.	$ cat /proc/<pid>/environ
找出該process的路徑設定，確認其中是否存在TNS_ADMIN的路徑設定：
 
3.	在正確的路徑放置tnsnames.ora：
 
4.	$ kill -9 <pid> [<pid> <pid> .....]
在OS層強制停止archive background process，請一次全數殺完。特別注意此動作不會影響DB運作。
 
5.	SQL> alter system switch logfile;
殺完archive background process後，該process不會自動重新長回來，故手動強制DB切換，此時archive background process會自動重啟：
 
6.	SQL> select status, error from v$archive_dest where dest_id = 2;另檢視alert log，確認是否回到預期的狀態。
7.	結束。如果原流程還有步驟沒做完的，回到原流程繼續操作。
 

若發現dataguard有gap為備份檔案過舊，請重新備份一次
或於production將漏掉的Archive檔補到standby上面
