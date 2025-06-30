--- 
date: 2025-06-30T11:34:25Z
title: 'The Potential of AI Agents'
description: ""
slug: ''
authors: []
tags: [ 'genai' ]
categories: []
externalLink: ""
series: []
---

## Introduction

I have been a wee bit skeptical about GenAI for some time now. However, recently I was shown the 'good' side of GenAI, and I mean pretty dang good.

Need an architectural proposal for your shiny new app? Bang, 2mins and 0.02cents of tokens later, something pretty decent is spat out of GenAI.

Want to have a conversation with AI? <https://www.sesame.com/>.

Want to generate a song from a prompt? <https://suno.com/about>

Want a short clip from a prompt? <https://flowvideo.ai/>

The landscape of GenAI changes rapidly, new models are coming out hard and fast, at least weekly there is a new model release and have beaten some benchmark you have never heard of, not to mention the rapid changes in tooling, web search, 'deep thinking', grounding, MCP, Agent 2 Agent communication, agentic workflows / AI.

However, the one that I am currently enamoured is the agentic AI.

## Potential

I had a light bulb moment, actually that's an understatement, my brain went off like fireworks when I saw Google's agent development kit repository <https://github.com/google/adk-samples/tree/5969ecf647a93e0505588c6438895ccbbb2f9cae>.

The part I want to focus on, is the architecture of this one agent.

![Agentic Workflow](https://raw.githubusercontent.com/google/adk-samples/5969ecf647a93e0505588c6438895ccbbb2f9cae/python/agents/travel-concierge/travel-concierge-arch.png)

It's a nice uni-directional flow of information on how the agent could operate.

### Personalization of agents

What if, we want to personalize the agents?

For example, this agent may be good at recommending the popular and safe things to visit while traveling to Paris. Guess what an LLM would recommended? Yep, that's the statistical average, probably the Eiffel tower and Arc de Triomphe is always going to be way up there.

What if I have visited Paris before and I don't want to agent to recommend places I have visited before, but I do want to make sure I squeeze in that really good restaurant I came across in my last visit? What if I'm not a history buff and don't want to visit all these famous places.

![Homer refusing to turn around to look at the leaning tower of pisa](../simpsons.jpg)

Humans are complicated. What if we had a 'personalization' agent, which already knows all this kind of information. What if it knows you recently injured your knee, so you can recommend attractions that require less walking? How would that look? What if this agent knew I took medication and made suggestions to pick up extra medication during my pre-trip planning, or if it's over the counter, suggest that I buy that before I head to somewhere remote when I'm expected to run out medication. What if I don't want to keep telling the AI Agent this again and again and I just want my preferences injected at every prompt.

![Personalization Agent](../personalization-agent.svg)

So now you have this agent that represents 'you' to add your own flavor to decision of planning this trip.

This idea is based off the [sidecar pattern in k8s](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/).

### Customization

The best thing about this personalization agent? It's interoperable, you don't like the one someone built, pick a different one.

Is it a bit crap at recommending places to see? Is it fixated on the fact you want visit all the food places and there are no attractions? or the complete opposite and forgot you're human and need to eat? No worries, pick another agent to represent you.

We can do one better, how about hyper-parameters? What you can configure your agent based on a breakdown of time spent

- 30% attractions to see (paid)
- 30% attractions to see (free)
- 10% clothes shopping
- 10% down time
- 20% wine tours

And another parameter for your budget.

Maybe that's everything _you_ want to see, how about you configure your agent to plan something that is totally out of your comfort zone, like a wild card event in your trip? Awesome, configure you agent to do that, or if it doesn't exists build one yourself. Maybe you want to quantify it, like 0.5 Standard deviation away from your comfort zone? or something crazier like 2 standard deviations away from its predictions.

### Networking

Well, maybe I'm not traveling alone, what if I am going with friends? Children (needs to be kept PG) or parents (also needs to be PG).

Well, guess what? your agent can talk to other agents and that can interoperate between the agentic network for planning your trip. Replace the `personalization_agent` with  `my_awesome_group_of_friends_agent`, which is all just an abstraction.

## To what end?

Whilst all these agents are conceptually really cool, but you may ask, to what end?

The interesting thing to note in all the architecture above is that they all 'begin' at the `root_agent`, but what is that really? Just an orchestrator? Nah, you're thinking too small, think bigger.

What if all this travel concierge is one node in a network of nodes? What if we expand the nodes to the left of `root_agent`?

![Expending on the root_agent](../left-of-root-agent.svg)

Interesting, the `family_agent` points to the concierge agents, because it is effectively performing the same task.

What if we keep expanding this?

![Even more agents are added](../agents-everywhere.drawio.svg)

If I kept going on this diagram... what do we end up with?

## A blueprint for Artificial General Intelligence

In conclusion, what if all these agents form a node in a network, and all these nodes can individually solve a problem. What if they could talk to each other and form a kind of 'intelligence' and solve more complicated problems... like a human.

Anyways, this is what ChatGPT 4 concluded about my question about the plausibility of this assuming no compute, money or physical limitation

> This theoretical framework posits an AGI as an ecosystem of communicating agents. By defining specialized agent types (sensory, cognitive, memory, meta, control) and allowing them rich interaction via ACLs and ontologies, a machine intelligence can exhibit perception, learning, creativity, and even self-reflection. Hierarchical and recursive organization ensures scalability and modularity. Emergent phenomena – collective reasoning, adaptable goals, and self-awareness – arise naturally from such a multi-agent society

Do you know what the cool thing about this is? It's all customizable, don't like an agent? or a node of an agent? Replace it with another implementation, change the configuration or build it yourself. It is (almost) true-ly debuggable, in-so far as LLMs can be 'debuggable'.

And the really really cool thing about this? Perhaps this is the definition of what 'consciousness' or 'soul' is? And, if you invert this, can you define a human's configuration in this Agentic Network? and can you download / upload this...?

This rabbit hole is very deep indeed.

![Buzz Lightyear](../buzz-lightyear.jpg)
