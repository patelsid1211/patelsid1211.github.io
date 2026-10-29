---
layout: project
title: LearnArXiv
kind: AI learning assistant
summary: Pick an AI or ML topic you want to understand. LearnArXiv finds the research papers, reads them, and explains how the ideas work, with citations you can check.
status: In development
stack: [Python, FastAPI, RAG, Next.js, Tailwind]
description: How LearnArXiv turns research papers into plain-language explanations with citations.
---

## The problem

Learning a new area of AI usually means searching arXiv, skimming dozens of abstracts, and reading long papers
to find the few ideas that matter. Most people give up before they get to the useful part.

LearnArXiv does the searching and reading for you, and gives you the idea without making you read the paper.

## How it works

1. **You name a topic,** like "speculative decoding" or "vision transformers."
2. **Retrieval.** It searches arXiv for relevant papers.
3. **Reranking.** An LLM and a cross-encoder rerank the results, so the most relevant papers rise to the top.
4. **Structured reading.** For each paper, it pulls out the problem, the method, the results and the main contribution.
5. **Synthesis.** It connects the papers into a picture of the field: which ideas build on each other,
   where they disagree, and what's still open.
6. **Explanation.** You get a conversational explanation of why the research was done and how it works,
   with every claim linked to the paper it came from.

## Design choices

- **Chat first, map second.** The main view feels like a knowledgeable person explaining the topic. A visual map
  of how the papers relate is one click deeper for people who want to explore.
- **Citations on every claim.** Each statement points back to a specific paper, so you can check the
  explanation instead of trusting it.
- **A warm, readable interface** rather than a cold academic look, because the goal is to make research approachable.

## Stack

A Python and FastAPI backend handles retrieval, reranking and extraction. The frontend is Next.js with Tailwind.
