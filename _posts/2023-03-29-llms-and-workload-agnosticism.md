---
layout: post
title: "Workload Agnosticism in Large Language Models: The Foundation for the Next Generation of Computing"
excerpt: "I think we’ll soon find that LLMs have strong parallels with Cloud Compute that will ensure they stay affordable and accessible resources, allowing the next generation of software projects and companies to thrive."
---

As discussed in my [previous post]({% post_url 2023-03-26-metaverse-vs-llms %}), LLMs such as OpenAI’s ChatGPT and GPT-4, Google’s Bard, and Meta’s LLaMA have risen seemingly out of nowhere, poised to disrupt the future of computing.

Cloud Compute changed the landscape drastically when it was introduced by Amazon in the mid 2000's, lowering the barriers to entry for new software-based businesses drastically by simplifying hosting and removing the need to worry about physical hardware.

The introduction of Cloud Compute allowed an entire genre of brand new businesses and projects to grow and thrive in an environment where they never could have before.

I think we'll soon find that LLMs have strong parallels with Cloud Compute that will ensure they stay affordable and accessible resources, allowing the next generation of software projects and companies to thrive.

## Workload Agnosticism

### In the Cloud

In Cloud Computing, there is something very pure about the pricing model of Virtual Machines. For a given customer, no matter who they are, they generally start out paying the same price as everyone else for their VMs. The only factor in the pricing of your compute is the amount of resources you consume, be it CPU, memory, disk, or network.

This type of pricing model, one that doesn't care about _what_ is running at all, just how much of the limited physical (or virtual) resources are consumed, is _workload agnostic_. It doesn't care what your workload is. If you're serving a website that has images of cats, you pay the same price as someone who is hosting _important business databases_ or something, assuming the resource utilization is the same.

In our `cats vs. bizDB` example, the business hosting their database probably derives a _lot_ more monetary value from their use of those compute resources than the person hosting their cat images (though maybe the cat image client is _happier_, having seen more cat images).

The price a customer pays is directly proportional to their consumption of compute resources. The value a customer receives, while potentially also proportional to the amount of compute used, doesn't necessarily correlate to the amount they spend on those resources.

In this _workload agnostic_ model, there's a pricing gap; some users are able to leverage the same resources as other users to gain more value. Capitalism hates this, so to solve the problem while pretending there isn't one to begin with, cloud providers:

- Find common business use cases for compute resources
- Build Level 2 products on top of their raw compute (like a Database as a Service)
- Adjust the pricing model to be proportional to the _value_ the target business customers receive
- Market and sell this special new service just to business customers, justifying the extra cost with the reduced work required by the business to maintain their database on raw compute

This process can continue up to even higher levels, a Level 3 product in this case being something like a _data lake_ that combines lots of different databases together and makes them queryable and charges _even more_ for the product. The pricing becoming even further detached from the cost of the raw compute, continuing to cut away at the business's excess utility from the cloud resources they're using.

While all that is happening, our cat image hosting friend is still able to buy the raw compute resources needed to host their cat images, because the cloud provider doesn't want to overspecialize in Layer 2 and Layer 3 services for one niche of customers and lose the rest of the market.

### In Language Models

So how does this all connect to LLMs and our friends like GPT-4? Well, there's a really fun property about today's LLMs that make them so powerful: they're general-purpose. Being born from the research field where a paper on a good generalized model is _better_ than a paper on a really good specialized one, most of the commercially available LLMs today are general purpose models trained to be good at _lots_ of things. Though a few models are tuned for specific tasks, the generalized models are so powerful that all you need to do is give them the right instructions and they're capable of an incredible variety of tasks.

The cost of operating a Large Language Model is based on two factors: the size of the model, and the number of input/output tokens you desire. For a model like GPT-4, the size of the model is fixed at either the 8k or 32k context flavor. The weights of the model are static and the training has already been done, so OpenAI just needs to load the model into the VRAM of some big GPU in the cloud and then feed in your tokens and get the output.

This means that the cost of running the same LLM scales only with the number of input and output tokens desired by the user. If you've been following along, you'll notice this is very similar to a VM that charges based on I/O usage or something like that. LLMs don't discriminate at all on the _task_ that they are asked to do, they just take the input context and write more based on the weights in the model.

Imagine a LLM user that really enjoys whimsical poetry, they can spend all day asking GPT-4 to produce different flavors of fun poems for them, feeding in prompts of 100-500 characters and getting back poems of 100-500 characters in return. The user in this case gets lots of enjoyment out of these poems, but there is not really any measurable or monetary value the user derives from the LLM.

Now imagine a LLM user that engages GPT-4 to help them do their job, they write software for some one-letter-domain company. They're feeding in prompts of 100-500 characters and getting back 100-500 characters of Python code that they can use to increase their productivity at work. As a result, they can work fewer hours and get more free time back. In this case, the user is clearly getting measurable monetary value out of the LLM.

A third case here could be a company that uses an LLM to write data analysis scripts for their data, in which case they have less of a need to hire as many data analysts, and can save money on the headcount. This company is definitely getting measurable monetary value from the LLM.

In all three of these cases, the cost of operating the model is the same. The same amount of input and output to and from the model is required, the same amount of compute resources are consumed, the same network bandwidth is used. The cost of the model does not in any way change based on the value derived from the model output. In some cases, businesses may wish to scale up usage of the model in such a way that they input and output lots more tokens and get more value that way, but the cost of operating the model still only scales with the size of the inputs and outputs, not directly with the value the user receives.

Large Language Models are _workload agnostic_, they can't discriminate based on the content of the input and output, and can't be priced based on the value received by the user. Long-time netizens may recognize this as similar to the principle behind [Net Neutrality](https://en.wikipedia.org/wiki/Net_neutrality).

## Conclusion

I believe LLMs, in this way, will be the next building blocks of computing. We'll of course have better tuned models for specific tasks that are priced more closely to account for the value of their use, just like how GPUs in a datacenter are priced more aggressively than CPUs and Memory since their use cases are more niche and easier to price discriminate.

I also believe that the Level 2 products and services we see in the LLM space will be of the following nature:

- Specifically tuned models for niche tasks (not quite Level 2 but more like Level 1*)
- Products that provide special, secret context to models, combining with a user's input to produce more reliable or accurate output, like for code autocomplete and/or auto documentation
- Products that integrate models into existing workflows and systems in such a way that they're only able to be used for specific purposes, like virtual meeting transcription and summarization or customer service chatbots

I hope that LLMs as a _raw compute_-like resource remain available to everyone. They have lots of potential to create, make people happy, and make their lives easier. I hope creative people will continue to have the ability to find great personal satisfaction at trivial cost from the use of these resources.

I also _know_ that the secretive nature of the best LLMs out there means that these organizations are already aware of the limitations of their business models. They're not sure if they want to become VM providers. They want to start selling the Database as a Services and the Data Lakes as soon as possible, but by opening the raw compute to users and collecting data today, they're better able to determine which Level 2 and 3 products they should be building.
