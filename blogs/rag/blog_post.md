Building a Powerful Question Answering Bot with Amazon SageMaker, Amazon
OpenSearch, Streamlit, and LangChain: A Step-by-Step Guide
================

*Amit Arora*, *Xin Huang*, *Navneet Tuteja*

One of the most common applications of Generative AI (GenAI) and Large
Language Models (LLMs) in an enterprise environment is answering
questions based on the enterprise’s knowledge corpus. Pre-trained
foundation models (FMs) perform well at Natural Language Understanding
(NLU) tasks such summarization, text generation and question answering
on a broad variety of topics but either struggle to provide accurate
(without hallucinations) answers or completely fail at answering
questions about content that they have not seen as part of their
training data. Furthermore, FMs are trained with a point in time
snapshot of data and have no inherent ability to access fresh data at
inference time, without this ability they might provide responses that
are potentially incorrect or inadequate.

A commonly used approach to address the above mentioned problem is to
use a technique called Retrieval Augumented Generation (RAG). In the RAG
approach we convert the user question into vector embeddings using an
LLM and then do a similarity search for these embeddings in a
pre-populated vector database holding the embeddings for the enterprise
knowledge corpus. A small number of similar documents (typically three)
is added as context along with the user question to the “prompt”
provided to another LLM and then that LLM generates an answer to the
user question using information provided as context in the prompt. RAG
models were introduced by [Lewis et
al.](https://arxiv.org/abs/2005.11401) in 2020 as a model where
parametric memory is a pre-trained seq2seq model and the non-parametric
memory is a dense vector index of Wikipedia, accessed with a pre-trained
neural retriever.

In this blog post we provide a step-by-step guide with all the building
blocks for creating an enterprise ready RAG application such as a
question answering bot. We use a combination of different AWS services,
open-source foundation models ([FLAN-T5
XXL](https://huggingface.co/google/flan-t5-xxl) for text generation and
[GPT-j-6B](https://huggingface.co/EleutherAI/gpt-j-6b) for embeddings)
and packages such as
[LangChain](https://python.langchain.com/en/latest/index.html) for
interfacing with all the components and
[Streamlit](https://streamlit.io/) for building the bot frontend.

We provide a cloud formation template to stand up all the resources
required for building this solution and then demonstrate how to use
LangChain for tying everything together from interfacing with LLMs
hosted on SageMaker, to chunking of knowledge base documents and
ingesting document embeddings into OpenSearch and implementing the
question answer task,

We can use the same architecture to swap the open-source models with the
[Amazon Titan](https://aws.amazon.com/bedrock/titan/) models. After
[Amazon Bedrock](https://aws.amazon.com/bedrock/) launches, we will
publish a follow-up post showing how to implement similar GenAI
applications using Amazon Bedrock, so stay tuned.

## Solution overview

We use the [SageMaker docs](https://sagemaker.readthedocs.io) as the
knowledge corpus for this post. We convert the html pages on this site
into smaller overalapping chunks of information and then convert these
chunks into embeddings using the gpt-j-6b model and store the embeddings
into OpenSearch. We implement the RAG functionality inside an AWS Lambda
function with an Amazon API Gateway frontend. We implement a chatbot
application in Streamlit which invokes the Lambda via the API Gateway
and the Lambda does a similarity search for the user question with the
embeddings in OpenSearch. The matching documents (chunks) are added to
the prompt as context by the Lambda and then the Lambda use the
flan-t5-xxl model deployed as a SageMaker Endpoint to generate an answer
to the user question. All code for this post is available in the [GitHub
repo](https://github.com/aws-samples/llm-apps-workshop/tree/main/blogs/rag).

The following figure represents the high-level architecture of the
proposed solution.

<figure>
<img src="img/ML-14328-architecture.png" id="fig-architecture"
alt="Figure 1: Architecture" />
<figcaption aria-hidden="true">Figure 1: Architecture</figcaption>
</figure>

As illustrated in the architecture diagram, we use the following AWS
services:

- [Amazon SageMaker](https://aws.amazon.com/pm/sagemaker) and [Amazon
  SageMaker JumpStart](https://aws.amazon.com/sagemaker/jumpstart/) for
  hosting the two LLMs.
- [Amazon OpenSearch
  Service](https://aws.amazon.com/opensearch-service/) for storing the
  embeddings of the enterprise knowledge corpus and doing similarity
  search with user questions.
- [AWS Lambda](https://aws.amazon.com/lambda/) for implementing the RAG
  functionality and exposing it as a REST endpoint via the [Amazon API
  Gateway](https://aws.amazon.com/api-gateway/).
- [Amazon SageMaker Processing
  Jobs](https://docs.aws.amazon.com/sagemaker/latest/dg/processing-job.html)
  for large scale data ingestion into OpenSearch.
- [Amazon SageMaker Studio](https://aws.amazon.com/sagemaker/studio/)
  for hosting the Streamlit application.
- [AWS IAM](https://aws.amazon.com/iam/) roles and policies for access
  management.
- [AWS CloudFormation](https://aws.amazon.com/cloudformation/) for
  creating the entire solution stack through infrastructure as code.

In terms of open-source packages used in this solution, we use
[LangChain](https://python.langchain.com/en/latest/index.html) for
interfacing OpenSearch and SageMaker, and
[FastAPI](https://github.com/tiangolo/fastapi) for implementing the REST
API interface in the Lambda.

The workflow for instantiating the solution presented in this blog in
your own AWS account is as follows:

1.  Run the AWS CloudFormation template provided with this blog in your
    account. This will create all the necessary infrastructure resources
    needed for this solution.

2.  Run the
    [`data_ingestion_to_vectordb.ipynb`](./data_ingestion_to_vectordb.ipynb)
    notebook in SageMaker Notebooks. This will ingest data from
    [SageMaker docs](https://sagemaker.readthedocs.io) into an
    OpenSearch index.

3.  Run the Streamlit application on a Terminal in SageMaker Studio and
    open the URL for the application in a new browser tab.

4.  Ask your questions about SageMaker via the chat interface provided
    by the Streamlit app and view the responses generated by the LLM.

These steps are discussed in detail in the sections below.

### Prerequisites

To implement the solution provided in this post, you should have an [AWS
account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup)
and familiarity with LLMs, OpenSearch and SageMaker.

#### Use AWS Cloud Formation to create the solution stack

We use AWS CloudFormation to create a SageMaker notebook called
`aws-llm-apps-blog` and an IAM role called `LLMAppsBlogIAMRole`. Choose
**Launch Stack** for the Region you want to deploy resources to.

|        AWS Region         |                                                                                                                                                   Link                                                                                                                                                   |
|:-------------------------:|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|  us-east-1 (N. Virginia)  |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=sagemake-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-14328/sagemaker-snowflake-template.yml)    |
|     us-east-2 (Ohio)      |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=sagemake-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-14328/sagemaker-snowflake-template.yml)    |
| us-west-1 (N. California) |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-west-1#/stacks/new?stackName=sagemake-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-14328/sagemaker-snowflake-template.yml)    |
|    us-west-2 (Oregon)     |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=sagemake-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-14328/sagemaker-snowflake-template.yml)    |
|    eu-west-1 (Dublin)     |   [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=sagemake-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-14328/sagemaker-snowflake-template.yml)    |
|  ap-northeast-1 (Tokyo)   | [<img src="./img/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/new?stackName=sagemake-snowflake-example-stack&templateURL=https://aws-blogs-artifacts-public.s3.amazonaws.com/artifacts/ML-14328/sagemaker-snowflake-template.yml) |

#### Step 2

## Clean up

To avoid incurring future charges, delete the resources. You can do this
by deleting the CloudFormation template used to create the IAM role and
SageMaker notebook.

<figure>
<img src="img/cfn-delete.png" id="fig-cleaning-up-2"
alt="Figure 2: Cleaning Up" />
<figcaption aria-hidden="true">Figure 2: Cleaning Up</figcaption>
</figure>

## Conclusion

In this post, we showed ..

We encourage you to learn more by exploring the [Amazon SageMaker Python
SDK](https://sagemaker.readthedocs.io/en/stable/) and building a
solution using the sample implementation provided in this post and a
dataset relevant to your business. If you have questions or suggestions,
leave a comment.

------------------------------------------------------------------------

## Author bio

<img style="float: left; margin: 0 10px 0 0;" src="img/ML-14328-Amit.png">Amit
Arora is an AI and ML specialist architect at Amazon Web Services,
helping enterprise customers use cloud-based machine learning services
to rapidly scale their innovations. He is also an adjunct lecturer in
the MS data science and analytics program at Georgetown University in
Washington D.C.

<br><br>

<img style="float: left; margin: 0 10px 0 0;" src="img/ML-14328-xinhuang.jpg">Dr. Xin
Huang is a Senior Applied Scientist for Amazon SageMaker JumpStart and
Amazon SageMaker built-in algorithms. He focuses on developing scalable
machine learning algorithms. His research interests are in the area of
natural language processing, explainable deep learning on tabular data,
and robust analysis of non-parametric space-time clustering. He has
published many papers in ACL, ICDM, KDD conferences, and Royal
Statistical Society: Series A..

<br><br>

<img style="float: left; margin: 0 10px 0 0;" src="img/ML-14328-ntuteja.jfif">Navneet
Tuteja is a Data Specialist at Amazon Web Services. Before joining AWS,
Navneet worked as a facilitator for organizations seeking to modernize
their data architectures and implement comprehensive AI/ML solutions.
She holds an engineering degree from Thapar University, as well as a
master’s degree in statistics from Texas A&M University.