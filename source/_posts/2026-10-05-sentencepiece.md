---
title: SentencePiece
description: A simple and language independent subword tokenizer
and detokenizer for Neural Text Processing
categories: [LLM, paper, tokenization]
tags: [LLM, paper, tokenization]
---

---

## **1. Dependency on Language-Specific Pre/Post-processing**

- **The Problem:** Many Neural Machine Translation (NMT) systems rely on language-dependent pre- and post-processors (e.g., specific tokenizers for English, Chinese, or Japanese) to handle text before and after model processing.
    
      
    
- **The Solution:** **SentencePiece** enables a purely end-to-end system that operates without any language-specific processing, training subword models directly from raw sentences.
    
      
    

## **2. Loss of Original Text Structure During Tokenization**

- **The Problem:** Standard tokenization often strips away essential textual information (like exact whitespace formatting), making it difficult or impossible to reconstruct the exact original raw text after processing.
    
      
    
- **The Solution:** **Lossless Tokenization**. SentencePiece treats input text strictly as a sequence of Unicode characters and explicitly escapes original whitespaces using a meta-symbol `_` (U+2581). This preserves spaces as tokenizable characters, making tokenization completely reversible.
    
      
    

## **3. Subword Segmentation Constraints on Raw Text**

- **The Problem:** Traditional subword segmentation algorithms typically require pre-tokenized text (text already split into words) before they can train subword models.
    
      
    
- **The Solution:** SentencePiece extends two subword segmentation algorithms—**Byte-Pair Encoding (BPE)** _(Sennrich et al., 2016)_ and **Unigram Language Model** _(Kudo, 2018)_—to support direct training straight from raw, un-tokenized sentences.
    
      
    

## **4. External Dependencies and Non-Portable Pipelines**

- **The Problem:** Subword segmentation often depends on external rules, vocabularies, or scripts that are detached from the neural network model itself.
    
      
    
- **The Solution:** **Self-Contained Models**. SentencePiece keeps all segmentation rules and vocabulary bundled natively within the model pipeline itself, creating an all-in-one, reproducible setup.

## Ref
https://arxiv.org/abs/1808.06226
