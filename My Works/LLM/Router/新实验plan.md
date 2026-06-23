Dataset: 
- 两个设置共有的数据集：
	- AIME — 竞赛数学，即使旗舰模型也难以全对，两个设置都有区分度
	- LiveCodeBench — 时新的编程题，避免数据泄露，两级模型都有差异
	- GPQA — 研究生级别知识题，对两级模型都有挑战
	- MMLU-Pro — 大规模知识测试（本地用1000题，云端用3000题）
- +一个cloud模型更擅长的 arenahard
- +一个本地模型也能完成的livemathbench

LLM:
- qwen3-coder-next 
- qwen3.5-35b-a3b

embedding: 
- 现有的 text-embedding=3-larde 
- Clustering Task Sota: F2LLM 4B [codefuse-ai/F2LLM-4B · Hugging Face](https://huggingface.co/codefuse-ai/F2LLM-4B) 
- MTEB Sota: KaLM-Embedding-V2.5 494M [KaLM-Embedding/KaLM-embedding-multilingual-mini-instruct-v2.5 · Hugging Face](https://huggingface.co/KaLM-Embedding/KaLM-embedding-multilingual-mini-instruct-v2.5)

