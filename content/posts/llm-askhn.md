+++
date = '2025-11-02T09:18:06+03:30'
title = 'My Thoughts on LLMs and Building AI Products'
showToC = true
[cover]
image = "/llm_zoomcamp.png"
+++

IIn the last few months, I’ve been working through a free, open-source course called [LLM Zoomcamp](https://github.com/DataTalksClub/llm-zoomcamp), which teaches how to build **Retrieval-Augmented Generation (RAG)** systems and follow best practices.

While there were additional materials about **agentic RAGs** and **function calling (agents)** — which were quite fun — the main direction of the course was toward building and understanding **simple RAGs**: how they work, how to connect them to vector databases, and how to evaluate and monitor them.

As part of the course, I built a project — a chatbot that answers questions based on Hacker News comments ([askhn](https://github.com/shayansm2/askhn)). Through that, and while watching [this video](https://www.youtube.com/watch?v=LCEmiRjPEtQ) about the three types of software, I started to understand not just how to build RAGs, but also _why_ large language models (LLMs) became such an important piece of software infrastructure.

### **From “Talking Like a Human” to “Thinking Like a Tool”**

When ChatGPT first came out, the most astonishing thing wasn’t its accuracy — it was how _human_ it sounded. We didn’t really care what it was saying; we were amazed that a computer could hold a natural conversation.

But things changed quickly. Now, we go to ChatGPT or similar models to ask things we _don’t know_, and when it gets something wrong, we call it “hallucinating.” In other words, our expectations evolved — we now treat LLMs not as chat partners, but as **search engines that can also talk back.**

And it’s gone even further. AI-powered IDEs like Cursor read our code, parse logs, and fix errors. AI agents use LLMs to reason about what actions to take, which tools to call, and in what order. The LLM has effectively become the **“decision-making core”** — the brain of these systems.

### **Language, Knowledge, and Reasoning — The Three Layers of LLMs**

As I worked on RAGs and agents, I realized that LLMs can be thought of as having three major capabilities:

1. **Language understanding** — parsing human language, instructions, and context.
2. **Knowledge representation** — using patterns learned from their training data to recall or summarize factual content.
3. **Reasoning** — chaining steps of logic or decisions to produce an outcome (like selecting a function to call or planning a sequence of actions).

It’s important to distinguish between these when building AI-based systems.
For example, in a **simple RAG**, your knowledge comes from _your own database_ — typically a text or vector store. The LLM doesn’t need to “know things” internally; it just needs to understand the query, find relevant information from your data, and summarize it into a coherent answer. Its built-in knowledge isn’t essential, as long as it can interpret language and synthesize retrieved content faithfully.

When you move beyond simple retrieval into **agentic RAGs** or **function calling**, reasoning becomes much more important. In these systems, the model doesn’t just answer — it _decides_ what to do next. That reasoning loop is what makes agentic systems so powerful.
The difference between **agentic RAGs** and **AI agents** mainly comes down to **who controls the orchestration**:

- In _agentic RAGs_, you (as the developer) define the orchestration logic in your own code — deciding how the model interacts with tools or retrieval pipelines.
- In _AI agent frameworks_ (like LangChain or LangGraph), the framework itself handles orchestration, planning, and memory, while your code just provides the tools.

In both cases, the **LLM is the reasoning core** — but it’s not a brain in the biological sense. It simulates reasoning patterns statistically, based on the vast amount of reasoning text it has seen during training.

### **Final Thoughts**

Working through LLM Zoomcamp and building my own project helped me learn how to **build AI products**, understand **their components**, and see how their **architecture** fits together.  
I also realized that creating an AI product that actually works reliably and with high accuracy in production is much harder than it sounds. At first, it seemed like just calling LLMs with different prompts — but in practice, there’s a lot more engineering involved: data pipelines, retrieval, evaluation, monitoring, and performance tuning all play huge roles.

I’d love to hear your thoughts on all this — and if you’re curious, check out my new personal project [**askhn**](https://github.com/shayansm2/askhn), a chatbot that answers questions based on Hacker News comments. Feedback and ideas are very welcome.
