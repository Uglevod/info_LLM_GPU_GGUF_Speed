# info_LLM_GPU_GGUF_Speed
Info about what LLM in what GPU how good work with GGUF model

Test promt:"write full rich fastapi todo app"
## some of P104-100 8GB                                             
| Backend   | Model                                   | SizeB | Quant     | Size |   t/s | Rating | Reasoning | Comment              | CanOutOfBoxCodeRun |
|-----------|-----------------------------------------|-------|-----------|------|-------|--------|-----------|----------------------|---------------------|
| llama.cpp | Qwen3-Next-80B-A3B-Thinking-UD          | 80    | Q4_k_XL   | 43GB | 17    | *****  |           | Multy fl, but wo HTML. Чутка огорчила по исправлению кода по ставнению с gpt-oss-20b|                      |
| llama.cpp | IQuest-Coder-V1-40B-Instruct           | 40    | q4_k_m    | 22GB | 7     | *****   |           | Write multiple files |                     |
| llama.cpp | qwen2.5-coder-32b-instruct             | 32    | q4_0      | 18GB | 10.2  | ****   |           | Write multiple files |                     |
| llama.cpp | Qwen3-Coder-30B-A3B-Instruct-UD         | 30    | Q4_K_XL   | 17GB | 28.5  | ***    |           | Write single file |                     |
| llama.cpp | gpt-oss-20b-MXFP4.gguf                 | 20    | Q4_K_XL   | 12B  | 32    | *****   |           | Multy + make dokerfile   |                     |
| llama.cpp | DeepCoder-14B-preview                  | 14    | q5_0      | 9GB  | 17    | ***    |     in this task -yes      |                    |                     |


## RTX3060 12GB
| Backend   | Model                 | SizeB | Quant | Size  |   t/s  | Rating | Reasoning | Comment               | CanOutOfBoxCodeRun |
|-----------|-----------------------|-------|-------|-------|--------|--------|-----------|-----------------------|--------------------|
| LMStudio  | DeepCoder-14B-preview | 14    | q5_0  | 9GB   | 29     | *****  |  Yes      | add React Frontend !) |                    |

