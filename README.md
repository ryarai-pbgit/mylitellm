# LiteLLMの調査

### 参考ドキュメント
https://docs.litellm.ai/docs/

### 1. 概要
LiteLLMをAKS上で稼働させて、Azure OpenAIのプロキシとして動作させてみる。構成図は下記の通り。
![](image/Arch.png)

### 2. 環境構築
AKSは簡易にポータルから構築。
![](image/aks.png)

管理画面を立ち上げる際にデータベースが必要なようなのでPostgreも調達。
![](image/postgresql.png)

LLMはAzure OpenAIでo4-miniを調達。
![](image/openai.png)

デプロイメントの定義（deployment.yaml）は下記の通り。
```
---
apiVersion: v1
kind: Service
metadata:
  name: litellm-service
spec:
  selector:
    app: litellm
  ports:
    - protocol: TCP
      port: 80
      targetPort: 4000
  type: LoadBalancer
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: litellm-config-file
data:
  config.yaml: |
      model_list: 
        - model_name: o4-mini
          litellm_params:
            model: o4-mini
            api_base: https://myopenai20250920.openai.azure.com/
            api_version: "2025-01-01-preview"
            api_key: XXX
      general_settings:
        master_key: {my_master_key}
        database_url: "postgresql://litellm:{password}@mypg.postgres.database.azure.com:5432/postgres"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: litellm-deployment
  labels:
    app: litellm
spec:
  selector:
    matchLabels:
      app: litellm
  template:
    metadata:
      labels:
        app: litellm
    spec:
      containers:
      - name: litellm
        image: ghcr.io/berriai/litellm:main-stable # it is recommended to fix a version generally
        args:
          - "--config"
          - "/app/proxy_server_config.yaml"
        ports:
        - containerPort: 4000
        volumeMounts:
        - name: config-volume
          mountPath: /app/proxy_server_config.yaml
          subPath: config.yaml
      volumes:
        - name: config-volume
          configMap:
            name: litellm-config-file

```

### 3.動作確認
まずはcurlでLiteLLM越しにOpenAIを呼び出してみる。
```
curl --location 'http://130.33.162.78/chat/completions' \
    --header "Authorization: Bearer {my_master_key}" \
    --header 'Content-Type: application/json' \
    --data '{
    "model": "o4-mini",
    "messages": [
        {
        "role": "user",
        "content": "what llm are you"
        }
    ]
}'
```
応答はこちら
```
{
    "id": "chatcmpl-CHr8WRLSX5TzWqrnIJvdp8iJquX3B",
    "created": 1758371868,
    "model": "azure/o4-mini-2025-04-16",
    "object": "chat.completion",
    "choices": [
        {
            "finish_reason": "stop",
            "index": 0,
            "message": {
                "content": "I’m ChatGPT, a large language model developed by OpenAI, and I’m based on the GPT-4 architecture.",
                "role": "assistant",
                "annotations": []
            },
            "provider_specific_fields": {
                "content_filter_results": {
                    "hate": {
                        "filtered": false,
                        "severity": "safe"
                    },
                    "protected_material_code": {
                        "detected": false,
                        "filtered": false
                    },
                    "protected_material_text": {
                        "detected": false,
                        "filtered": false
                    },
                    "self_harm": {
                        "filtered": false,
                        "severity": "safe"
                    },
                    "sexual": {
                        "filtered": false,
                        "severity": "safe"
                    },
                    "violence": {
                        "filtered": false,
                        "severity": "safe"
                    }
                }
            }
        }
    ],
    "usage": {
        "completion_tokens": 172,
        "prompt_tokens": 11,
        "total_tokens": 183,
        "completion_tokens_details": {
            "accepted_prediction_tokens": 0,
            "audio_tokens": 0,
            "reasoning_tokens": 128,
            "rejected_prediction_tokens": 0
        },
        "prompt_tokens_details": {
            "audio_tokens": 0,
            "cached_tokens": 0
        }
    },
    "prompt_filter_results": [
        {
            "prompt_index": 0,
            "content_filter_results": {
                "hate": {
                    "filtered": false,
                    "severity": "safe"
                },
                "jailbreak": {
                    "detected": false,
                    "filtered": false
                },
                "self_harm": {
                    "filtered": false,
                    "severity": "safe"
                },
                "sexual": {
                    "filtered": false,
                    "severity": "safe"
                },
                "violence": {
                    "filtered": false,
                    "severity": "safe"
                }
            }
        }
    ]
}
```

次に管理画面を確認。先ほどのリクエスト、レスポンスが記録されていることがわかる。
![](image/admin.png)