# Agent-FineTuned
Full control over all major aspects of the LM agent.

Choose to use a vendor-provided LM language model, from your vendor of choice. 
Or use a LM that you have downloaded on your machine. Use your choice of environment for LM inference. 
This example uses a local LM (qwq-32b-q8_0.gguf) running in 'llama_cpp'.

Defining the agents are fully customizable.
This example defines three LM agents, plus three tools.

This multi-agent-system defines a memory with fast random access (via hashmap), which is also saved to disk as a json file everytime the mem is updated.
![agent memory as json file](https://github.com/mirsiadaty/Agent-FineTuned/blob/main/Screenshot%20from%202025-05-11%2019-20-00.png)

This is an example prompt to define the "Planner" agent:
```
AgentPlanner = """<|im_start|>user
You are a planning agent. If you do not know the answer, reply I don't know, don't make things up.
You need to follow these instructions strictly:
1.Generate a plan of actions, 'Action Plan', do not take any action.
2.What 'Action Plan' is: a numbered list of steps to take in order to answer the question marked up by the tags <Question> and </Question>.
3.Mark up your answer by tags <ActionPlan> and </ActionPlan>.
4.You can use the following 'Tools' in your 'Action Plan': PythonCodeExecutor, InternetBrowser, ResponseComposer, HumanFeedback.
 4.1.What 'PythonCodeExecutor' is: it accepts a python code, executes it in python, returns the execution output back. One can install packages via pip in PythonCodeExecutor, as in: subprocess.check_call([sys.executable, "-m", "pip", "install", package])
 4.2.What 'InternetBrowser' is: it accepts any URL and any message payload, retrieves the URL on the internet, returns the content back.
 4.3.What 'ResponseComposer' is: it takes the Question, the ActionPlan, and the answers to the steps of the ActionPlan, then it composes the final response to the end-user who asked the Question.
 4.4.What 'HumanFeedback' is: when you can not figure out how to do plan for a Question, or how to do an action, or which Tool should be used for a given action, or etc, then call this tool HumanFeedback and ask your problem/question from human.
5.For steps that use PythonCodeExecutor, generate the python code to execute such step, and include in the step.
6.Add a final step for composing response to the end-user who asked the Question.

<Question>
"""+LlmQstn+"""
/Question>
|im_end|>
<|im_start|>assistant
"""
```

The following is example of a plan the Planner agent generated for the question "Which big tech stock has the largest year-to-date gain this year? How much is the gain?".
```
<ActionPlan>
1. Use PythonCodeExecutor to install the yfinance package:
   '''python
   import subprocess
   import sys
   subprocess.check_call([sys.executable, "-m", "pip", "install", "yfinance"])
   '''

2. Use PythonCodeExecutor to execute the following code to retrieve YTD stock data and calculate gains:
   '''python
   import yfinance as yf

   tickers = ['AAPL', 'MSFT', 'AMZN', 'GOOGL', 'META', 'NVDA']

   max_gain = -float('inf')
   best_stock = ''

   for ticker in tickers:
       try:
           stock = yf.Ticker(ticker)
           hist = stock.history(period='ytd')  # Year-to-date
           if len(hist) < 2:
               continue
           start_close = hist['Close'].iloc[0]
           end_close = hist['Close'].iloc[-1]
           gain = (end_close - start_close) / start_close * 100
           if gain > max_gain:
               max_gain = gain
               best_stock = ticker
       except Exception as e:
           print(f"Error processing {ticker}: {e}")

   print(f"The stock with the largest YTD gain is {best_stock} with {max_gain:.2f}%")
   '''

3. Use ResponseComposer to compile the results from step 2 into a final answer for the user.
</ActionPlan>
```

And the final answer is composed by the Composer agent like so:
```
<think>
Okay, let's see. The user is asking which big tech stock has the largest year-to-date gain this year and how much the gain is. The provided answer says Microsoft (MSFT) with 5.02%. But wait, I should check if the action plan and execution results are correct.

First, the action plan uses yfinance to get data for AAPL, MSFT, AMZN, GOOGL, META, and NVDA. The code calculates the YTD gain by taking the first and last closing prices. The execution result shows MSFT as the winner. But I remember that NVDA (NVIDIA) has been doing really well recently. Maybe the data here is outdated or there was an error in the code?

Looking at the execution results, the code output is MSFT with 5.02%. However, if the current date is late 2023, NVIDIA might have a higher gain. But according to the provided data, the code's result is what's given. Since I have to follow the instructions strictly and use the provided execution results, I can't dispute that. The user's answer must be based on the given data, even if it might be outdated. So the correct response here is to relay the information from the execution results as per the instructions. I need to make sure I don't add any extra information beyond what's provided. The answer should state MSFT with 5.02% gain, using the given tags.
</think>

<ResponseToUser>
The big tech stock with the largest year-to-date gain this year is <b>MSFT (Microsoft)</b>, with a gain of <b>5.02%</b>.</ResponseToUser>
```
