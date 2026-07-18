# Attention Backends
Links :
- https://docs.vllm.ai/en/latest/design/attention_backends/
- https://vllm.ai/blog/2026-03-04-vllm-triton-backend-deep-dive
- https://pytorch.org/blog/enabling-vllm-v1-on-amd-gpus-with-triton/
# Triton
In vLLM, the attention backend is the GPU execution layer responsible for attention computation and KV-cache access. Triton may be used to implement some of those optimized GPU kernels.



# Definitions
- kv_cache - key-value cache; data structures allocated in VRAM;
    - K: key vector
    - V: value vector
    - Distinction: 
        - GPU hardware cache: L1/L2 cache, managed mostly by hardwar
        - LLM KV cache: tensors stored mainly in GPU VRAM, managed by inference software
    - demo flow: Text → tokenization → token IDs → model computation → K/V vectors → KV cache
    - Tokenization decides what the tokens are. KV-cache paging decides how the computed data for those tokens is organized in GPU memory.
    - LLM inference also relies heavily on key value (KV) cache, the short-term memory of an LLM, to store intermediate results
- prefill phase - prompt processing; creates the first KV cache enries
- decode phase - token generation; memory bandwidth bound, involves reading and writing to the KV cache with less compute