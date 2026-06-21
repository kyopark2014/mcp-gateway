# MCP Gateway

여기에서는 MCP Gateway를 이용해 MCP 서버를 구현하고 관리하는 방법에 대해 설명합니다. 전체적인 architecture는 아래와 같습니다. 사용자가 [streamlit](https://streamlit.io/)으로 구현된 생성형 AI application을 통해 질문을 입력하면, [LangGraph](https://langchain-ai.github.io/langgraph/) 또는 [Strands](https://strandsagents.com/latest/documentation/docs/) agent가 mutli step reasoning을 통해, Kubernetes로 된 중요한 workload를 조회하거나 관리할 수 있고, 사내의 중요한 데이터를 RAG를 이용해 활용할 수 있습니다. 여기에서는 Knowledge base를 조회하는 [kb-retriever](./gateway/kb-retriever/lambda-kb-retriever-for-mcp/lambda_function.py)와 AWS 인프라를 관리할 수 있는 [use-aws](./gateway/use-aws/lambda-use-aws-for-mcp/lambda_function.py)를 MCP tool로 제공하며, 이 tool들은 AgentCore에 runtime 또는 AgentCore Gateway로 배포됩니다.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/e5305fd3-ac8c-4cfa-9bf8-2e8e114d42d3" />

## MCP tool

### AWS 인프라 관리: use-aws

MCP client가 AgentCore의 gateway로 MCP capability에 대한 discovery 요청하면, use-aws에 대한 [tool_spec.json](./gateway/use-aws/tool_spec.json)을 전달합니다. Agent가 description을 보고 use_aws tool을 선택하면, 아래의 inputSchema에 해당하는 parameter 값을 설정하여 gateway로 요청합니다. 

```java
{
    "name": "use_aws",
    "description": "Make a boto3 client call with the specified service, operation, and parameters. Boto3 operations are snake_case.",
    "inputSchema": {
        "type": "object",
        "properties": {
            "service_name": {
                "type": "string",
                "description": "The name of the AWS service"
            },
            "operation_name": {
                "type": "string",
                "description": "The name of the operation to perform"
            },
            "parameters": {
                "type": "object",
                "description": "The parameters for the operation"
            },
            "region": {
                "type": "string",
                "description": "Region name for calling the operation on AWS boto3"
            },
            "label": {
                "type": "string",
                "description": "Label of AWS API operations human readable explanation. This is useful for communicating with human."
            },
            "profile_name": {
                "type": "string",
                "description": "Optional: AWS profile name to use from ~/.aws/credentials. Defaults to default profile if not specified."
            }
        },
        "required": [
            "service_name",
            "operation_name",
            "parameters",
            "label"
        ]
    }
}
```

[use_aws](./gateway/use-aws/lambda-use-aws-for-mcp/lambda_function.py) tool이 선택되어 lambda가 trigger되면, 아래와 같이 event에서 tool, service, operation, parameters, region, label, profile과 같은 parameter가 전달되고, lambda는 tool 이름이 'use_aws'인 경우에 use_aws 함수를 실행합니다.

```python
def lambda_handler(event, context):
    toolName = context.client_context.custom['bedrockAgentCoreToolName']
    service_name = event.get('service_name')
    operation_name = event.get('operation_name')
    parameters = event.get('parameters')
    region = event.get('region')
    label = event.get('label')
    profile_name = event.get('profile_name')
    
    if toolName == 'use_aws':
        body = use_aws(service_name, operation_name, parameters, region, label, profile_name)
        print(f"body: {body}")
        return {
            'statusCode': 200, 
            'body': json.dumps(body)            
        }
```

use_aws 함수는 boto3로 전달받은 operation을 수행하고 결과를 리턴합니다.

```python
def get_boto3_client(service_name: str, region_name: str, profile_name: Optional[str] = None) -> Any:
    session = boto3.Session(profile_name=profile_name)
    return session.client(service_name=service_name, region_name=region_name)

def use_aws(
    service_name: str,
    operation_name: str,
    parameters: Dict[str, Any],
    region: Optional[str] = None,
    label: str = "AWS Operation Details",
    profile_name: Optional[str] = None
) -> Dict[str, Any]:
    client = get_boto3_client(service_name, region, profile_name)
    operation_method = getattr(client, operation_name)

    response = operation_method(**parameters)
    response = handle_streaming_body(response)
    response = aws_utils.convert_datetime_to_str(response)

    return {
        "status": "success",
        "content": [{"text": f"Success: {str(response)}"}],
    }
```

### RAG의 활용: kb-retriever

MCP client가 gateway를 통해 tool 정보를 요청하면, 아래와 같은 [kb-retriever에 대한 tool 정보](./gateway/kb-retriever/tool_spec.json)가 전달됩니다. 이때 description을 보고 retrieve tool이 선택되면, inputSchema의 keyword를 설정하여 kb-retriever를 호출하게 됩니다.

```java
{
    "name": "retrieve",
    "description": "keyword to retrieve the knowledge base",
    "inputSchema": {
        "type": "object",
        "properties": {
            "keyword": {
                "type": "string"
            }
        },
        "required": ["keyword"]
    }
}
```

[kb-retriever](./gateway/kb-retriever/lambda-kb-retriever-for-mcp/lambda_function.py)에서는 완전관리형 RAG 서비스인 Knowledge base의 정보를 조회합니다. 이를 위해 lambda는 event에서 tool 이름과 paramter인 keyword를 추출하여 retrieve 함수를 호출합니다.

```python
def lambda_handler(event, context):
    toolName = context.client_context.custom['bedrockAgentCoreToolName']
    delimiter = "___"
    if delimiter in toolName:
        toolName = toolName[toolName.index(delimiter) + len(delimiter):]

    keyword = event.get('keyword')
    if toolName == 'retrieve':
        result = retrieve(keyword)
        return {
            'statusCode': 200, 
            'body': result
        }
```

[kb-retriever](./gateway/kb-retriever/lambda-kb-retriever-for-mcp/lambda_function.py)의 retrieve 함수는 아래와 같이 knowledge base에서 keyworkd에 대한 vector 검색을 수행하고 결과에서 text, location 정보를 추출하여 리턴합니다.

```python
def retrieve(query: str) -> str:
    response = bedrock_agent_runtime_client.retrieve(
        retrievalQuery={"text": query},
        knowledgeBaseId=knowledge_base_id,
            retrievalConfiguration={
                "vectorSearchConfiguration": {"numberOfResults": number_of_results},
            },
        )    
    retrieval_results = response.get("retrievalResults", [])

    json_docs = []
    for result in retrieval_results:
        text = url = name = None
        if "content" in result:
            content = result["content"]
            if "text" in content:
                text = content["text"]
        if "location" in result:
            location = result["location"]
            if "s3Location" in location:
                uri = location["s3Location"]["uri"] if location["s3Location"]["uri"] is not None else ""                
                name = uri.split("/")[-1]                
            elif "webLocation" in location:
                url = location["webLocation"]["url"] if location["webLocation"]["url"] is not None else ""
                name = "WEB"
        json_docs.append({
            "contents": text,              
            "reference": {
                "url": url,                   
                "title": name,
                "from": "RAG"
            }
        })
    return json.dumps(json_docs, ensure_ascii=False)
```


## AgentCore Gateway

AgentCore의 Gateway를 이용하면 Lambda로 Streamable HTTP 방식의 MCP 서버를 만들어 편리하게 이용할 수 있습니다. 

### AgentCore Gateway의 Role 생성

[create_gateway_role.py](./gateway/kb-retriever/create_gateway_role.py)를 이용해 아래와 같이 AgentCore Gateway를 위한 Role을 배포할 수 있습니다.

```text
python create_gateway_role.py
```

이때 동작은 Policy와 Role을 순차적으로 생성하고 config.json에 관련된 정보를 업데이트하는 과정을 수행합니다. Policy에는 Lambda, AgentCore, Secret, Cognito, ECR, CloudWatch에 대한 권한을 포함하여야 합니다. 상세한 권한은 [create_gateway_role.py](./gateway/kb-retriever/create_gateway_role.py)을 참조합니다. 

Policy 생성의 상세 코드는 아래와 같습니다. 

```python
iam_client = boto3.client('iam')
response = iam_client.create_policy(
    PolicyName=policy_name,
    PolicyDocument=json.dumps(policy_document),
    Description=policy_description
)
return response['Policy']['Arn']
```

생성된 policy로 아래와 같이 role을 생성합니다.

```python
iam_client = boto3.client('iam')
response = iam_client.create_role(
    RoleName=role_name,
    AssumeRolePolicyDocument=json.dumps(trust_policy),
    Description="Role for Bedrock AgentCore MCP access"
)
iam_client.attach_role_policy(
    RoleName=role_name,
    PolicyArn=policy_arn
)
role_arn = response['Role']['Arn']
```


### AgentCore Gateway를 이용해 MCP 서버 배포하기

#### AgentCore Gateway 생성

[create_gateway_tool.py](./gateway/kb-retriever/create_gateway_tool.py)를 이용해 MCP server인 target을 AgentCore Gateway에 배포할 수 있습니다.

```python
python create_gateway_tool.py
```

Gateway에 배포를 위해서는 bearer token이 필요합니다. 이 token은 먼저 secret에 이미 저장된 값을 먼저 쓰고, 403같은 에러가 발생하면 Cognito를 접속해서 업데이트 합니다. 여기서는 편의상 secret_name으로 아래와 같이 project 이름을 이용하므로, gateway의 모든 target은 같은 secret을 이용합니다. 

```python
secret_name = f'{projectName.lower()}/credentials'

session = boto3.Session()
client = session.client('secretsmanager', region_name=region)
response = client.get_secret_value(SecretId=secret_name)
bearer_token_raw = response['SecretString']        
token_data = json.loads(bearer_token_raw)  
bearer_token = token_data['bearer_token']
```

인증에 필요한 client_id는 생성할때 config.json에 저장했다가 사용하거나 아래와 같이 client_name을 이용해 검색해서 사용할 수 있습니다. 

```python
gateway_client = boto3.client('bedrock-agentcore-control', region_name=region)
if not client_id:
    response = cognito_client.list_user_pool_clients(UserPoolId=user_pool_id)
    for client in response['UserPoolClients']:
        if client['ClientName'] == client_name:
            client_id = client['ClientId']
            cognito_config['client_id'] = client_id     
            break
```

AgentCore Gateway의 생성을 위해 미리 생성한 client_id, role을 활용합니다. 생성후에 Gateway ID와 URL은 config.json에 저장해 활용합니다.

```python
cognito_discovery_url = f'https://cognito-idp.{region}.amazonaws.com/{user_pool_id}/.well-known/openid-configuration'

client_id = cognito_config.get('client_id')
agentcore_gateway_iam_role = config['agentcore_gateway_iam_role']
auth_config = {
    "customJWTAuthorizer": { 
        "allowedClients": [client_id],  
        "discoveryUrl": cognito_discovery_url
    }
}
response = gateway_client.create_gateway(
    name=gateway_name,
    roleArn = agentcore_gateway_iam_role,
    protocolType='MCP',
    authorizerType='CUSTOM_JWT',
    authorizerConfiguration=auth_config, 
    description=f'AgentCore Gateway for {projectName}'
)
gateway_id = response["gatewayId"]
gateway_url = response["gatewayUrl"]
```

AgentCore Gateway에 접속하기 위한 경로는 아래와 같이 Gateway의 ID와 region 정보를 이용해 얻을 수도 있습니다. 

```python
gateway_url = f'https://{gateway_id}.gateway.bedrock-agentcore.{region}.amazonaws.com/mcp'
```

#### Lambda deployment

Gateway에서 MCP 서버의 동작은 lambda를 활용합니다. Lambda를 배포하기 위하여 아래와 같이 Lambda 폴더의 code들을 압축을 합니다. 이후 아래와 같이 기생성한 lambda role과 zip 파일의 코드를 이용해 lambda를 생성합니다. 

```python
os.makedirs(lambda_function_path)
create_dummpy_lambda_function(lambda_function_path)

with zipfile.ZipFile(lambda_function_zip_path, 'w', zipfile.ZIP_DEFLATED) as zip_file:     
    for root, dirs, files in os.walk(lambda_dir):
        for file in files:
            if file == 'lambda_function.zip':
                continue
            file_path = os.path.join(root, file)
            arcname = os.path.relpath(file_path, lambda_dir)
            zip_file.write(file_path, arcname)

lambda_client = boto3.client('lambda', region_name=region)
response = lambda_client.create_function(
    FunctionName=lambda_function_name,
    Runtime='python3.13',
    Handler='lambda_function.lambda_handler',
    Role=lambda_function_role,
    Description=f'Lambda function for {lambda_function_name}',
    Timeout=60,
    Code={
        'ZipFile': open(lambda_function_zip_path, 'rb').read()
    }
)
lambda_function_arn = response['FunctionArn']
```

### Target의 생성

아래와 같이 target을 생성합니다.

```python
credential_config = [ 
    {
        "credentialProviderType" : "GATEWAY_IAM_ROLE"
    }
]
response = gateway_client.create_gateway_target(
    gatewayIdentifier=gateway_id,
    name=targetname,
    description=f'{targetname} for {projectName}',
    targetConfiguration=lambda_target_config,
    credentialProviderConfigurations=credential_config)
target_id = response["targetId"]
```


### 배포 방법

여기에서는 Role 및 Tool을 배포하는 2가지의 python 파일을 이용하여 Runtime Gateway에 배포를 수행합니다. [create_gateway_role.py](./gateway/kb-retriever/create_gateway_role.py)와 같이 gateway를 위한 IAM role을 생성합니다. 

```text
python create_gateway_role.py
```

[create_gateway_tool.py](./gateway/kb-retriever/create_gateway_tool.py)와 같이 gateway에서 실행할 lambda에 대한 role을 생성하고, target을 배포합니다. 이때 기존 lambda가 없는 경우에 새로 생성합니다. 

```text
python create_gateway_tool.py
```

이때, use-aws와 같은 tool은 [lambda folder](./gateway/use-aws/lambda-use-aws-for-mcp)에 use-aws를 위한 colorama, rich, typing_extensions를 설치하여야 합니다. 이와 같은 패키지들은 아래와 같은 방법으로 folder에 설치합니다.

```text
pip install rich --target ./lambda-kb-retriever-for-mcp
```

이제 아래와 같이 [create_gateway_tool.py](./gateway/kb-retriever/test_mcp_remote.py)을 이용해 MCP 서버에 대한 동작을 테스트 할 수 있습니다.

```text
python test_mcp_remote.py
```

### Message Trim

LangGraph 에이전트([application/langgraph_agent.py](./application/langgraph_agent.py)의 `call_model`)는 LLM 호출 직전에 **HumanMessage 기준 최근 N턴**만 남깁니다. LangGraph state의 `messages`는 checkpointer에 그대로 두고, **모델에 넘기는 메시지만** trim합니다. `history_mode=Enable`/`Disable` 모두 동일하게 적용됩니다.

**기본값:** `MAX_CONTEXT_TURNS = 5` (일반 채팅의 `SimpleMemory(k=5)`와 동일한 “최근 5턴” 의도)

**설정 변경:**

- [application/langgraph_agent.py](./application/langgraph_agent.py)의 `MAX_CONTEXT_TURNS` 상수 수정
- 또는 [application/chat.py](./application/chat.py)의 `run_langgraph_agent()`에서 config의 `max_turns` / `configurable.max_turns` 지정
- `max_turns=0`이면 trim 비활성화

상수와 trim 함수는 `langgraph_agent.py`에 정의합니다.

```python
# application/langgraph_agent.py
MAX_CONTEXT_TURNS = 5


def trim_messages_by_human_turns(messages: list, max_turns: int) -> list:
    """Keep messages from the last N HumanMessage turns (inclusive)."""
    if max_turns <= 0 or not messages:
        return messages

    human_indices = [i for i, msg in enumerate(messages) if isinstance(msg, HumanMessage)]
    if len(human_indices) <= max_turns:
        return messages

    return messages[human_indices[-max_turns]:]
```

`call_model`에서는 trim 후 `chain.ainvoke(messages)`로 LLM을 호출합니다.

```python
# application/langgraph_agent.py — call_model() 내부
        messages = list(state["messages"])
        max_turns = (
            config.get("configurable", {}).get("max_turns")
            or config.get("max_turns")
            or MAX_CONTEXT_TURNS
        )
        trimmed = trim_messages_by_human_turns(messages, max_turns)
        if len(trimmed) < len(messages):
            logger.info(
                f"trimmed messages from {len(messages)} to {len(trimmed)} "
                f"(max_turns={max_turns})"
            )
            messages = trimmed

        prompt = ChatPromptTemplate.from_messages(
            [
                ("system", system),
                MessagesPlaceholder(variable_name="messages"),
            ]
        )
        chain = prompt | model
        response = await chain.ainvoke(messages)
```

에이전트 config는 `chat.py`의 `run_langgraph_agent()`에서 생성하며, `history_mode`와 관계없이 `max_turns`를 전달합니다.

```python
# application/chat.py — run_langgraph_agent()
    if history_mode == "Enable":
        app = langgraph_agent.buildChatAgentWithHistory(tools)
        config = {
            "recursion_limit": 50,
            "configurable": {"thread_id": user_id},
            "tools": tools,
            "system_prompt": None,
            "max_turns": langgraph_agent.MAX_CONTEXT_TURNS,
        }
    else:
        app = langgraph_agent.buildChatAgent(tools)
        config = {
            "recursion_limit": 50,
            "configurable": {"thread_id": user_id},
            "tools": tools,
            "system_prompt": None,
            "max_turns": langgraph_agent.MAX_CONTEXT_TURNS,
        }
```

**`max_turns=5`의 의미**

- **사용자 HumanMessage 5개**와, 각 턴에 이어진 **모든 후속 메시지**를 유지
- 1턴 = `HumanMessage` 1개 + 그 뒤의 `AIMessage`, `ToolMessage`, 도구 feedback loop 전체
- 도구를 여러 번 호출해도 **같은 사용자 질문이면 1턴**으로 카운트

**예 (도구 사용 포함)**

```
Human(Q1) → AI(tool_calls) → ToolMessage → AI(A1)
Human(Q2) → AI(A2)
Human(Q3) → AI(tool_calls) → ToolMessage → AI(A3)
```

`max_turns=2`이면 **Q2부터** 유지:

```
Human(Q2) → AI(A2) → Human(Q3) → AI(tool_calls) → ToolMessage → AI(A3)
```

**메시지 개수 trim과의 차이**

| 방식 | `N=5`일 때 |
|------|------------|
| 이전 (메시지 개수) | 메시지 객체 5개만 유지 → 도구 루프 때문에 사용자 턴 수가 불규칙 |
| 현재 (HumanMessage 턴) | 사용자 질문 5개 + 각 턴의 AI/Tool 응답 전체 유지 |

**Checkpointer와의 관계**

- `history_mode=Enable`일 때 `MemorySaver` checkpointer에는 **전체 대화 이력**이 저장됩니다.
- trim은 LLM 컨텍스트 윈도우 관리용이며, 저장된 history를 삭제하지 않습니다.
- CloudWatch(`/ecs/...`) 또는 애플리케이션 로그에서 `trimmed messages from X to Y (max_turns=5)`로 trim 여부를 확인할 수 있습니다.

## 실행 결과

Streamlit app을 아래와 같이 실행합니다. 

```python
streamlit run application/app.py
```

아래와 같이 AgentCore Gateway를 선택합니다.

<img width="384" height="350" alt="image" src="https://github.com/user-attachments/assets/9ec6381e-091d-47a4-99eb-93d817277835" />


"내 EKS 현황은?"라고 질문하면, 아래와 같이 use-aws tool을 이용하여 EKS의 상황을 조회할 수 있습니다. 

<img width="655" alt="image" src="https://github.com/user-attachments/assets/d4d2548d-0e60-4e57-b390-53a2ee02bd03" />

"보일러 에러 코드?"로 입력하면, Streamable HTTP를 지원하는 knowledge base MCP에 접속하여 관련된 문서를 아래와 같이 가져와서 답변할 수 있습니다.

<img width="655" alt="image" src="https://github.com/user-attachments/assets/e499b049-9f93-4136-a330-0dc679389b6d" />





## Reference 

[AWS Samples - Amazon Bedrock AgentCore Gateway](https://github.com/awslabs/amazon-bedrock-agentcore-samples/tree/main/01-tutorials/02-AgentCore-gateway)

[Hosting MCP Server on Amazon Bedrock AgentCore Runtime](https://github.com/awslabs/amazon-bedrock-agentcore-samples/blob/main/01-tutorials/01-AgentCore-runtime/02-hosting-MCP-server/hosting_mcp_server.ipynb)

[Bedrock AgentCore Starter Toolkit](https://github.com/aws/bedrock-agentcore-starter-toolkit)

[LangChain MCP Adapters](https://github.com/langchain-ai/langchain-mcp-adapters)

[Strands Agents](https://github.com/strands-agents/sdk-python)
