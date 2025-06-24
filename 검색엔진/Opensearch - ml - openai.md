POST /_plugins/_ml/model_groups/_register
{
  "name": "remote_model_group",
  "description": "A model group for external models"
}
# 2M2DoJcBZGLRABg107Le

POST /_plugins/_ml/connectors/_create
{
    "name": "OpenAI Chat Connector",
    "description": "The connector to public OpenAI model service for GPT 3.5",
    "version": 1,
    "protocol": "http",
    "parameters": {
        "endpoint": "api.openai.com",
        "model": "gpt-3.5-turbo"
    },
    "credential": {
        "openAI_key": "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"
    },
    "actions": [
        {
            "action_type": "predict",
            "method": "POST",
            "url": "https://${parameters.endpoint}/v1/chat/completions",
            "headers": {
                "Authorization": "Bearer ${credential.openAI_key}"
            },
            "request_body": "{ \"model\": \"${parameters.model}\", \"messages\": ${parameters.messages} }"
        }
    ]
}

# 3M2EoJcBZGLRABg1kLKQ

GET /_plugins/_ml/tasks/4M2FoJcBZGLRABg1I7LX


POST /_plugins/_ml/models/_register
{
    "name": "openAI-gpt-3.5-turbo",
    "function_name": "remote",
    "model_group_id": "2M2DoJcBZGLRABg107Le",
    "description": "test model",
    "connector_id": "3M2EoJcBZGLRABg1kLKQ"
}

{
  "task_id": "4M2FoJcBZGLRABg1I7LX",
  "status": "CREATED",
  "model_id": "4c2FoJcBZGLRABg1JLJB"
}

POST /_plugins/_ml/models/4c2FoJcBZGLRABg1JLJB/_deploy

{
  "task_id": "682GoJcBZGLRABg1d7KI",
  "task_type": "DEPLOY_MODEL",
  "status": "COMPLETED"
}


POST /_plugins/_ml/models/4c2FoJcBZGLRABg1JLJB/_predict
{
  "parameters": {
    "messages": [
      {
        "role": "system",
        "content": "너는 한국인이야."
      },
      {
        "role": "user",
        "content": "안녕?"
      }
    ]
  }
}
