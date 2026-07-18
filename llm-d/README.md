Links:

---
What is llm-d and why we need it
---

- LLM inference also relies heavily on key value (KV) cache, the short-term memory of an LLM, to store intermediate results
- With Kubernetes, the current approach has been to deploy LLMs as monolithic containers–large, black boxes with no visibility or control. This, in addition to ignoring prompt structure, token count, response latency goals (Service Level Objectives or SLO), cache availability, and many other factors, makes it difficult to scale effectively.
- llm-d is making inference more efficient and more cost-effective
    - 3x improvement in time to first token
    - double throughput 
    - disaggregation (separates prefill and decode phase into indiv. workloads, called pods) 
- intelligent scheduling layer: more nuanced routing

- The llm-d modular architecture is made up of: 
    - Kubernetes: an open source container-orchestration platform that automates many of the manual processes involved in deploying, managing, and scaling containerized applications.
    - vLLM: an open source inference server that speeds up the outputs of gen AI applications.
    - Inference Gateway (IGW): a Kubernetes Gateway API extension that hosts features like model routing, serving priority, and “smart” load-balancing capabilities. 

## Links
- https://www.redhat.com/en/blog/what-llm-d-and-why-do-we-need-it
- https://www.redhat.com/en/topics/ai/what-is-llm-d
- Github: https://github.com/llm-d/llm-d
- Quickstart: https://llm-d.ai/docs/getting-started/quickstart
- https://www.redhat.com/en/engage/artificial-intelligence-for-enterprise-beginners-guide-ebook - some sales oriented pdf but high level info is useful