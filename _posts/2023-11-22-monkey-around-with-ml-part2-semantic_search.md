---
title: Monkey Around With ML and Library Data -- Semantic Search (Part 2)
date: 2023-11-22
categories: [AI]
tags: [ai, ml, python, chatgpt, semantic-search]     # TAG names should always be lowercase
image:
  path: /monkey-around-with-ml-data_PART2.webp
  alt: Chimping-Around
math: true
author: <ray_voelker>
pin: false
---

# Semantic Search For Newsdex

This is a continuation of Part One of the "Monkey Around With ML and Library Data" series of posts. See [Part One](/blog/posts/monkey-around-with-ml/) if you want to start from the beginning.

## What is Newsdex?

[newsdex.chpl.org](https://newsdex.chpl.org/)

> Newsdex is an index of notable local stories as well as death notices and obituaries, created and maintained by Library staff. Use it to search for the dates and pages a story may have appeared in one of our local newspapers. Entries for events covered in newspapers earlier than 1900 are being added retrospectively, so the indexing now includes partial coverage of articles as far back as the early 1800s. 

Technically, it's an SQLite database that has been converted from MARC record data.

Thanks to [Datasette](https://datasette.io/), it's possible to explore the structure of this database here: [newsdex.chpl.org/Newsdex+Record+Data](https://newsdex.chpl.org/Newsdex+Record+Data)

If you'd like to explore this dataset for your own ML project, it's possible to download all 1.8 million records from the project's GitHub page: [github.com/cincinnatilibrary/newsdex](https://github.com/cincinnatilibrary/newsdex)

## What is Semantic Search?

> In a nutshell, semantic search is “search with meaning”. 

https://www.nowpublishers.com/article/Details/INR-032

Semantic search differs significantly from traditional keyword searches, in that it allows for matching documents that are semantically similar to the query. Keyword searching on the other hand relies on literal lexical matching of words. While keyword searching is very important for matching proper names, places, entities, etc, semantic search offers the ability to link other documents on contextual meaning

For example, consider these newspaper headlines (they're **not** from Newsdex in case you were wondering):

```python
[
    "Advancements in Space Exploration in 2021",
    "2021: A Year of Space Exploration",
    "NASA's Breakthroughs in Martian Terrain Analysis",
    "New Horizons Probe Reveals Secrets of Pluto",
]
```
A user performs a keyword search query of:

```python
"Space exploration advancements in 2021"
```

Results may include the first two titles:
- "Advancements in Space Exploration in 2021" 
- "2021: A Year of Space Exploration"

The last two do not contain any of the literal search terms in the query, so they may be missed entirely even though they may still offer significant value to the user.

## How is Semantic Search Performed?

Semantic search is made possible by the use of AI models that can generate output that represent things that humans can easily recognize -- the intent, context, meaning of something such as a newspaper headline. This is done by creating what are known as "embeddings" or "vector embeddings" (see this post -- [Monkey Around With ML and Library Data Part 1](/blog/posts/monkey-around-with-ml/) -- for a brief review)  

Embeddings are essentially a mathematical representation of the input that creates semantic meaning for the AI model. These embeddings are able to be compared to other embeddings -- finding how similar, or dissimilar they are to one another. To perform a query, an embedding must first be created for it. The embedding is then compared to other embeddings which are often stored in what are known as "vector databases" that offer indexing and efficient searches for this purpose.

### Semantic Search Basic Workflow:

1. All documents in the dataset -- newspaper headlines in this case -- are transformed into embeddings using an AI model. 

    > **Note:** It's important to mention here that creating these embeddings is a resource-intense processing task. While this task *can* be completed on general purpose CPUs, for larger datasets -- such as Newsdex -- the process isn't feasible unless GPU(s) are used. GPUs are preferred in machine learning tasks primarily for their parallel processing capabilities -- CPUs can handle multiple tasks at once with their multi-core architectures, but GPUs excel with their hundreds to thousands of cores that can efficiently make the computations required in ML algorithms. For example, creating embeddings for 10k headlines using one of the more basic AI models was expected to take over 3 hours using an AMD Ryzen 7 CPU. Using an Nvidia Tesla T4 GPU, 1.8M headlines were able to be transformed into embeddings in under 30 minutes. More complex and capable models, combined with larger amounts of "tokens" or words in larger datasets only increase the amount of resources needed.
    {: .prompt-tip}

    ![This image displays a bar chart titled 'Time Required to Process 1.8 Million Headlines.' It compares the processing time of a CPU versus a GPU for handling 1.8 million records. The chart has two vertical bars, each representing one of the processing methods. The X-axis labels the processing methods as 'CPU' and 'GPU.' The Y-axis indicates the time required in minutes, ranging from 0 to over 5,400 minutes. The first bar, colored blue, represents the CPU. It extends upwards to indicate a processing time of 5,400 minutes. The second bar, colored green, represents the GPU and shows a significantly shorter processing time of just 30 minutes. This visual contrast starkly illustrates the substantial efficiency gain when using a GPU compared to a CPU for processing large datasets. The chart effectively conveys the message that a GPU is much faster than a CPU for this task, processing the same volume of data in a fraction of the time.](/time-to-process-1_8million_records.png "time to process 1.8M records"){: .w-100 .right width="1024" height="768"}
    
2. Embeddings are stored in a vector database. A vector database -- such as Qdrant (https://qdrant.tech/) -- indexes, and provides other methods for quickly searching for vectors presenting the closest semantic similarity for example.

3. When a user is writing and submitting a search query, that query must first be transformed into into embeddings using the same AI model that was used on the dataset in the first step.

4. The embedding is used with the vector database to retrieve which document provides the best match 

## Conclusion

In this post, we've briefly scratched the surface with an overview of what semantic search can offer in terms of a creating a more nuanced, context-aware approach to discovery of library resources. In an upcoming post, I'll cover some more of the technical aspects and how we can use some open source tools to work with our headlines, and then perform some semantic searches. 

The goal will be to produce a simple web interface where patrons can perform semantic searches for themselves.