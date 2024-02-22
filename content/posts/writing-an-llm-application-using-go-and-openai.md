+++
title = 'Writing an LLM application using Go and OpenAI'
date = 2024-02-22T00:00:00+01:00
draft = false
+++

Large Language Models (LLM) has been a hot technology for the last two years after the success of ChatGPT. I frequently use it in addition to Github Copilot at work and in day-to-day life. Recently I read [this article](https://github.blog/2023-10-30-the-architecture-of-todays-llm-applications/) about the architecture of LLM applications and decided to implement something similar to try the new technology in practice.

We will be developing a support assistance API for the local internet provider, the same example as in Github's article I mentioned above. The fictional company has a database of successfully solved support cases and adds new records like these every day to the system. Our goal is to find the most relevant support case and, based on it, ask LLM to generate a possible solution to the user's request. Ideally, we would find more than one support cases and start a chat with the user instead of giving only one response back. But for simplicity, let's stick to one case and one response. 

Here is the flow diagram of how the system will work.
```
User request┌───────────────────────────────┐
───────────►│                               │
Answer      │                               ├──────────────────┐
◄───────────┤ support-assistance-api-server │                  │
            └──────┬────────────────┬───────┘                  │
                   │                │                          │
                   │                │                          │
                   │                │                          │
     Get embeddings│ Generate answer│  Get similar support case│
                   │                │                          │
                   │                │                          │
                   │                │                          │
                   ▼                ▼                          ▼
            ┌───────────────────────────────┐       ┌────────────────────┐
            │                               │       │                    │
            │                               │       │                    │
            │ OpenAI                        │       │ PostgreSQL+pgvector│
            └───────────────────────────────┘       └────────────────────┘
                   ▲                                           ▲
                   │                                           │
                   │                                           │
                   │                                           │
                   │                                           │
     Get embeddings│         Save support case with embeddings │
                   │                                           │
                   │                                           │
            ┌──────┴────────────────────────┐                  │
            │                               │                  │
            │                               ├──────────────────┘
            │support-assistance-load-history│
            └───────────────────────────────┘
```
(Generated by https://asciiflow.com.)

The main component here is an LLM, in our case it is OpenAI, which we will call via API.

What happens when we get a user request:
1. `support-assistance-api-server` receives a text of user's support request.
2. We ask `OpenAI` to generate embeddings of the user request's text. Embeddings are the meaning of the text represented in a high-dimensional vector space. Using [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity) of embeddings we can determine how close the texts are to each other in meaning. There are a lot of materials on the internet about it; for example take a look at [this article](https://platform.openai.com/docs/guides/embeddings). How generating embeddings looks in the code:
```
func (openaiLLM *openaiLLM) CreateEmbeddings(ctx context.Context, input string) ([]float32, error) {
	request := openai.EmbeddingRequest{
		Input: input,
		Model: openai.SmallEmbedding3,
	}

	response, err := openaiLLM.openaiClient.CreateEmbeddings(ctx, request)
	if err != nil {
		return nil, err
	}

	return response.Data[0].Embedding, err
}
```
Where `input` is the text for which we generate embeddings.

3. Using embeddings from the previous step we get similar support case from the PostgreSQL database with installed [pgvector extension](https://github.com/pgvector/pgvector). It might the case you already have an application with PostgreSQL database and just installing an extension seems really convenient. Here is the query to the database:
```
SELECT chat FROM support_history ORDER BY embedding <=> $1 LIMIT 1
```
Where `<=>` is a cosine distance operator provided by the extension.

4. And finally we ask OpenAI to generate a response to the user using predefined prompt with injected there the content of the support case we found on the previous step. How it looks in the code:
```
func (llm *openaiLLM) AnswerUserRequest(ctx context.Context, supportRequest, similarSupportCase string) (string, error) {
	request := openai.ChatCompletionRequest{
		Model: openai.GPT3Dot5Turbo,
		Messages: []openai.ChatCompletionMessage{
			{
				Role:    openai.ChatMessageRoleSystem,
				Content: "You are a helpful assistant providing support to users of a local internet provider.",
			},
			{
				Role:    openai.ChatMessageRoleUser,
				Content: supportRequest,
			},
			{
				Role: openai.ChatMessageRoleAssistant,
				Content: fmt.Sprintf(
					`Context: 
The user's current request is similar to a previously solved support case. Here is the information from the relevant historical case:

Historical Support Case:
"%s"

Provide direct instructions to the user based on the insights gained from the historical support case.`,
					similarSupportCase,
				),
			},
		},
	}

	response, err := llm.openaiClient.CreateChatCompletion(context.Background(), request)
	if err != nil {
		return "", err
	}

	return response.Choices[0].Message.Content, nil
}
```

Pay attention to the prompt where we inject similar support case:
```
Context: 
The user's current request is similar to a previously solved support case. Here is the information from the relevant historical case:

Historical Support Case:
"%s"

Provide direct instructions to the user based on the insights gained from the historical support case.
```
For me it's the most fascinating part, it's like programming in plain English. Imagine writing software in the future will be only like this?

Such systems are also called Retrieval Augmented Generation (RAG), there is an [excellent blog post](https://eli.thegreenplace.net/2023/retrieval-augmented-generation-in-go/) about implementing it in Go from Eli Bendersky.

Generated response may look like this:
```
$ curl -X POST -d '{"support_request":"My internet is slow today. Can you help me with it?"}' http://localhost:8080/v1/support
{"answer":"Certainly! I'd be happy to help you with your slow internet speed issue. Let's start by troubleshooting a few things. Please try the following steps:\n\n1. Connect your computer directly to the modem using an Ethernet cable if possible. This will help us determine if the slow speeds are related to your Wi-Fi network.\n\n2. Once connected, visit a speed testing website and note down the download and upload speeds you're getting.\n\n3. If the speeds are significantly better when connected directly to the modem, it could indicate a problem with your Wi-Fi network. In that case, try resetting your router by unplugging it from the power source, waiting for about 30 seconds, and plugging it back in. Then, retest your internet speeds.\n\n4. If the speeds improve after resetting the router, the issue was likely with your Wi-Fi network. If not, please let me know, and we can proceed with further troubleshooting steps.\n\nI hope these steps help improve your internet speed. Let me know the results, and we can proceed accordingly."}
```

In the background `support-assistance-load-history` is running and filling database with solved support cases and appropriate embeddings:
1. In our simplified architecture it loads support cases from the disk.
2. Asks `OpenAI` to generate embeddings using the same API request as in `support-assistance-api-server`.
3. Saves support case text with associated embeddings in the database, so that `support-assistance-load-history` can load it new user request is similar to what was already solved.

Example support cases were generated by ChatGPT using this prompt:
```
Hello! Please, generate a conversation between support engineer from local internet company and a customer asking for a help. The case should be solved and the end.
```

See the repository for the full source code: https://github.com/seregayoga/support-assistant-api

**Conclusion**

It was fun to play with the new technology and I can see many applications where it can shine, for example customized chatbots, searching information in unstructured documents or text classification. But for getting my bank account balance I would prefer good old software rather than asking an LLM "What do I have on my account?".