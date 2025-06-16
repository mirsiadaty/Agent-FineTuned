# Agent-FineTuned
Full control over all major aspects of the LM agent.

Choose to use a vendor-provided LM language model, from your vendor of choice. Or use a LM that you have downloaded on your machine. Use your choice of environment for LM inference.

This example uses a local QwQ language model (qwq-32b-q8_0.gguf). QwQ is the reasoning model of the Qwen series. This example runs the inference on 'llama_cpp'. Llama.cpp is a software library written in C/C++ designed for efficient inference of large language models.

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
Okay, let me see. The user asked for the year-to-date gains of the top 10 big tech stocks and which company has the largest gain. 

First, I need to recall the action plan steps. The first step was installing the yfinance Python package. That's necessary because yfinance is used to fetch stock data from Yahoo Finance. The second step was running the Python code that uses yfinance to get the data. The code imports the necessary modules, defines the tickers, and calculates the YTD gains. Then it sorts them and finds the maximum.

Looking at the execution results, the code output shows the YTD percentages for each stock. Microsoft (MSFT) has the highest gain at 10.40%, while Apple (AAPL) is down the most. The code also correctly identifies MSFT as the top performer. 

I should check if there are any errors in the output. The data seems consistent, with all tickers listed. The sorting is in descending order, so the first entry is indeed the largest gain. The user might want a clear list and the answer to the second part. 

I need to present the information in a user-friendly way. List each company with their YTD percentage, then state which one has the largest gain. Make sure to mention that the data is as of the current date when the code ran. Also, note any companies with negative returns if relevant. 

Wait, the user specified "top 10 big tech stocks"—the list includes the usual suspects like AAPL, MSFT, AMZN, etc. The code used those exact tickers, so that's correct. The results are sorted properly. 

No data unavailable entries, so all tickers had valid data. The largest gain is clearly MSFT. I should format the response with bullet points for clarity and bold the key answer. Also, mention that the percentages are approximate and based on closing prices. 

Double-check the numbers from the execution results to ensure accuracy. Yep, MSFT at 10.40% is the highest. Alright, time to structure the response as per the instructions, using the specified tags.
</think>

<ResponseToUser>
Based on the latest available data, here are the year-to-date (YTD) gains for the top 10 big tech stocks as of today:

- **MSFT (Microsoft)**: 10.40%  
- **META (Meta Platforms)**: 8.15%  
- **AVGO (Broadcom)**: 4.67%  
- **NVDA (NVIDIA)**: -2.29%  
- **INTC (Intel)**: -3.31%  
- **AMZN (Amazon)**: -6.91%  
- **AMD (Advanced Micro Devices)**: -8.21%  
- **TSLA (Tesla)**: -8.65%  
- **GOOGL (Alphabet)**: -9.23%  
- **AAPL (Apple)**: -17.44%  

The company with the **largest YTD gain** is **Microsoft (MSFT)** with **10.40%**.  

*Note: These values are based on closing prices from January 1 to today and may fluctuate with market changes.*
</ResponseToUser>
```
