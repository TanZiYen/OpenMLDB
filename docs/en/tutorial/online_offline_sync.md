# Offline Data Synchronization

Offline data synchronization involves the process of copying online data to an offline storage location. This offline storage location is a user-specified high-capacity persistent storage address, which may not necessarily coincide with the offline data address in the OpenMLDB table. Currently, offline synchronization supports two methods: synchronization to a **disk table** and writing to an **HDFS cluster**.

To enable the offline synchronization feature, two components need to be deployed: DataCollector and SyncTool. In the initial phase, only a single SyncTool is supported, while DataCollector should be deployed on **each machine where TabletServer is running**. For instance, if multiple TabletServers are running on a single machine, the synchronization task will utilize one DataCollector on that machine. If additional DataCollectors are added, they won't function until the currently running DataCollector is taken offline, after which the next DataCollector will continue the task.

Although SyncTool supports only single-unit operations, it can resume its work progress upon restarting, without requiring additional actions.

## Deployment Method

Due to the behavior of SyncTool, if it is started first, it might attempt to allocate synchronization tasks without the presence of a DataCollector. Therefore, it's important to ensure that all DataCollectors are started before initiating SyncTool.

For offline synchronization to work, the TabletServer version in the cluster must be greater than v0.7.3. If your cluster version is lower than v0.7.3, please perform an upgrade first. The DataCollector is located in the `bin` directory of the deployment package, and SyncTool is located in the root directory under `synctool`. Both can be initiated through `bin/start.sh`.

### DataCollector

#### Configuration (Important)

Make sure to update the configuration file `<package>/conf/data_collector.flags`. In the configuration file, enter the correct ZooKeeper (zk) address and path, and configure the `endpoint` without port conflicts (the endpoint should match the TabletServer's endpoint. If TabletServer uses the local public IP and DataCollector endpoint uses the 127.0.0.1 address, it cannot be automatically converted).

It's important to choose the `collector_datadir` carefully. During synchronization, we create hard links to the disk data of TabletServer. Therefore, the `collector_datadir` must share the same disk as TabletServer's `hdd_root_path/ssd_root_path`, or else you may encounter an error message like `Invalid cross-device link`.

The `max_pack_size` defaults to 1M. If you have numerous synchronization tasks, you might encounter prompts like `[E1011]The server is overcrowded`. In such cases, consider adjusting this configuration appropriately. You can also modify the `socket_max_unwritten_bytes` to increase the write cache tolerance.

#### Start

```
./bin/start.sh start data_collector
```

#### Status Confirmation

After starting SyncTool, you can use the following command to access real-time DataCollector RPC status pages. In case of failure, please refer to the log for further details.

```
curl http://<data_collector>/status
```

Currently, we do not have the ability to query the list of `DataCollectors`. We plan to provide support for this feature in future updates of related tools.

### SyncTool

#### Configuration

- To configure SyncTool, please update the `<package>/conf/synctool.properties` file. This configuration will overwrite the `<package>/synctool/conf/synctool.properties` file during startup.
- Currently, SyncTool only supports direct writing to HDFS. You can configure the HDFS connections through the `hadoop.conf.dir` property in the properties file or set the `HADOOP_CONF_DIR` environment variable. Ensure that the OS startup user of SyncTool has write access to the HDFS path specified when creating each synchronization task.

#### Start

```
./bin/start.sh start synctool
```

SyncTool currently operates as a single process. If you start multiple processes, they will operate independently of each other. It's important for users to verify that there are no duplicate synchronization tasks. SyncTool continuously saves progress in real-time, and if it goes offline and comes back online, it can resume task progress from where it left off.

SyncTool is responsible for managing synchronization tasks, collecting data, and writing to offline addresses. Let's first clarify the task hierarchy. SyncTool receives a "table synchronization task" from the user and then breaks it down into multiple "sharded synchronization tasks," which we will refer to as subtasks.

In terms of task management, if a DataCollector becomes unavailable or encounters an error, it will be instructed to recreate the task. If SyncTool cannot find a suitable DataCollector for task reassignment, it will mark the task as failed. If this happens, SyncTool will continue trying to assign new tasks, and the synchronization task's progress will stall without obvious errors. To promptly detect such issues, these situations are labeled as failed subtasks.

Since creating a table synchronization task does not support starting from a specific point, it's currently not advisable to delete the task and create it anew. Doing so, especially if the destination remains the same, can result in a significant amount of duplicate data. Instead, consider altering the synchronization destination or restarting SyncTool (Note: SyncTool recovery does not check whether the subtask has previously failed; it starts the task in an 'init' state).

#### SyncTool Helper

Create, delete, and query synchronization tasks using `<package>/tools/synctool_helper.py`.

```bash
# create 
python tools/synctool_helper.py create -t db.table -m 1 -ts 233 -d /tmp/hdfs-dest [ -s <sync tool endpoint> ] 
# delete
python tools/synctool_helper.py delete -t db.table [ -s <sync tool endpoint> ] 
# task status
python tools/synctool_helper.py status [ -s <sync tool endpoint> ] 
# sync tool status for dev
python tools/synctool_helper.py tool-status [ -f <properties path> ]
```

The `Mode` configuration can be set to 0, 1, or 2, corresponding to three synchronization modes: FULL for full synchronization, INCREMENTAL_BY_TIMESTAMP for incremental synchronization based on timestamps, and FULL_AND_CONTINUOUS for continuous full synchronization. If using Mode 1, you can configure the `-ts` option to specify a start time, and data older than this time will not be synchronized.

In Mode 0, there is no strict termination point. Each sub-task will end once it has synchronized the current data. Upon completion, the entire table task will be deleted. If you use a helper to query the status and find no synchronization task for the table, it is considered as completed. In Modes 1 and 2, synchronization will never stop and will run continuously.

Status results:

Executing the `status` command will print the status of each partition task. If you are only interested in the overall situation, you can look at the content after the `table scope`, which displays the table-level synchronization task status. If there are any `FAILED` sub-tasks for a table, it will be indicated.

For each sub-task, pay attention to its 'status' field. If it has just started and has not yet received the first data synchronization from DataCollector, it will be in the INIT status. Once it receives the first data synchronization, it will transition to the RUNNING status. (We particularly pay attention to the initial status of DataCollector and SyncTool, so INIT status is set specially.) If the synchronization task is resumed with the restart of SyncTool, it will directly enter the RUNNING status. During the process, there may be a REASSIGNING status, which is an intermediate state and does not mean the task is unavailable. In Mode 0, you may encounter the SUCCESS status, indicating that the task has been completed. When all sub-tasks for a table are completed, SyncTool will automatically clean up the task for that table, and the task for that table will not be found using a helper query.

Only the FAILED status indicates that the sub-task has failed and will not be retried or deleted. After identifying and fixing the cause of the failure, you can delete and recreate the synchronization task. If you don't want to lose the progress that has been imported, you can restart SyncTool to resume the task (but it may result in more duplicate data).

## Functional Boundary

The way the table snapshot progress is marked in the DataCollector lacks uniqueness. In the event that a subtask terminates prematurely and a new task is created using the current progress, there may be a section of duplicated data.

In the case of SyncTool writing to HDFS, it first performs the write operation and then persists the progress. If SyncTool were to shut down during this process, since the progress hasn't been updated yet, a portion of the data might be repeatedly written. Given the inherent potential for duplication in this feature, no further action will be taken at this time.