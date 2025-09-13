---
title: "深度定制 Amazon Bedrock Agents：构建自定义业务流程编排器"
date: 2025-09-13
tags: ["GenAI", "AWS", "BedRock"]
---

# 深度定制 Amazon Bedrock Agents：构建自定义业务流程编排器

生成式 AI Agent 旨在通过与外部工具和知识库的交互，自主完成复杂的多步骤工作流，从而实现任务自动化并增强人类的能力。为了高效管理这些工作流，Agent 依赖于一个核心的“编排（Orchestration）”策略，该策略负责协调与各种工具、知识源乃至其他 Agent 的交互，以确保工作流的高效、准确和灵活。

Amazon Bedrock Agents 作为一项全托管服务，极大地简化了生成式 AI 应用的开发。其默认的编排策略是 **ReAct (Reasoning and Action)**。ReAct 是一种通用的问题解决方法，它利用基础模型（FM）的推理能力，在每一步动态规划下一步的行动。这种“思考 -> 行动 -> 观察 -> 再思考”的迭代模式虽然灵活，但在涉及多工具调用的复杂场景下，多次往返于模型的调用会显著增加延迟。

为了赋予开发者对业务流程更强的控制力，Amazon Bedrock Agents 推出了**自定义编排器（Custom Orchestrator）**功能。开发者可以通过一个 AWS Lambda 函数来完全接管 Agent 的编排逻辑，根据特定的业务需求微调 Agent 的行为，从而在性能、精度和成本之间找到最佳平衡。

本文将深入探讨自定义编排器的工作原理，并通过一个具体的 **ReWoo (Reasoning without Observation)** 模式实现案例，展示其相对于默认 ReAct 模式的优势。

## 1. 应用场景介绍

虽然默认的 ReAct 模式足以应对许多场景，但在以下情况下，您应该优先考虑使用自定义编排器：

*   **对性能（延迟）有极致要求的场景**：在实时问答、在线交易等场景中，默认 ReAct 模式的多次模型调用带来的延迟是不可接受的。自定义编排器可以通过更优化的策略（如 ReWoo）显著减少模型调用次数，降低延迟。
*   **需要执行确定性业务规则的场景**：当 Agent 的决策流程需要严格遵循固定的业务规则，而不是依赖于 LLM 的概率性推理时。例如，“如果用户是 VIP，则优先调用A工具；否则调用B工具”。这种逻辑判断更适合在确定性的代码（Lambda）中实现。
*   **需要实现复杂任务规划与并行的场景**：对于可以分解为多个独立子任务的复杂请求，自定义编排器可以一次性生成完整的执行计划，并探索并行调用工具的可能性，从而大幅提升效率。
*   **需要隐藏中间推理过程的场景**：在某些安全或合规要求较高的应用中，不希望向最终用户或日志中暴露模型的完整思考过程。自定义编排器可以精确控制返回给用户的内容，隐藏内部的执行细节。
*   **构建“元 Agent”或“路由器 Agent”**：您可以设计一个顶层 Agent，其唯一的工具就是一个自定义编排器。该编排器的职责是根据用户意图，将任务路由到不同的、更专业的下游 Agent 或工具集，实现更复杂的 Agent 协作模式。

## 2. 技术挑战与解决方案

### 2.1. 技术挑战：默认 ReAct 模式的局限性

![img](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/11/26/react_flow-1024x403.png)
*图 1: 默认 ReAct 模式的迭代流程*

默认的 ReAct 模式主要面临两大挑战：

1.  **高延迟**：ReAct 的核心是“观察-思考-行动”的循环。对于一个需要 N 个工具调用的任务，它至少需要 N+1 次对大模型的调用（N 次决定下一步行动，1 次生成最终答案）。随着任务复杂度的增加，这种串行、多次的 LLM 调用会累积导致显著的延迟。
2.  **控制力不足**：整个编排流程由 LLM 主导，开发者难以植入确定性的业务逻辑。虽然可以通过提示工程进行影响，但无法保证 LLM 在关键决策点上总能做出符合预期的选择。

### 2.2. 解决方案：自定义编排器

自定义编排器通过一个 Lambda 函数接管了 Agent 的“大脑”，将编排的控制权交还给开发者。其工作流程如下：

![img](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/11/26/custom_orchestrator-1024x450.png)
*图 2: 自定义编排器工作流*

1.  **状态与事件驱动**：Bedrock Agent 与自定义编排器 Lambda 之间通过一个基于 JSON 的“契约”进行通信。Agent 将当前的工作流**状态（State）**（如 `START`, `MODEL_INVOKED`, `TOOL_INVOKED`）传递给 Lambda。
2.  **Lambda 决策**：Lambda 函数根据接收到的状态和业务逻辑，决定下一步要执行的**事件（Event）**（如 `INVOKE_MODEL`, `INVOKE_TOOL`, `FINISH`），并将该事件作为响应返回给 Agent。
3.  **Agent 执行**：Bedrock Agent 根据 Lambda 返回的事件，执行相应的操作（调用模型、调用工具等），然后将执行结果连同新的状态再次发送给 Lambda，形成一个闭环，直到 Lambda 返回 `FINISH` 事件。

![img](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/11/26/custom_orchestrator_states-1024x490.png)
*图 3: 状态与事件流转图*

#### 案例：使用自定义编排器实现 ReWoo 模式

为了具体展示自定义编排器的威力，我们可以用它来实现 **ReWoo (Reasoning without Observation)** 模式。

![img](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/11/26/rewoo_flow-1024x138.png)
*图 4: ReWoo 模式的流程*

ReWoo 模式的核心思想与 ReAct 完全不同，它将任务执行分为两步：

1.  **规划（Plan）**：首先，通过一次模型调用，让 LLM 根据用户请求和可用工具列表，生成一个完整、详细的执行计划。这个计划明确列出了需要调用哪些工具，以及调用的顺序和参数。
2.  **执行（Execute）**：自定义编排器解析这个计划，然后依次（或并行）执行所有工具调用，**期间不再与 LLM 交互**。所有工具的输出被收集起来。
3.  **总结（Summarize）**：最后，将所有工具的输出结果一次性地提交给 LLM，让它根据这些信息生成最终的、自然的语言答案。

对于一个需要 N 个工具的复杂任务，ReWoo 模式最多只需要 **2 次**模型调用（一次规划，一次总结），从而极大地降低了延迟。

**场景对比：餐厅预订助手**

假设我们有一个餐厅助手 Agent，它能查询菜单（知识库）和预订座位（行动组）。

![img](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/11/26/restaurant_assistant_agent-1024x436.png)
*图 5: 餐厅助手 Agent 架构*

当用户提出复杂请求：“**今晚晚餐有什么菜？可以帮我预订一个今晚9点四人位吗？**”

*   **ReAct 模式**：会先调用知识库查询菜单，观察结果后，再调用行动组预订座位，期间至少发生 3 次模型调用。
    ![img](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/11/26/react_flow-1.gif)
*   **ReWoo 模式**：会先生成计划（“步骤1：查询晚餐菜单；步骤2：预订9点四人位”），然后连续执行这两个工具，最后将菜单信息和预订结果一起交给 LLM 生成答案。全程只需 2 次模型调用。经测试，对于此类复杂查询，延迟可降低 50-70%。
    ![img](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/11/26/rewoo_flow.gif)

## 3. 企业生产落地考虑和专业问答

### 企业生产落地考虑

*   **复杂性与成本**：实现自定义编排器（特别是 ReWoo）需要编写更复杂的提示词（用于生成计划）和代码逻辑（用于解析计划和调度），这会增加开发和维护成本。您需要权衡其带来的性能提升是否值得。
*   **错误处理与回滚**：在 ReWoo 模式中，由于工具链是连续执行的，一旦中间某个环节出错，缺乏 ReAct 的即时观察和调整能力。因此，编排器 Lambda 中必须包含强大的错误处理和重试逻辑，甚至在必要时能够回滚已执行的步骤。
*   **状态管理**：对于跨越多次用户交互的复杂任务，您需要在自定义编排器中设计一个明确的状态管理机制，可以利用 Lambda 的 payload、Amazon DynamoDB 或 Amazon ElastiCache 来持久化会话状态。
*   **可观测性**：由于核心逻辑转移到了 Lambda 中，必须建立完善的日志和追踪体系（如 AWS X-Ray）。确保记录下每个状态的输入、决策逻辑和输出事件，以便于调试和问题定位。
*   **安全性**：自定义编排器 Lambda 的 IAM 权限需要遵循最小权限原则。同时，对从 LLM 返回的执行计划进行安全校验，防止恶意或不符合预期的工具调用。

### 专业问答 (FAQ)

**Q1: 自定义编排器与默认 ReAct 模式的核心区别是什么？**
A: 核心区别在于**控制权**。ReAct 模式下，编排的控制权在 LLM，它在每一步进行推理决策。而在自定义编排器模式下，控制权在您的 Lambda 代码中，LLM 变成了被代码按需调用的一个“工具”，您可以实现任何确定性或非确定性的编排逻辑。

**Q2: 我应该在什么时候考虑使用自定义编排器？**
A: 当默认 ReAct 模式无法满足您的业务需求时。典型场景包括：对延迟非常敏感的应用、需要严格遵循业务规则的流程、或者希望实现 ReWoo、树状思考（Tree of Thoughts）等更高级 Agent 模式的场景。

**Q3: 实现 ReWoo 模式的主要难点在哪里？**
A: 主要有两点：1. **提示工程**：您需要设计一个能让 LLM 稳定输出结构化、可解析的执行计划的提示（Prompt）。2. **代码实现**：您需要编写 Lambda 代码来精确地解析这个计划，并调度相应的工具执行。这比直接使用默认 Agent 的开发成本要高。

**Q4: 自定义编排器会影响 Bedrock Agent 的其他功能（如知识库、Guardrails）吗？**
A: 不会。自定义编排器是与这些功能协同工作的。您的 Lambda 函数可以决定何时调用知识库（通过返回 `INVOKE_TOOL` 事件并指定知识库 ARN），或者何时对模型输出应用 Guardrails（通过返回 `APPLY_GUARDRAILS` 事件）。它提供了更灵活的组合这些功能的能力。

## 5. 代码解释：自定义编排器 Lambda 伪代码

虽然完整的代码在参考仓库中，但以下伪代码清晰地展示了编排器 Lambda 的核心逻辑：

```python
import json

def lambda_handler(event, context):
    """
    自定义编排器 Lambda 函数的核心逻辑。
    它根据 Bedrock Agent 传递过来的状态，决定下一步的行动。
    """
    
    # 从输入事件中获取当前状态和用户请求等信息
    current_state = event['state']
    user_input = event['userInput']
    session_attributes = event['sessionAttributes']

    # 决策逻辑：根据不同状态执行不同操作
    if current_state == 'START':
        # 工作流开始，第一步是让模型生成执行计划
        # 这里的 prompt_for_plan 是一个精心设计的、用于生成 ReWoo 计划的提示
        response_event = {
            'type': 'INVOKE_MODEL',
            'modelInvocationInput': {
                'prompt': prompt_for_plan(user_input),
                'inferenceConfig': { ... }
            },
            # 将下一步的状态定义为 'PLAN_GENERATED'，以便后续处理
            'nextState': 'PLAN_GENERATED' 
        }

    elif current_state == 'PLAN_GENERATED':
        # 模型已生成计划，现在需要解析计划并执行工具
        model_output = event['modelOutput']['text']
        
        # parse_plan() 是一个自定义函数，用于从模型输出中解析出结构化的工具调用列表
        tool_calls = parse_plan(model_output)
        
        # 将解析出的工具调用计划存入会话属性，以便后续步骤使用
        session_attributes['tool_plan'] = tool_calls
        session_attributes['current_tool_index'] = 0
        
        # 准备执行第一个工具
        next_tool = tool_calls[0]
        response_event = {
            'type': 'INVOKE_TOOL',
            'toolInvocationInput': {
                'toolName': next_tool['name'],
                'parameters': next_tool['parameters']
            },
            'nextState': 'TOOL_EXECUTED'
        }

    elif current_state == 'TOOL_EXECUTED':
        # 一个工具已执行完毕，保存其输出，并检查是否还有下一个工具
        tool_result = event['toolOutput']
        session_attributes['tool_results'].append(tool_result)
        
        current_index = session_attributes['current_tool_index']
        tool_plan = session_attributes['tool_plan']
        
        if current_index + 1 < len(tool_plan):
            # 如果还有未执行的工具，则继续执行下一个
            session_attributes['current_tool_index'] += 1
            next_tool = tool_plan[current_index + 1]
            response_event = {
                'type': 'INVOKE_TOOL',
                'toolInvocationInput': { ... }, # 调用下一个工具
                'nextState': 'TOOL_EXECUTED'
            }
        else:
            # 所有工具都已执行完毕，现在将所有结果汇总给模型进行总结
            final_prompt = prompt_for_summary(user_input, session_attributes['tool_results'])
            response_event = {
                'type': 'INVOKE_MODEL',
                'modelInvocationInput': {
                    'prompt': final_prompt
                },
                'nextState': 'FINAL_RESPONSE_GENERATED'
            }

    elif current_state == 'FINAL_RESPONSE_GENERATED':
        # 模型已生成最终答案，工作流结束
        final_answer = event['modelOutput']['text']
        response_event = {
            'type': 'FINISH',
            'finalResponse': {
                'text': final_answer
            }
        }
        
    else:
        # 异常处理
        response_event = {
            'type': 'FINISH',
            'finalResponse': { 'text': '抱歉，我遇到一个未知错误。' }
        }

    # 将决策后的事件返回给 Bedrock Agent
    return response_event

```

## 6. 参考文档

*   **官方代码示例仓库**: [amazon-bedrock-samples on GitHub](https://github.com/aws-samples/amazon-bedrock-samples/tree/main/agents-and-function-calling/bedrock-agents/features-examples/14-create-agent-with-custom-orchestration)
*   **Amazon Bedrock Agents 官方文档**: [Custom orchestration for Agents for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-custom-orchestration.html)
