---
title: 'Adversarial Text Defense'
date: 2020-12-21
permalink: /posts/2020/12/adverserial-text-defense/
tags:
  - adversarial text
  - adversarial defense
  - masked language model
  - nlp
---

A Defense Method Against Adversarial Examples in Text.

Authors: [Alireza Bakhtiari](https://github.com/sarb9), [Parsa Kavehzadeh](https://github.com/parsareal).

# A Defense Method Against Adversarial Examples in Text

Adversarial examples and the defense mechanisms against them have been mainly concentrated on vision area in recent years. In comparison to vision, there are considerably fewer works, especially for defense methods, in adversarial learning in text. First, we will describe what adversarial examples in text are and then we will elaborate our defense strategy for word-level attacks.

## Table of Contents
* [What is adversarial learning?](#what-is-adversarial-learning)
	* [Attacks](#attacks)
    * [Defense](#defense)
* [Adversarial examples in text](#adversarial-examples-in-text)
	* [Different types of attacks in text](#different-types-of-attacks-in-text)
	* [Few defense methods used in text](#few-defense-methods-used-in-text)
* [Our Method](#our-method)
	* [Masked language modeling](#masked-language-modeling)
	* [Masking the tokens](#masking-the-tokens)
  	* [Creating candidates](#creating-candidates)
  	* [Checking each candidate](#checking-each-candidate)

## What is adversarial learning?
In this section, we explain what is adversarial learning in general. In recent years, there was a discovery indicating that even large deep networks are vulnerable against some specific inputs, called adversarial examples. These examples are created from original input data in a way that in human eye, there is no difference between adversarial examples and original data. However, the small purturbations added by attackers could fool deep models in various tasks. For instance, the figure in below shows that a small purturbation in pixels of an panda image resulted in fooling the classifier to label panda image as a gibbon; although in human eye, the right figure is still vividly a panda. 

![alt text](https://miro.medium.com/max/4000/1*PmCgcjO3sr3CPPaCpy5Fgw.png)

### Attacks
Most proposed strategies to generate adversarial examples have been based on defining an optimization problem. Fast Gradient Sign Method (FGSM) is roughly the most famous method proposed for generating adversarial images. More details and mathematics could be found in [Adversarial Examples: Attacks and Defenses for Deep Learning](https://arxiv.org/pdf/1712.07107.pdf). 

### Defense
Relatively fewer works have been done for defense mechanisms against adversarial examples. One simple and trival approach is adversarial training, whcih entails generating adversarial examples and training the model with them to make it robust against future attacks. Our approach would be different from adversarial training and we use pre-trained models like BERT to predict adversarial attacks in text.

## Adversarial learning in text
After observing the effect of adversarial examples in vision, some works proved that small purturbations in text would also result in fooling deep networks, even transformer-based models, in various NLP tasks such as sentiment analysis, named entity recognition, and etc. There also have been some works proposing novel approaches in helping NLP models to handle textual adversarial examples.
 
### Different types of attacks in text
There are four major types of adversarial examples in text: character-level, word-level, sentence-level, and multi-level attacks. Our defense method will be willing to overcome word-level attacks. Typos and synonyms are two major word-level attacks discovered in recent years. Some examples of these two types of attacks are presented in below:

![alt text](https://d3i71xaburhd42.cloudfront.net/472cd41fa2ba2e520706f232cae12db4a7b5e60a/1-Figure1-1.png)

![alt text](https://d3i71xaburhd42.cloudfront.net/32bc789f96acb37361ac55f36940bb52b759c229/2-Figure2-1.png)

### Few defense methods used in text
Most defense methods in text are based on deep learning approachs, including training the defense model on adversarial data. [DISP](https://arxiv.org/pdf/1909.03084.pdf) was a specific defense method employing BERT and its hidden representations to predict and corroct adversarial word-level examples.

## Our method
We decided to use Masked Language Modeling (MLM) in BERT to predict word-level adversarial examples in text. Since BERT is pretrainded with MLM objective, it could give very reliable suggestions for the masked tokens in a sentence. We will illustrate different parts of our approach as bellow.

### Masked language model
One of the pretraining obejectives for most transformer-based architectures is masked language modeling. In this task, some of the tokens in input sentences are set as [MASK] tokens and the model is forced to predict the masked tokens based on other words in sentence.

![alt text](https://miro.medium.com/max/2348/0*m_kXt3uqZH9e7H4w.png)

### Masking the tokens
The first stage of our method is to set each token in a sentence as a [MASK] token and ask BERT to predict the probability of the remaining sentence. For this, we should employ a separate tonenizer instead of BERT tokenizer since BERT tokenizer may split meaningful tokens into sub meaningless parts. 

### Creating candidates
After calculating the probability of the remaining sentence by masking each token, we we set those tokens as potential adversarial typos. Those words that their absence would result in a significantly higher probability of sentence in comparison with the situations in which they are not masked are set as candidates for typos. Our assumption is that BERT would give lower probabilities to those sentences that entail adversarial typos because typos seem usually unfamiliar to BERT.

``` python
def calculate_score(str_input):
  inputs = tokenizer(str_input, return_tensors="pt", padding = True)
  input_ids = inputs["input_ids"][0].detach().numpy()
  mask_index = np.where(input_ids == 103)[0][0]
  outputs = model(**inputs)
  logits = outputs[0]

  log_prob = 0
  for i in range(1, len(input_ids) - 1):
    token_logits = logits[0][i].detach().numpy()
    sum_all = np.sum(token_logits)
    if i == mask_index:
      max_index = np.argmax(token_logits)
      log_prob += np.log(np.abs(token_logits[max_index]/sum_all))
    else:
      log_prob += np.log(np.abs(token_logits[input_ids[i]]/sum_all))
      
  print('log_prob: ' + str(log_prob))
  return log_prob
  
def gain_candidates(sentence):
  alpha = 3
  tokens = adv_tokenizer(sentence)
  masked_sentences = generate_masks(tokens)

  probs = []
  for masked_sent in masked_sentences:
    probs.append(calculate_score(masked_sent))
  tmp_indice = sorted(range(len(probs)), key=lambda i: probs[i])[-2:]
  candidate_indice = []
  for cindice in tmp_indice:
    if np.abs(probs[cindice] - np.mean(probs)) > alpha:
       candidate_indice.append(cindice)

  return candidate_indice
```

### Checking each candidate
After all, we check all 1 edit distances of each typo candidate to see whether on of them could elavate noticeably the probability of the sentence or not.