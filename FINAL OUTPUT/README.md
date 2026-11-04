# Router - Retrieval RAG Project

This project builds two-agent (Retreival-Augmented Reality) RAG system using **CreW AI**

Crew AI is a python library that provides necessary tools and important coordination work for buildig **Agents** and **RAGs** systems. The CreW AI acts like a orchestration level, which coordinates calls to LLMs, tools, and so on.
It is instrumental in connecting two or more agents. Multiple agents are useful when a task can be divided into different roles. Each agent focuses on one job, which makes the system more reliable and easier to check and fix.

## Agents
The following two agents are built in the project:
* Router Agent: When a query is sent by the user, this agent directs the **Retrieval** agent to refer to the correct source of information. In this project, the information can be found in the PDF, on the Web, or it can be answered  using the AI model's (LLM) own general knowledge. The Router itself does not search or answer the question.
* Retrieval Agent: This agent acts based on the decision taken by the Router agent. For exampl, if the Router agent directs to the PDF document, the Retrieval Agent spcifically searches for answers only in the PDF. This saves time of the Retriever agent to looking somewhere else.

## Knowledge Source
The following tools/routes are used for the following search:
* PDF Search: pdf_search. This tool comes for Crew AI (crewai-tools)..
* Web search: web_search. This tool is provided by Tavilly.
* LLM General Knowledge: direct_llm: No tool is used; the LLM answers directly.

## Setup and installation

This sections helps us to configure the following keys:
* Open API key: Use this key to connect to the LLM. This key is required to search the PDF document and to answers questions based on general knowledge that needs no retrival.
* Tavily keys: Use to search the web if the Router directs the query to web search.

The tools we are using here are the PDF search tool (CrewAI) and the web search tool (Tavily). These two tools are used only by the Retriever.

## How it works (for example) 
For example, Question 2: "What BLEU score did the Transformer (big) model achieve?"

1. The question typed by the user is received by Crew AI.
2. The question is then forwarded to the Router Agent.
3. The Router, with the help of the LLM (AI Model), decides where the answer can be found: in the attached PDF, on the web, or from the AI model's own general knowledge. In this case, the answer is in the PDF. The Router writes down its decision in a fixed format: where to look, why, and what to search for. Please note that the Router itself does not answer the question and does not search anything. The Router Agent fixes the tool that needs to be used and passes it to the Retriever Agent.
4. The decision is passed to the Retriever Agent to act on it. The Retriever is allowed to search only the PDF, so it cannot look in the wrong place.
5. The Retriever with the help of LLM uses the tools to search for the answers in PDF. The LLM decides what to search for and how, with the help of PDF Search tool provided by Crew AI, which returns the most relevant parts of the papers.
6. The searched result is refined by the LLM and passed again to Retriever. The LLM writes the answer only from the retrived text and adds the source (section from which the answer is received, 6.1).
7. The Reriever passes the final answer to Crew AI, which is then sent to the user. Every step is recorded in the trace logs.


## Challenges faced and trade-offs made

* Python 3.14 was not supported by CrewAI, so the project was run on Google Colab.
  
**Crew AI does not support Py 3.14, which was installed on my laptop. So the project was moved to Google Colab, which already uses a supported Python version (3.13). CrewAI supports Python 3.10 - 3.13.**





* Upgrading all packages caused version conflicts (pandas), so only the missing packages were installed.

%pip install -q -U crewai crewai-tools langchain-community tavily-python pydantic pandas matplotlib python-dotenv
%pip install -q crewai crewai-tools langchain-community tavily-python python-dotenv

**Here we used the second command as the -U upgrades to newer versions, even if they're already installed. This was creating conflict as an older version of pandas was already installed, which Colab needs. We removed -U and put the older pandas version back, so no conflict was created.**

* CrewAI could not run directly inside Colab (event loop error), so each agent run was started in a separate thread.
* Using two agents adds one extra LLM call per question, but makes each decision clear and easy to check.
* If the Router's decision cannot be read, a simple keyword-based backup router is used.