---
title: activiti流程退回
date: 2021-01-18 18:40:27
tags: activiti
---
activiti 默认是没有标准退回功能的，然而在业务需求是要一个退回功能，而且他们还不想自己拖一个退回的线去实现退回，没办法只能做一个通用一点的退回给他们使用。


### activiti 流程退回


```
import com.definesys.mpaas.common.exception.MpaasBusinessException;
import org.activiti.bpmn.model.FlowElement;
import org.activiti.bpmn.model.FlowNode;
import org.activiti.bpmn.model.SequenceFlow;
import org.activiti.engine.HistoryService;
import org.activiti.engine.RepositoryService;
import org.activiti.engine.RuntimeService;
import org.activiti.engine.TaskService;
import org.activiti.engine.history.HistoricTaskInstance;
import org.activiti.engine.impl.interceptor.Command;
import org.activiti.engine.impl.interceptor.CommandContext;
import org.activiti.engine.impl.persistence.entity.ExecutionEntity;
import org.activiti.engine.impl.persistence.entity.TaskEntity;
import org.activiti.engine.task.Task;

import java.util.List;

public class BackWardCmd implements Command<String> {

    private Task task;

    public BackWardCmd(Task task) {
        this.task = task;
    }

    @Override
    public String execute(CommandContext commandContext) {
        FlowElement element = this.getPreNode(this.task, commandContext);
        if (element == null) {
            throw new MpaasBusinessException("该节点不能进行退回");
        }
        SequenceFlow flow = this.findAcessSequenceFlow((FlowNode) element);
        ExecutionEntity currentEntry = commandContext.getExecutionEntityManager().findById(task.getExecutionId());

        ExecutionEntity parentExecutionEntry = currentEntry.getParent();

        commandContext.getExecutionEntityManager().deleteChildExecutions(parentExecutionEntry, "backWard", true);
        ExecutionEntity childEntry = commandContext.getExecutionEntityManager().createChildExecution(parentExecutionEntry);
        childEntry.setCurrentFlowElement(flow);
        commandContext.getAgenda().planContinueProcessOperation(childEntry);
        return childEntry.getId();
    }

    private FlowElement getPreNode(Task task, CommandContext context) {
        HistoryService historyService = context.getProcessEngineConfiguration().getHistoryService();
        List<HistoricTaskInstance> items = historyService.createHistoricTaskInstanceQuery()
                .processInstanceId(task.getProcessInstanceId())
                .orderByHistoricTaskInstanceStartTime()
                .desc()
                .list();
        if (items == null || items.size() == 0) {
            throw new MpaasBusinessException("未找到上一节点，无法退回");
        }
        String currentNodeId = task.getTaskDefinitionKey();
        String preNodeId = null;
        for (int i = 0; i < items.size(); ++i) {
            HistoricTaskInstance item = items.get(i);
            if (currentNodeId.equals(item.getTaskDefinitionKey()) || "reject".equals(item.getDeleteReason()) || "backWard".equals(item.getDeleteReason())) {
                continue;
            }
            preNodeId = item.getTaskDefinitionKey();
            break;
        }
        if (preNodeId == null) {
            return null;
        }
        RepositoryService repositoryService = context.getProcessEngineConfiguration().getRepositoryService();
        org.activiti.bpmn.model.Process process = repositoryService.getBpmnModel(task.getProcessDefinitionId()).getMainProcess();
        FlowElement node = process.getFlowElement(preNodeId);
        return node;
    }

    private SequenceFlow findAcessSequenceFlow(FlowNode node) {
        List<SequenceFlow> flows = node.getIncomingFlows();
        if (flows == null || flows.size() == 0) {
            throw new MpaasBusinessException("该节点不能进行退回");
        }
        //找没有加条件的连线
        for (SequenceFlow flow : flows) {
            if (flow.getConditionExpression() == null) {
                return flow;
            }
        }
        //如果都没有选择第一条
        return flows.get(0);
    }
}

```
调用方法很简单。直接使用managementService进行调用，其中task为谁点的退回按钮，就是谁的task。
```
managementService.executeCommand(new BackWardCmd(task));
```