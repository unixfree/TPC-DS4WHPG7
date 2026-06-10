# Scripts for TPCDS Workload Test for WarehousePG 6 and 7

## Mixed workload type
Mixed workload can be adjusted by modifying the source. 
1. Simple loading (15)   : CTAS with select * from tables 
2. transform loading (6) : CTAS with array_agg and group by
3. Simple queries (8)    : For clearing cache memory of Segment nodes
4. Analysis queryes (TPC-DS, 40) : 40 of 99 TPCDS

## Current test environment
 - TPDCS : GEN_DATA_SCALE: 3000GB
 - 1 Master + 4 Segment Nodes
 - Segment node spec
   * CPU : 48 core (96w/HT)
   * Memory: 512 GB
   * Disk : NVME 4EA (no raid)
   * Network : 100Gbps * 2EA

## Rrerequisite

```
1. Install WarehousePG 6 or 7
2. Install WarehousePG Enterprise Manager
3. Add permissions to pg_hba.conf
[gpadmin@mdw]$ cd $COORDINATOR_DATA_DIRECTORY/
[gpadmin@mdw gpseg-1]$ vi pg_hba.conf
local all all trust
host all all 0.0.0.0/0 trust
[gpadmin@mdw gpseg-1]$ gpstop -u
[gpadmin@mdw gpseg-1]$

4. Setting up the environment to perform WarehousePG TPCDS
## Set greenplum_path.sh environment on all segment nodes in WarehousePG
[gpadmin@mdw data]$ ssh sdw1 
[gpadmin@sdw1 ~]$ vi .bashrc
source /usr/local/greenplum-db/greenplum_path.sh               ## Add greenplum_path.sh on .bashrc of segment node
[gpadmin@sdw1 ~]$ scp .bashrc sdw2:/home/gpadmin/.bashrc
.bashrc                                                        100%  280   165.2KB/s   00:00
[gpadmin@sdw1 ~]$ scp .bashrc sdw3:/home/gpadmin/.bashrc
.bashrc                                                        100%  280   436.8KB/s   00:00
[gpadmin@sdw1 ~]$ scp .bashrc sdw4:/home/gpadmin/.bashrc
.bashrc                                                        100%  280   345.6KB/s   00:00
[gpadmin@sdw1 ~]$

```

## Path and File
```
$ cd /data
$ unzip TPC-DS4WHPG7-main.zip
$ cd TPC-DS4WHPG7-main
$ ls -la
합계 32
drwxrwxr-x  6 gpadmin gpadmin    79  4월  9 21:49 .
drwxr-xr-x 15 gpadmin gpadmin  4096  4월 11 10:23 ..
-rw-rw-r--  1 gpadmin gpadmin    99  4월  9 21:49 README.md
drwxrwxr-x 14 gpadmin gpadmin  4096  4월 11 10:26 TPC-DS         ## WarehousePG TPC-DS github codes from https://github.com/pivotal/TPC-DS
drwxrwxr-x  2 gpadmin gpadmin     6  4월 11 10:24 log            ## log folder
drwxrwxr-x  2 gpadmin gpadmin  4096  4월  9 21:49 log_gather     ## Scripts to extract query time and resource usage from logs       
drwxrwxr-x  2 gpadmin gpadmin 12288  4월  9 21:49 shell          ## Scripts for tpcds and mixed workload
```

## How to run
```
1. Initial WarehousePG GUC setup
   run with Resource Group or Resource Queue
$ ls
README.md  TPC-DS  log  log_gather  shell
$ cd shell/
$ ll gpconfigs.sh
-rwxr-xr-x 1 gpadmin gpadmin 461  4월  9 21:49 gpconfigs.sh
$ cat gpconfigs.sh
gpconfig -c gp_vmem_protect_limit -v 24576
gpconfig -c gp_workfile_compression -v on --masteronly
gpconfig -c max_connections -m 500 -v 1500
gpconfig -c gp_resource_manager -v group
gpconfig -c gp_segment_connect_timeout -v 20min
gpconfig -c gp_fts_probe_timeout -v 60s
gpconfig -c log_duration -v on --masteronly
gpconfig -c log_min_duration_statement -v 0 --masteronly
gpconfig -c gp_resource_group_cpu_limit -v 1

#gpconfig -c gp_resource_manager -v "queue"
$ sh gpconfigs.sh
$ gpstop -af
$ gpstart -a

2. Performing TPCDS and Mixed workloads
$ ll poc*
-rwxr-xr-x 1 gpadmin gpadmin 146  4월 15 09:04 poc_run_all.sh
-rwxr-xr-x 1 gpadmin gpadmin 655  4월  9 21:49 poc_tpcds_init.sh
-rwxr-xr-x 1 gpadmin gpadmin 657  4월  9 21:49 poc_tpcds_normal.sh
-rwxr-xr-x 1 gpadmin gpadmin 648  4월  9 21:49 poc_workload.sh
$ cat poc_run_all.sh
## init or normal
./poc_tpcds_init.sh          ## When first performing compilation and data generation, etc.
#./poc_tpcds_normal.sh       ## When re-executing  tpcds after already executing poc_tpcds_init.sh (data is not regenerated)

sleep 600

./1.1_create_role.sh         ## Create role and Resource Queue/Group
./1.2_create_tb_all.sh       ## Create tables to perform mixed workload, tpcds must be run in advance.
sleep 600

./poc_workload.sh            ## Perform mixed workload
$ nohup ./poc_run_all.sh & 

```


## Extract log summaries and system resource usage
```
1. Initial WarehousePG GUC setup
$ cd log_gather/
$ cat gather_log_all.sh
##### For tpcds
#tpcds log summary
./tpcds_sum_normal.sh
./gather_tpcds_normal.sh > ../log/tpcds.log.normal.result

#Extract system resource usage when performing tpcds.
#Check the start time and end time in ../poc_tpcds_init.sh.log or poc_tpcds_normal.sh.log
./2.1.system_rsc.sh normal '20240318_22:50:00' '20240319_04:00:00'     #2.1.system_rsc.sh.log.nomal

##### For mixed workload
# mixed workload log summary
./workload_result.sh       # output file: ../log/workload_result.csv
./workload_result_avg.sh   # output file: ../log/workload_result_avg.csv

#Extract system resource usage when performing mixed workload.
#Check the start time and end time in ../poc_woload.sh.log
./2.1.system_rsc_2min.sh workload '2024-02-25_11:00:00' '2024-02-25_13:10:00'        #2.1.system_rsc_2min.sh.log.workload
./2.1.system_rsc_cpu.sh workload_cpu '2024-02-25_11:00:00' '2024-02-25_13:10:00'     #2.1.system_rsc_cpu.sh.log.workload_cpu

## for TPCDS
$ sh ./tpcds_sum_normal.sh
$ sh ./gather_tpcds_normal.sh > ../log/tpcds.log.normal.result
$ cat ../log/tpcds.log.normal.result

$ sh ./2.1.system_rsc.sh normal 'Start time' 'End Time'     ## Extract time from ../log/poc_tpcds_init.sh.log or ../log/poc_tpcds_normal.sh.log
$ cat ../log/2.1.system_rsc.sh.log.normal

## For mixed workload
$ sh ./workload_result.sh 
$ cat ../log/workload_result.csv
$ sh workload_result_avg.sh
$ cat ../log/workload_result_avg.csv

$ sh ./2.1.system_rsc_2min.sh workload 'Start time' 'End Time'     ## Extract time from ../log/poc_workload.sh.log 
$ cat ../log/2.1.system_rsc_2min.sh.log.workload

$ ./2.1.system_rsc_cpu.sh workload_cpu 'Start time' 'End Time'     ## Extract time from ../log/poc_workload.sh.log 
$ cat ../log/2.1.system_rsc_cpu.sh.log.workload_cpu
```

## Notice
```
Rockey 9.5에서 TPCDS 컴파일할 때 오류 발생시에는 gcc 9.5 설치로 해결
Rockey 9.5에서 gcc 9.5 설치 방법은 Rockey9.5_WarehousePG_TPCDS.txt 파일 참조
```
