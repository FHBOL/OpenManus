# OpenManus 项目架构图

本文档使用 Mermaid 图表展示 OpenManus 项目的架构、流程和类结构。

## 项目框架图

```mermaid
graph TD
    subgraph "核心组件"
        A["main.py"] --> B["Manus Agent"]
        C["run_flow.py"] --> D["Flow Factory"]
        D --> E["Planning Flow"]
        E --> B
    end

    subgraph "Agent 模块"
        B --> BA["BaseAgent"]
        B --> BB["ToolCallAgent"]
        B --> BC["BrowserContextHelper"]
    end

    subgraph "Flow 模块"
        E --> EA["BaseFlow"]
        E --> EB["PlanningFlow"]
    end

    subgraph "Tool 模块"
        F["Tools"] --> FA["BaseTool"]
        F --> FB["PlanningTool"]
        F --> FC["PythonExecute"]
        F --> FD["BrowserUseTool"]
        F --> FE["StrReplaceEditor"]
        F --> FF["Terminate"]
        F --> FG["ToolCollection"]
    end

    subgraph "Prompt 模块"
        G["Prompts"] --> GA["Manus Prompts"]
        G --> GB["Planning Prompts"]
        G --> GC["Browser Prompts"]
        G --> GD["SWE Prompts"]
    end

    B --> F
    E --> F
    B --> G
    E --> G
```

## 执行流程图

### main.py 执行流程

```mermaid
sequenceDiagram
    participant User
    participant Main as main.py
    participant Manus as Manus Agent
    participant LLM
    participant Tools

    User->>Main: 输入提示(prompt)
    Main->>Manus: 创建 Manus 实例
    Main->>Manus: 调用 run(prompt)
    Manus->>LLM: 发送系统提示和用户输入
    loop 执行步骤
        LLM->>Manus: 返回下一步操作
        Manus->>Tools: 执行工具调用
        Tools->>Manus: 返回工具执行结果
        Manus->>LLM: 更新上下文
    end
    Manus->>Main: 完成任务
    Main->>User: 显示结果
```

### run_flow.py 执行流程

```mermaid
sequenceDiagram
    participant User
    participant RunFlow as run_flow.py
    participant FlowFactory
    participant PlanningFlow
    participant Manus as Manus Agent
    participant Tools

    User->>RunFlow: 输入提示(prompt)
    RunFlow->>Manus: 创建 Manus 实例
    RunFlow->>FlowFactory: 创建流程(PLANNING)
    FlowFactory->>PlanningFlow: 实例化 PlanningFlow
    RunFlow->>PlanningFlow: 执行 flow.execute(prompt)

    PlanningFlow->>PlanningFlow: 创建初始计划
    loop 执行计划步骤
        PlanningFlow->>Manus: 获取执行器
        Manus->>Tools: 执行工具调用
        Tools->>Manus: 返回工具执行结果
        Manus->>PlanningFlow: 更新步骤状态
    end

    PlanningFlow->>RunFlow: 返回执行结果
    RunFlow->>User: 显示结果
```

## 类图

### Agent 类层次结构

```mermaid
classDiagram
    class BaseAgent {
        <<abstract>>
        +name: str
        +description: str
        +system_prompt: str
        +next_step_prompt: str
        +llm: LLM
        +memory: Memory
        +state: AgentState
        +max_steps: int
        +current_step: int
        +initialize_agent()
        +state_context()
        +update_memory()
        +run(prompt: str)
        +handle_stuck_state()
        +is_stuck()
        +step()*
    }
    class ReactAgent {
        +step()
        +think()*
        +act()*
    }

    class ToolCallAgent {
        +available_tools: ToolCollection
        +special_tool_names: list
        +tool_choices: TOOL_CHOICE_TYPE
        +tool_calls: List[ToolCall]
        +max_observe:Optional[Union[int, bool]]
        +execute_tool()
        +cleanup()
        +act()
        +think()

    }

    class Manus {
        +browser_context_helper: BrowserContextHelper
        +initialize_helper()
        +cleanup()
        +think()
    }

    BaseAgent <|-- ReactAgent
    ReactAgent <|-- ToolCallAgent
    ToolCallAgent <|-- Manus
```

### Flow 类层次结构

```mermaid
classDiagram
    class BaseFlow {
        <<abstract>>
        +agents: Dict[str, BaseAgent]
        +tools: List
        +primary_agent_key: str
        +primary_agent: BaseAgent
        +get_agent(key: str)
        +add_agent(key: str, agent: BaseAgent)
        +execute(input_text: str)*
    }

    class PlanningFlow {
        +llm: LLM
        +planning_tool: PlanningTool
        +executor_keys: List[str]
        +active_plan_id: str
        +current_step_index: int
        +get_executor(step_type: str)
        +execute(input_text: str)
    }

    BaseFlow <|-- PlanningFlow
```

### Tool 类层次结构

```mermaid
classDiagram
    class BaseTool {
        <<abstract>>
        +name: str
        +description: str
        +parameters: dict
        +execute()*
        +to_param()
    }

    class ToolResult {
        +output: Any
        +error: str
        +base64_image: str
        +system: str
        +replace()
    }

    class PlanningTool {
        +name: str
        +description: str
        +parameters: dict
        +plans: dict
        +_current_plan_id: str
        +execute()
    }

    class ToolCollection {
        +tools: List[BaseTool]
        +get_tool(name: str)
        +add_tool(tool: BaseTool)
        +to_params()
    }

    BaseTool <|-- PlanningTool
    BaseTool --o ToolCollection
    BaseTool --> ToolResult
```



## 数据流图

```mermaid
graph LR
    A["用户输入"] --> B["Agent/Flow"]
    B --> C["LLM 处理"]
    C --> D["工具调用"]
    D --> E["工具执行结果"]
    E --> C
    C --> F["最终输出"]
    F --> G["用户"]

    subgraph "内部状态"
        H["Memory"] <--> B
        I["Plan"] <--> B
        J["AgentState"] <--> B
    end
```


## 提示词
### 系统提示词

```
SYSTEM_PROMPT = (
    "你是OpenManus，一个全能的AI助手，旨在解决用户提出的任何任务。你拥有各种工具可以调用，以高效完成复杂的请求。无论是编程、信息检索、文件处理还是网页浏览，你都能胜任。"
    "初始目录是：{directory}"
)
```
### 用户提示词
#### 下一步提示词

```
NEXT_STEP_PROMPT = """
根据用户需求，主动选择最合适的工具或工具组合。对于复杂任务，你可以将问题分解，并逐步使用不同的工具来解决它。在使用每个工具后，清晰地解释执行结果并建议下一步操作。
"""
BROWSER_NEXT_STEP_PROMPT（如果最近的三个message中有使用BROWSER） = """
What should I do next to achieve my goal?

When you see [Current state starts here], focus on the following:
- Current URL and page title{url_placeholder}
- Available tabs{tabs_placeholder}
- Interactive elements and their indices
- Content above{content_above_placeholder} or below{content_below_placeholder} the viewport (if indicated)
- Any action results or errors{results_placeholder}

For browser interactions:
- To navigate: browser_use with action="go_to_url", url="..."
- To click: browser_use with action="click_element", index=N
- To type: browser_use with action="input_text", index=N, text="..."
- To extract: browser_use with action="extract_content", goal="..."
- To scroll: browser_use with action="scroll_down" or "scroll_up"

Consider both what's visible and what might be beyond the current viewport.
Be methodical - remember your progress and what you've learned so far.
"""
```


## ToolCallAgent
react模式的进行工具调用的Agent, 其act方法只负责工具的执行, 不负责思考

## Message的类型
- system: 系统消息
- user: 用户消息
- assistant: 助手消息
- tool: 工具消息

