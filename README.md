# Agent-FineTuned
Full control over all major aspects of the LM agent.

Choose to use a vendor-provided LM language model, from your vendor of choice. 
Or use a LM that you have downloaded on your machine. Use your choice of environment for LM inference. 
This example uses a local LM (qwq-32b-q8_0.gguf) running in 'llama_cpp'.

Defining the agents are fully customizable.
This example defines three LM agents, plus three tools.

This multi-agent-system defines a memory with fast random access (via hashmap), which is also saved to disk as a json file everytime the mem is updated.
![Screenshot from 2025-05-11 18-53-40](https://github.com/user-attachments/assets/5056ea4e-2750-4a97-883a-feb064620705)

