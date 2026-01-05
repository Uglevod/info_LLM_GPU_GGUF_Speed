# info_LLM_GPU_GGUF_Speed
Info about what LLM in what GPU how work with GPU model

Test promt:"write full rich fastapi todo app"
## 6XP104-100 8GB                                             
| Backend | Size | Model | Speed | Usage | Rating | Reasoning | Comment | CanOutOfBoxCodeRun |
|---------|------|-------|-------|-------|--------|-----------|---------|---------------------|
| llama.cpp | 17GB | Qwen3-Coder-30B-A3B-Instruct-UD-Q4_K_XL | 28.5 t/s | Write single file | *** |   |       |                     |
| llama.cpp | 18GB | qwen2.5-coder-32b-instruct-q4_0 | 10.2 t/s | Write multiple files | **** |   |       |                     |
| llama.cpp | 22GB | IQuest-Coder-V1-40B-Instruct.q4_k_m | 7 t/s | Write multiple files | ***** |   |       |                     |
| llama.cpp | 9GB | DeepCoder-14B-preview-q5_0 | 17 t/s |  |  |   |       |                     |



## RTX3060 12GB
LMStudio 9GB DeepCoder-14B-preview-q5_0 - 29 t\s 
