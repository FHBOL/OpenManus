# OpenManus 项目详细架构图

本文档使用 Mermaid 图表详细展示 OpenManus 项目的架构、执行流程、类结构和组件交互。

## 项目整体架构

```mermaid
graph TD
    subgraph "入口文件"
        A1["main.py"] --> |"直接使用"| B1["Manus Agent"]
        A2["run_flow.py"] --> |"使用Flow"| B2["Flow Factory"]
        A3["run_mcp.py"] --> B3["MCP Agent"]
        A4["run_mcp_server.py"] --> B4["MCP Server"]
    end

    subgraph "核心模块"
        B2 --> |"创建"| C1["Planning Flow"]
        C1 --> |"使用"| B1
        C1 --> |"使用"| C2["Planning Tool"]
        B1 --> |"使用"| C3["Tool Collection"]
        C3 --> |"包含"| C4["各种工具"]
    end

    subgraph "Agent 层次结构"
        D2["ToolCallAgent"] --> D1["BaseAgent"]
        B1["Manus Agent"] --> D2
        D3["ReactAgent"] --> D1
        D4["SWEAgent"] --> D1
        B3["MCP Agent"] --> D1
    end

    subgraph "Flow 层次结构"
        C1["Planning Flow"] --> E1["BaseFlow"]
    end

    subgraph "Tool 层次结构"
        F2["PlanningTool"] --> F1["BaseTool"]
        F3["PythonExecute"] --> F1
        F4["BrowserUseTool"] --> F1
        F5["StrReplaceEditor"] --> F1
        F6["Terminate"] --> F1
        F7["BashTool"] --> F1
        F8["WebSearchTool"] --> F1
        F9["DeepResearchTool"] --> F1
    end

    subgraph "Prompt 模板"
        G1["系统提示"] --> G2["Manus Prompts"]
        G1 --> G3["Planning Prompts"]
        G1 --> G4["Browser Prompts"]
        G1 --> G5["SWE Prompts"]
        G1 --> G6["MCP Prompts"]
    end

    subgraph "外部集成"
        H1["LLM 接口"] --> H2["各种模型支持"]
        H3["Sandbox"] --> H4["安全执行环境"]
    end

    B1 --> C3
    B1 --> G2
    C1 --> G3
    C1 --> C2
    B1 --> H1
    C1 --> H1
```

## 详细执行流程

### main.py 执行流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Main as main.py
    participant Manus as Manus Agent
    participant LLM as 语言模型
    participant Tools as 工具集合

    User->>Main: 输入提示(prompt)
    Main->>Manus: 创建 Manus 实例
    Main->>Manus: 调用 run(prompt)
    activate Manus
    Manus->>Manus: 初始化内存和状态
    Manus->>LLM: 发送系统提示和用户输入
    activate LLM
    LLM-->>Manus: 返回初始响应
    deactivate LLM

    loop 执行步骤 (最多max_steps次)
        Manus->>Manus: 更新状态为THINKING
        Manus->>LLM: 发送next_step_prompt
        activate LLM
        LLM-->>Manus: 返回下一步操作(工具调用)
        deactivate LLM
        Manus->>Manus: 更新状态为EXECUTING
        Manus->>Tools: 执行工具调用
        activate Tools
        Tools-->>Manus: 返回工具执行结果
        deactivate Tools
        Manus->>Manus: 更新内存
        alt 任务完成
            Manus->>Manus: 更新状态为DONE
            Manus->>Main: 返回最终结果
        end
    end
    deactivate Manus
    Main->>User: 显示结果
```

### run_flow.py 执行流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant RunFlow as run_flow.py
    participant Factory as FlowFactory
    participant Flow as PlanningFlow
    participant Agent as Manus Agent
    participant PlanTool as PlanningTool
    participant LLM as 语言模型

    User->>RunFlow: 输入提示(prompt)
    RunFlow->>Agent: 创建 Manus 实例
    RunFlow->>Factory: 调用 create_flow(PLANNING)
    Factory->>Flow: 创建 PlanningFlow 实例
    RunFlow->>Flow: 调用 execute(prompt)
    activate Flow

    Flow->>LLM: 分析任务并创建计划
    activate LLM
    LLM-->>Flow: 返回初始计划
    deactivate LLM
    Flow->>PlanTool: 创建计划(create)
    PlanTool-->>Flow: 返回计划ID

    loop 执行计划步骤
        Flow->>Flow: 选择下一个未完成步骤
        Flow->>Agent: 获取执行器
        activate Agent
        Flow->>PlanTool: 更新步骤状态(in_progress)

        Agent->>LLM: 分析步骤并决定操作
        LLM-->>Agent: 返回工具调用
        Agent->>Agent: 执行工具调用
        Agent-->>Flow: 返回步骤执行结果
        deactivate Agent

        Flow->>PlanTool: 更新步骤状态(completed)
        alt 所有步骤完成
            Flow->>Flow: 生成最终总结
            Flow->>RunFlow: 返回执行结果
        end
    end
    deactivate Flow

    RunFlow->>User: 显示结果
```

## 详细类图

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
        +step()*
        +think()*
    }

    class ToolCallAgent {
        +available_tools: ToolCollection
        +special_tool_names: list
        +max_observe: int
        +execute_tool_call()
        +observe()
        +step()
        +think()
    }

    class Manus {
        +name: str = "Manus"
        +description: str
        +system_prompt: str
        +next_step_prompt: str
        +max_observe: int = 10000
        +max_steps: int = 20
        +available_tools: ToolCollection
        +special_tool_names: list
        +browser_context_helper: BrowserContextHelper
        +initialize_helper()
        +think()
        +cleanup()
    }

    class ReactAgent {
        +name: str
        +description: str
        +system_prompt: str
        +next_step_prompt: str
        +step()
        +think()
    }

    class SWEAgent {
        +name: str
        +description: str
        +system_prompt: str
        +next_step_prompt: str
        +step()
        +think()
    }

    class MCPAgent {
        +name: str
        +description: str
        +system_prompt: str
        +next_step_prompt: str
        +step()
        +think()
    }

    ReactAgent <|-- ToolCallAgent
    BaseAgent <|-- ReactAgent
    BaseAgent <|-- SWEAgent
    BaseAgent <|-- MCPAgent
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
        -create_initial_plan(input_text: str)
        -execute_plan_steps()
        -format_plan_for_display()
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
        +__bool__()
        +__add__(other: ToolResult)
        +__str__()
        +replace()
    }

    class BashTool {
        +execute()
    }

    class PlanningTool {
        +name: str = "planning"
        +description: str
        +parameters: dict
        +plans: dict
        +_current_plan_id: str
        +execute()
        -_create_plan()
        -_update_plan()
        -_list_plans()
        -_get_plan()
        -_set_active_plan()
        -_mark_step()
        -_delete_plan()
    }

    class PythonExecute {
        +name: str
        +description: str
        +parameters: dict
        +execute()
    }

    class BrowserUseTool {
        +name: str
        +description: str
        +parameters: dict
        +execute()
    }

    class StrReplaceEditor {
        +name: str
        +description: str
        +parameters: dict
        +execute()
    }

    class Terminate {
        +name: str
        +description: str
        +parameters: dict
        +execute()
    }

    class ToolCollection {
        +tools: List[BaseTool]
        +get_tool(name: str)
        +add_tool(tool: BaseTool)
        +to_params()
    }

    BaseTool <|-- PlanningTool
    BaseTool <|-- PythonExecute
    BaseTool <|-- BrowserUseTool
    BaseTool <|-- StrReplaceEditor
    BaseTool <|-- Terminate
    BaseTool --o ToolCollection
    BaseTool --> ToolResult
```

### function调用原理

``` python
def to_param(self) -> Dict:
        """Convert tool to function call format."""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            },
        }
@abstractmethod
    async def execute(self, **kwargs) -> Any:
        """Execute the tool with given parameters."""
```

每个tool类型通过to_param方法转换为function call格式，然后作为参数传递给LLM。LLM根据用户输入和可用工具的参数，选择合适的工具进行调用。并且LLM会根据工具的参数类型，自动生成参数的schema，然后调用调用tool的execute方法。所以开发一个新的tool类型，只需要实现to_param所需的name、description和parameters（遵循openai协议）以及execute方法即可。
参考：https://github.com/mannaandpoem/OpenManus/blob/main/app/tool/bash.py

### mcp server调用原理

通过请求mcp server的api，mcp server 会返回一个json格式的response，response中包含了tool的name、description和parameters。这样就可以根据function call的原理，向模型发送这些信息，模型选择合适toolname和参数，将其发送给mcp，然后mcp server会根据tool的name，调用对应的tool的execute方法。
参考：https://github.com/mannaandpoem/OpenManus/blob/main/app/tool/mcp.py

## 数据流图

```mermaid
graph TD
    A["用户输入"] --> B["Agent/Flow 处理"]

    subgraph "处理流程"
        B --> C["LLM 分析与决策"]
        C --> D["工具调用"]
        D --> E["工具执行结果"]
        E --> C
    end

    subgraph "内部状态管理"
        F["Memory"] <--> B
        G["Plan 状态"] <--> B
        H["Agent 状态"] <--> B
    end

    subgraph "外部资源"
        I["文件系统"] <--> D
        J["网络资源"] <--> D
        K["Python 解释器"] <--> D
    end

    C --> L["最终输出"]
    L --> M["用户"]
```

## 组件交互图

```mermaid
graph TB
    subgraph "用户交互层"
        A1["命令行界面"] --> B1["main.py"]
        A1 --> B2["run_flow.py"]
        A1 --> B3["run_mcp.py"]
    end

    subgraph "Agent层"
        B1 --> C1["Manus Agent"]
        B2 --> C2["Flow Factory"]
        B3 --> C3["MCP Agent"]
        C2 --> C4["Planning Flow"]
        C4 --> C1
    end

    subgraph "工具层"
        C1 --> D1["Tool Collection"]
        D1 --> D2["Python Execute"]
        D1 --> D3["Browser Use Tool"]
        D1 --> D4["Str Replace Editor"]
        D1 --> D5["Terminate"]
        C4 --> D6["Planning Tool"]
    end

    subgraph "LLM层"
        C1 --> E1["LLM Interface"]
        C4 --> E1
        C3 --> E1
        E1 --> E2["Model Providers"]
    end

    subgraph "执行环境"
        D2 --> F1["Sandbox"]
        D3 --> F2["Browser"]
        D4 --> F3["File System"]
    end
```

## 项目目录结构

```mermaid
graph TD
    A["OpenManus/"] --> B["app/"]
    A --> C["config/"]
    A --> D["examples/"]
    A --> E["tests/"]
    A --> F["main.py"]
    A --> G["run_flow.py"]
    A --> H["run_mcp.py"]
    A --> I["run_mcp_server.py"]

    B --> B1["agent/"]
    B --> B2["flow/"]
    B --> B3["prompt/"]
    B --> B4["tool/"]
    B --> B5["sandbox/"]
    B --> B6["mcp/"]
    B --> B7["__init__.py"]
    B --> B8["config.py"]
    B --> B9["llm.py"]
    B --> B10["logger.py"]
    B --> B11["schema.py"]
    B --> B12["exceptions.py"]

    B1 --> BA1["__init__.py"]
    B1 --> BA2["base.py"]
    B1 --> BA3["manus.py"]
    B1 --> BA4["react.py"]
    B1 --> BA5["swe.py"]
    B1 --> BA6["mcp.py"]
    B1 --> BA7["toolcall.py"]
    B1 --> BA8["browser.py"]

    B2 --> BB1["__init__.py"]
    B2 --> BB2["base.py"]
    B2 --> BB3["flow_factory.py"]
    B2 --> BB4["planning.py"]

    B3 --> BC1["__init__.py"]
    B3 --> BC2["manus.py"]
    B3 --> BC3["planning.py"]
    B3 --> BC4["browser.py"]
    B3 --> BC5["swe.py"]
    B3 --> BC6["mcp.py"]
    B3 --> BC7["toolcall.py"]
    B3 --> BC8["cot.py"]

    B4 --> BD1["__init__.py"]
    B4 --> BD2["base.py"]
    B4 --> BD3["planning.py"]
    B4 --> BD4["python_execute.py"]
    B4 --> BD5["browser_use_tool.py"]
    B4 --> BD6["str_replace_editor.py"]
    B4 --> BD7["terminate.py"]
    B4 --> BD8["bash.py"]
    B4 --> BD9["web_search.py"]
    B4 --> BD10["deep_research.py"]
    B4 --> BD11["tool_collection.py"]
```
