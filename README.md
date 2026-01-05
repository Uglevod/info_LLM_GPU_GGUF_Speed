# info_LLM_GPU_GGUF_Speed
Info about what LLM in what GPU how work with GPU model

Test promt:"write full rich fastapi todo app"
## 6XP104-100 8GB                                             
| Backend | Model | Size | Quant | Speed | Rating | Reasoning | Comment | CanOutOfBoxCodeRun |
|---------|-------|------|-------|-------|--------|-----------|---------|---------------------|
| llama.cpp | Qwen3-Coder-30B-A3B-Instruct-UD | 17GB | Q4_K_XL | 28.5 t/s | *** |   | Write single file |                     |
| llama.cpp | qwen2.5-coder-32b-instruct | 18GB | q4_0 | 10.2 t/s | **** |   | Write multiple files |                     |
| llama.cpp | IQuest-Coder-V1-40B-Instruct | 22GB | q4_k_m | 7 t/s | ***** |   | Write multiple files |                     |
| llama.cpp | DeepCoder-14B-preview | 9GB | q5_0 | 17 t/s | ***  |   |       |                     |



## RTX3060 12GB
| Backend | Model | Size | Quant | Speed | Rating | Reasoning | Comment | CanOutOfBoxCodeRun |
|---------|-------|------|-------|-------|--------|-----------|---------|---------------------|
| LMStudio | DeepCoder-14B-preview | 9GB | q5_0 | 29 t/s | ***** |  Yes | add React Frontend !) | |

