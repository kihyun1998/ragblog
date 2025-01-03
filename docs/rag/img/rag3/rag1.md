---
sidebar_position: 1
---

# RAG란

RAG는 Retrieval Augmented Generation 의 약자로 문서들을 기반으로 검색해서 데이터를 발췌하고 발췌된 데이터를 통해 데이터를 생성하는 기술입니다.

크게 전처리 단계와 서비스 단계로 나눠서 살펴볼 수 있습니다.

## 전처리 단계
---

전처리 단계에서는 4가지 동작을 하게 됩니다.

`문서 로드`와 `Chunk 분할`, `임베딩`, `저장`을 하게 됩니다.  

### 문서 로드

문서 로드 단계에서는 다양한 확장자의 문서를 로드합니다. word, pdf 등 다양한 정보가 담겨있는 파일을 로드합니다.


```typescript
  /// loader 초기화
  const loader = new PDFLoader(pdfPath);

  /// pdf 로딩
  const docs = await loader.load();
```


### Chunk 분할

Chunk 분할 단계에서는 로드한 문서를 임베딩 하기 좋게 Chunk로 분할하는 작업을 거칩니다. 이를 문서라고 표현합니다.

```typescript
    /// splitter 정의
    const splitter = new RecursiveCharacterTextSplitter({
      chunkSize: 1000,
      chunkOverlap: 1000,
    });

    /// chunk 분할
    const splitDocs = await splitter.splitDocuments(docs);

```

### 임베딩

분할된 Chunk들을 LLM을 사용하여 vector값으로 임베딩하는 작업을 거칩니다. 이렇게 vector 값으로 임베딩하는 이유는 문장들의 유사도를 찾을 때 이용하기 위해서 입니다.

```typescript
    await this.faissStore.addDocuments(documents);
```

### 저장

임베딩한 값들을 vector store에 저장합니다. vector가 저장되는 DB입니다.

```typescript
    const faissPath = path.join(__dirname, '..', '..', this.DB_PATH);
    await this.faissStore.save(faissPath);
```

## 서비스 단계
---

서비스 단계에서는 다음과 같은 흐름으로 작업이 이뤄집니다.

1. 사용자가 질문을 합니다.

2. LLM으로 질문을 vector로 임베딩합니다.

3. 임베딩된 질문을 토대로 vector store에서 유사한 문서를 찾습니다. (retrieve 과정)

4. 초기 사용자의 질문과 찾은 문서들과 프롬프트로 통합하여 LLM에게 전달합니다.

5. LLM은 사용자에게 답변을 합니다.


아래 코드는 이 과정을 chain에 담아서 사용하고 있는 예시 코드입니다. Langchain에서 제공해주는 chain 입니다.

```typescript
  private async initializeChain() {
    const template = `
    You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question.
    If you cannot find the answer in the provided context, you must respond with only the digit "I am sorry.." - nothing else, no explanations, no apologies.
    If you can find the answer, use three sentences maximum and keep the answer concise.
    Question: {question}
    Context: {context}
    Answer: 
  `;

    const promptTemplate = ChatPromptTemplate.fromMessages([
      HumanMessagePromptTemplate.fromTemplate(template),
    ]);
    const retriever = await this.faissService.getRetriever();

    this.chain = RunnableSequence.from([
      {
        context: async (query: string) => {
          const docs = await retriever.invoke(query);

          const fDocs = this.formatDocs(docs);
          this.logger.debug(`docs ${fDocs}`);

          return fDocs;
        },

        question: new RunnablePassthrough(),
      },
      promptTemplate,
      this.llm,
      new StringOutputParser(),
    ]);
  }

  async processMessage(message: string) {
    try {
      const result = await this.chain.invoke(message);
      return { response: result };
    } catch (error) {
      this.logger.log('Error processing message:', error);
      return {
        response: "Sorry, I couldn't process your message at this time.",
      };
    }
  }
```