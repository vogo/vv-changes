# Requirement: Orchestrator Agent as Main Agent

## Original Requirement (Chinese)

vv 命令启动的时候,把获取当前目录; 主agent为编排调度agent, 主agent获得指令后,理解并拆解任务调度子agent完成任务,然后汇总结果回复用户

## Translation

When the vv command starts:
1. Get the current working directory
2. The main agent is an orchestration/scheduling agent (replacing the current Router)
3. When the main agent receives an instruction, it should:
   - Understand the instruction
   - Decompose the task into sub-tasks
   - Dispatch sub-agents to complete the sub-tasks
   - Aggregate results and reply to the user
