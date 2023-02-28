### Motivation

For workload changing system, we need to adjust the resources allocation accorading to workload, to avoid resource waste and high latency.

However, the current method confront with at least two problems:

1. Use indirect metrics to calculate relevant thread or make decisions.
2. Use an oversimplified model, and a majority of methods are not used during streaming data processing

### Introduction

Use more precise metrics and model to do the policy.

### Terms

1. Bayesian optimization:
   Bayesian optimization is a sequential design strategy for global optimization of black-box functions[1] that does not assume any functional forms. It is usually employed to optimize expensive-to-evaluate functions.

2. transfer learning algorithm :
   Transfer learning (TL) is a research problem in machine learning (ML) that focuses on storing knowledge gained while solving one problem and applying it to a different but related problem

3. state-of-the-art method : 

   refers to the highest level of general development, as of a device, technique, or scientific field achieved at a particular time

4. Backpressure:
   The term means that consumers are slower than the producers.