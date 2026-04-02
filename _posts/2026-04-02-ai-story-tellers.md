---
layout: post
title: "The Developers getting the most out of AI are great story tellers"
description: Crafting context and prompts for reliable, and consistent output.
img: "ai_context/mid.jpg"
tags: [engineering, code, ai]
comments: false
---
  Have you ever met someone who is an incredible story teller? You never get lost, they hold your attention and you follow along effortlessly. They leave out extraneous details, and still add emotional color to the story; potentially even foreshadowing events as they tell it. The best story tellers know how to tailor their words to match their audience. To get the most out of agentic software development, you must similarly be a great story teller to your agent. You must write codes with words.

  Early on in AI populism, the fluff term "prompt engineering" became widespread. It wasn't engineering, but the term acknowledged that there existed methods to achieve better output with higher consistency. The key difference between a good prompt and a bad prompt is dictating the correct level of context and seed guidance. 

  Good context prompting is the most important thing you can do to help your agents produce reliable and predictable code changes. Giving short, amorphous context forces agents to explore your organization's code base, or single repo, using only the basic part of organization to start. It may then try to fit general patterns that exist into the problem, and completely disregard the style and methods of your codebase. Letting the agent decide on how to implement the feature feels like a dice roll. One iteration, your agent may decide "let me see what the HTTP api is capable of" and spend 25,000 tokens on brute forcing an existing API. Another run, the model decides to reimplement the same logic that exists elsewhere. Your organization's code doesn't exist on the internet and there is no knowledge of your codebase. You, the developer must guide the agent into doing what you think is the best course of action is.

  Guiding an agent also leads to a number of benefits. It gives the agent a shortcut into what files and directories are the most important and relevant. This reduces the tokens spent on exploration and also leads to more predictable changes. By declaring what sorts of changes are needed, and the rough region where they are needed, the agent also tends to generate the changeset that resembles what you had cooking in your head. Changes that are expected are easier to understand and review. The surprise implementations that agents produce en masse is a huge contributing cause for review fatigue that many developers are facing today. Below are some strategies I have used to achieve better, predictable, and concise output.

  * Summarize what you are trying to accomplish and how you think you should do it. Give the agent the initial path. The less the agent has to research, the more predictable the output will be.
  * List ordered operations using numbered lists. The agent generally will do the items in the list in that order and prioritize things in that order.
  * Keep general ideas as bulleted points. Use this for style and keeping written code under regular checks. This acts as a constant reminder to stay within the bounds.
  * Force the agent to run/write tests, lint and iterate. Don't waste organizational CI time on agent produced output when you can run tests locally and iterate faster.
  * For extra stability, have the agent look over your prompt or plan and have it elaborate and revise. This checks your ideas against the reality of the code and can provide corrections if needed.
  * Do not over add context or extraneous details. This may cause the agent to forget or de-prioritize key points you have given or simply not finish the job.

![You've done it, the bugs are gone]({{ "/assets/img/ai_context/header.jpg" }} ){:width="800px"}

  You must be the story teller to your aide. Explain what you expect from the agent, where to look and how you would personally try to solve the problem. The agent is not a free 10xEngineer, but it is a pretty darn good autocomplete.
