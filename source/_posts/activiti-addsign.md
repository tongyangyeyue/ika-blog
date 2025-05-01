---
title: Activiti6 加签且保留待办
date: 2021-01-12 12:25:02
---
在一个平常且忙碌的一天，突然接收到一个需求，你们的加签不符合我们想的情况，需要可以加签多人，并且还保留操作人的待办。  
虽说这个需求也挺合理，但开发时可就犯了难，activiti默认也没有加待办的功能啊。只能从底层的对象关系入手了。


## 思路
把上面的需求转变下，无非就是要向待办中添加人去审批。根据activiti的方式来看，无非就是把task添加到instance而已。所以只要能将task添加到instance中，并且把所有参数都弄的和原始的待办相似 就差不多完成了。

## 实现API

先看到taskService里面好像有保存task的API。不过这个API看起来就太简陋了，一看就不能用。
```
Task task = taskService.newTask();
taskService.saveTask(task);
```

除此之外，常用的TaskService、RuntimeService就没有啥可以用的了，只能用上功能强大的ManagementService
```
ExecutionEntityManager.createChildExecution()
TaskEntityManager.create()
TaskEntityManager.insert()
```

## 实现过程
先把需要加签的那个人的task信息查出来，然后用传入到ManagementService里做处理

```
public void testAddone(String taskId){
    Task task = taskService.createTaskQuery()
                .taskId(taskId)
                .includeProcessVariables()
                .singleResult();
    String processInstanceId = task.getProcessInstanceId();
    managementService.executeCommand(new AddoneTestCmd(processInstanceId,task));
    List<Task> afterTask = taskService.createTaskQuery()
    .processInstanceId(processInstanceId)
                .includeProcessVariables()
                .list();
    System.out.println(afterTask.size());
}
```

先找到这个task所在的ExecutionEntity与TaskEntity
```
public class AddoneTestCmd implements Command<String> {

    private String instanceId;

    private Task task;

    public AddoneTestCmd(String instanceId, Task task) {
        this.instanceId = instanceId;
        this.task = task;
    }

    @Override
    public String execute(CommandContext commandContext) {
        ExecutionEntity executionEntity = commandContext.getExecutionEntityManager()
                .findById(task.getExecutionId());
        TaskEntity taskEntity = commandContext.getTaskEntityManager().findById(task.getId());
    }
}

```
然后要新建一个TaskEntity并将原来task的数据放进去最后放进去
```
@Override
public String execute(CommandContext commandContext) {
    ExecutionEntity executionEntity = commandContext.getExecutionEntityManager()
                .findById(task.getExecutionId());
    TaskEntity taskEntity = commandContext.getTaskEntityManager().findById(task.getId());
    TaskEntity newTaskEntity = commandContext.getTaskEntityManager().create();
    newTaskEntity.setVariables(task.getProcessVariables());
    newTaskEntity.setAssignee("admin");
    newTaskEntity.setProcessInstanceId(instanceId);
    newTaskEntity.setCategory(task.getCategory());
    newTaskEntity.setDescription(task.getDescription());
    newTaskEntity.setName(taskEntity.getName());
    newTaskEntity.setTaskDefinitionKey(taskEntity.getTaskDefinitionKey());
    newTaskEntity.setExecution(executionEntity);
    newTaskEntity.setExecutionId(task.getExecutionId());
    commandContext.getTaskEntityManager().insert(newTaskEntity, executionEntity);
}

```
这看起来不错，task也加进去了，再次查看时候参数也都差不多在。
正当高兴的时候，我点击了下同意，芜湖，果不其然报错了。
```
org.activiti.engine.ActivitiException: UserTask should not be signalled before complete
```
在找到底层的代码后，发现是complete的时候需要遍历这个流程的task，看里面的各种标志位是否都正确。  
仔细看了一圈之后，也不知道咋改，只能再想想哪里出错了。
后来发现流程里面有3个变量nrOfInstances、nrOfActiveInstances、nrOfCompletedInstances  分别代表这条实例的任务总数、未完成任务数、已完成任务数。
把task添加进去的时候 这些变量不会自动加，那就手动修改下。
```
Integer beginNrofInstance = (Integer) executionEntity.getVariable("nrOfInstances");
Integer beginNrOfActiveInstances = (Integer) executionEntity.getVariable("nrOfActiveInstances");
executionEntity.setVariable("nrOfInstances", beginNrofInstance + 1);
executionEntity.setVariable("nrOfActiveInstances", beginNrOfActiveInstances + 1);
```
然而，还是不行，依旧是上面的错误。看起来并不是这个变量的问题，不过这个变量修改确实是需要的。  
这时，突然发现其实一个instance里面 每个task的executionId都是不同的，而我按上面的操作后，新生成的executionId和原来传入的task相同了。于是我就明白了，应该新生成一个ExecutionEntity才对。
```
ExecutionEntity parentExecutionEntry = executionEntity.getParent();
ExecutionEntity newChildExecution = commandContext.getExecutionEntityManager().createChildExecution(parentExecutionEntry);
commandContext.getTaskEntityManager().insert(newTaskEntity, newChildExecution);
```
好家伙，这一通操作下来看起来应该是可以了。在页面点了一下同意，咔 又报错了，不过这次错误变了，说明有戏
```
org.activiti.engine.ActivitiException: Programmatic error: no current flow element found or invalid type: null. Halting.
```
看起来像没有CurrentFlowElement，那好 就添加一个CurrentFlowElement。
```
newChildExecution.setCurrentFlowElement(executionEntity.getCurrentFlowElement());
```
最终，基础的功能实现了。

## 整体代码
```
public void testAddone(String taskId){
    Task task = taskService.createTaskQuery()
                .taskId(taskId)
                .includeProcessVariables()
                .singleResult();
    String processInstanceId = task.getProcessInstanceId();
    managementService.executeCommand(new AddoneTestCmd(processInstanceId,task));
    List<Task> afterTask = taskService.createTaskQuery()
    .processInstanceId(processInstanceId)
                .includeProcessVariables()
                .list();
```

```
public class AddoneTestCmd implements Command<String> {

    private String instanceId;

    private Task task;

    public AddoneTestCmd(String instanceId, Task task) {
        this.instanceId = instanceId;
        this.task = task;
    }

    @Override
    public String execute(CommandContext commandContext) {
        ExecutionEntity executionEntity = commandContext.getExecutionEntityManager()
                .findById(task.getExecutionId());
        ExecutionEntity parentExecutionEntry = executionEntity.getParent();
        ExecutionEntity newChildExecution = commandContext.getExecutionEntityManager().createChildExecution(parentExecutionEntry);
        Integer beginNrofInstance = (Integer) newChildExecution.getVariable("nrOfInstances");
        Integer beginNrOfActiveInstances = (Integer) newChildExecution.getVariable("nrOfActiveInstances");
        newChildExecution.setCurrentFlowElement(executionEntity.getCurrentFlowElement());
        TaskEntity taskEntity = commandContext.getTaskEntityManager().findById(task.getId());
        TaskEntity newTaskEntity = commandContext.getTaskEntityManager().create();
        newTaskEntity.setVariables(task.getProcessVariables());
        newTaskEntity.setAssignee("admin");
        newTaskEntity.setProcessInstanceId(instanceId);
        newTaskEntity.setCategory(task.getCategory());
        newTaskEntity.setDescription(task.getDescription());
        newTaskEntity.setName(taskEntity.getName());
        newTaskEntity.setTaskDefinitionKey(taskEntity.getTaskDefinitionKey());

        commandContext.getTaskEntityManager().insert(newTaskEntity, newChildExecution);
        newChildExecution.setVariable("nrOfInstances", beginNrofInstance + 1);
        newChildExecution.setVariable("nrOfActiveInstances", beginNrOfActiveInstances + 1);
        return null;
    }
}
```
