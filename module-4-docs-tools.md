# Building RAG application

## Introduction to RAG
- https://js.langchain.com/docs/tutorials/rag/
- explain the RAG flow
- list of loaders: https://js.langchain.com/docs/integrations/document_loaders/

## load, split, embed and query documents
- we will start with an in-memory store
- install `@langchain/community` package with `npm`
- install pdf-parse: `npm install pdf-parse`
- under `server/services` create a new file `docs.ts`

__docs.ts__
```typescript
import 'dotenv/config'
import {MemoryVectorStore} from 'langchain/vectorstores/memory'
import {OpenAIEmbeddings} from "@langchain/openai";
import {PDFLoader} from "@langchain/community/document_loaders/fs/pdf";
import {DirectoryLoader} from "langchain/document_loaders/fs/directory";
import {RecursiveCharacterTextSplitter} from "@langchain/textsplitters";
import OpenAI from 'openai'
import * as path from "path";

// set the working directory that contain the documents
const dir = path.join(path.dirname(process.cwd()), 'playground', 'src', 'uploads');
const openAi = new OpenAI();

// DirectoryLoader will load all the documents in the
// specified directory.
// TODO: Explore more loaders
const directoryLoader = new DirectoryLoader(
  dir,
  {
    ".pdf": (path: string) => new PDFLoader(path),
  }
);

// RecursiveCharacterTextSplitter will split the text
// into smaller chunks
const textSplitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1000,
  chunkOverlap: 200,
});

const loadDocs = async () => {
  return await directoryLoader.load();
}

const splitDocs = async () => {
  const docs = await loadDocs();
  return await textSplitter.splitDocuments(docs);
}

// we will use the MemoryVectorStore to store the documents
// and the OpenAIEmbeddings to vectorize the text
const loadStore = async () => {
  const docs = await splitDocs();
  return MemoryVectorStore.fromDocuments(docs, new OpenAIEmbeddings())
}

export const docQuery = async (userPrompt: string) => {
  const store = await loadStore()
  const results = await store.similaritySearch(userPrompt, 1)

  const response = await openAi.chat.completions.create({
    model: 'gpt-3.5-turbo',
    temperature: 0,
    messages: [
      {
        role: 'assistant',
        content:
          'You are a helpful AI HR assistant. Answer questions to your best ability.',
      },
      {
        role: 'user',
        content: `
        Answer the following question using the provided context. If you cannot answer the question with the context,
        don't lie and make up stuff. Just say you need more context.
        Provide as many details as possible. expect a "skills" section on provided context.
        Question: ${userPrompt}
        Context: ${results.map((r) => r.pageContent).join('\n')}
        `,
      },
    ],
  })

  return {
    answer: response.choices[0].message.content,
    sources: results.map((r) => r.metadata['source']),
  }
}
```

## Connect to a VectorStore (chrome)
- For this example we will use https://www.trychroma.com/ 
- an open source AI native vector store
- we will run it with docker - make sure yuo have docker desktop installed

## Setup
- install chromadb: `npm install chromadb`
- make sure you: `@langchain/community @langchain/openai @langchain/core`

```bash
npm i @langchain/community @langchain/openai @langchain/core chromadb
```

- make sure docker demon is running

```bash
docker pull chromadb/chroma
docker run -p 8000:8000 chromadb/chroma
```

## Create store service
- under 'server/services' create a new file `store.ts`
- inside, we will create a new Chroma store an connect it to the OpenAIEmbeddings

__store.ts__
```typescript
import { Chroma } from "@langchain/community/vectorstores/chroma";
import { OpenAIEmbeddings } from "@langchain/openai";
import { PDFLoader } from "@langchain/community/document_loaders/fs/pdf";
import OpenAI from 'openai'

const openAi = new OpenAI();

// Instantiate the OpenAIEmbeddings class with the model you want to use
const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",
});

// Instantiate the Chroma class with the embeddings
// url is optional and will default to "http://localhost:8000"
const vectorStore = new Chroma(embeddings, {
  collectionName: "candidate-cv-collection",
});

// utility function to load a single PDF file from file system
// into a LangChainDocument
export const loadFileAsDocumentAndStore = async (filePath: string) => {
  const pdfLoader = new PDFLoader(filePath);

  // create a LangChainDocument from the PDF file
  const document = await pdfLoader.load();

  // add the document to the vector store
  await vectorStore.addDocuments(document);
}

// Utility function to query the vector store
export const queryVectorStore = async (query: string) => {
  const results =  await vectorStore.similaritySearch(query);

  const response = await openAi.chat.completions.create({
    model: 'gpt-3.5-turbo',
    temperature: 0,
    messages: [
      {
        role: 'assistant',
        content:
            'You are a helpful AI HR assistant. Answer questions to your best ability.',
      },
      {
        role: 'user',
        content: `
        Answer the following question using the provided context. If you cannot answer the question with the context,
        don't lie and make up stuff. Just say you need more context.
        Provide as many details as possible. expect a "skills" section on provided context.
        Question: ${query}
        Context: ${results.map((r) => r.pageContent).join('\n')}
        `,
      },
    ],
  })

  return {
    answer: response.choices[0].message.content,
    sources: results.map((r) => r.metadata['source']),
  }
}


export default vectorStore;

```

- refactor the upload route to use the new store service
- after uploading a file, use the `loadFileAsDocumentAndStore` function to load the file into the stor

__upload.post.ts__
```typescript
import { defineEventHandler, readMultipartFormData } from 'h3';
import { promises as fs } from 'fs';
import * as path from 'path';
import {loadFileAsDocumentAndStore} from "../services/store";

export default defineEventHandler(async (event: any) => {
  const body = await readMultipartFormData(event);

  if (body && body.length > 0) {
    const file = body[0];
    const filePath = path.join(process.cwd(), 'src', 'uploads', file.filename!);

    await fs.writeFile(filePath, file.data);

    // load the file into the store
    await loadFileAsDocumentAndStore(filePath);
  }

  return { message: 'File uploaded successfully' };
});
```

## Query the store
- refactor the `docs` route to use the new store service

__docs.post.ts__
```typescript
import { defineEventHandler, readBody } from 'h3';
import {docQuery} from "../services/docs";
import {queryVectorStore} from "../services/store";

export default defineEventHandler(async (event) => {
  const body = await readBody(event);

  // H3 will return 204 - no content
  if (!body.query) return null;

  // return docQuery(body.query);
  return queryVectorStore(body.query);
});
```

- upload a PDf document and test the refactored endpoints

__api.http__
```http
###
POST http://localhost:5173/api/docs
Content-Type: application/json

{
  "query": "Tell me about nir kaufman"
}
```

## Implement docs fronted
- create a `doc.service.ts` file under `src/app/services`

__doc.service.ts__
```typescript
import {HttpClient} from "@angular/common/http";
import {inject, Injectable, signal} from "@angular/core";
import {switchMap} from "rxjs";
import {toObservable} from "@angular/core/rxjs-interop";

@Injectable({providedIn: 'root'})
export class DocsService {
  private readonly httpClient = inject(HttpClient);
  private readonly prompt = signal<string>('');

  private getResponse(userQuery: string) {
    return this.httpClient.post<{ answer: string, sources: string[] }>(`http://localhost:5173/api/docs`, {query: userQuery}, {
      responseType: 'json'
    });
  }

  searchResponse$ = toObservable(this.prompt).pipe(
    switchMap((userQuery) => this.getResponse(userQuery)),
  );

  updateQuery(userInput: string) {
    this.prompt.set(userInput);
  }

}
```

- update the `explore page` to use the new service

__explore.page.ts__
```typescript
import {Component, inject} from '@angular/core';
import {toSignal} from "@angular/core/rxjs-interop";
import {DocsService} from "../../services/docs.service";

@Component({
  selector: 'docs',
  standalone: true,
  template: `
    <div class="flex flex-col justify-between h-screen">
      <div class="overflow-auto p-4">
        <div class=" p-2 bg-white p-6 border rounded shadow-lg mb-4">
          <p class="text-gray-700">{{ docsResponse().answer  }}</p>

          @for(source of docsResponse().sources; track source) {
            <div class="p-2 bg-gray-100 my-2">
              <p class="text-gray-700">{{ source.split('/').pop() }}</p>
            </div>
          }

        </div>
      </div>
      <div class="p-4 bg-gray-200 sticky bottom-0">
        <input class="w-full p-2 rounded-md"
               placeholder="Search for candidate..."
               (keydown.enter)="handleUserQuery($event)">
      </div>
    </div>

  `,
  styles: ``
})
export default class ExplorerComponent {
  private readonly docsService = inject(DocsService);

  docsResponse = toSignal(this.docsService.searchResponse$, { initialValue: {answer: '', sources: []} });

  handleUserQuery(event: any) {
    const inputElement = event.target as HTMLInputElement;
    this.docsService.updateQuery(inputElement.value);
    inputElement.value = '';
  }

}
```

# Using custom tools
- LLM can't do everything, sometimes you need to use custom tools
- we will implement an helper tool that calculate the distance between two locations
- also. the whether in a specific location
- It will help the HR assistant to answer questions like:
  - How far is the candidate from the office?
  - What is the whether in the candidate location?
  - What is the average salary in the candidate location? 

- start by implementing utilities function to calculate the distance between two locations, amd whether in a specific location
- we will start with hard-codded values that will be replaced with real data
- create a new files named `hiring-tools.ts` under `server/utils`

__hiring-tools.ts__
```typescript
// Example dummy function hard coded to return the same weather
// In production, this could be your backend API or an external API
export function getCurrentWeather(location: string, unit = "fahrenheit") {
  // this log will help us know when this function is called
  console.log('getCurrentWeather executed');
    
  // haed codded values for specific locations
  if (location.toLowerCase().includes("tokyo")) {
    return JSON.stringify({location: "Tokyo", temperature: "10", unit: "celsius"});
  } else if (location.toLowerCase().includes("san francisco")) {
    return JSON.stringify({location: "San Francisco", temperature: "72", unit: "fahrenheit"});
  } else if (location.toLowerCase().includes("paris")) {
    return JSON.stringify({location: "Paris", temperature: "22", unit: "fahrenheit"});
  } else {
    return JSON.stringify({location, temperature: "unknown"});
  }
}

// use location services and google map API, and transport API to get the distance
export function getDistance(source: string, destination: string) {
  return JSON.stringify({source, destination, distance: "100 miles"});
}
```

- implement the `tools` service with the tools defined in the `hiring-tools.ts` file

__services/tools.ts__
```typescript
import 'dotenv/config'
import OpenAI from "openai";
import {getCurrentWeather, getDistance} from "../utils/hiring-tools";

const openAi = new OpenAI();

export async function startConversation(content: string | any[]) {
  // Step 1: send the conversation and available functions to the model
  // example: "What's the weather like in San Francisco, Tokyo, and Paris?"
  const messages: any[] = [];

  const response = await openAi.chat.completions.create({
    model: "gpt-4o",
    messages: [
      ...messages,
      {
        role: "system",
        content: "You are an HR assistant. You are helping a tech human resource professional to hire candidates.",
      },
      {
        role: "user",
        content: content,
      },
    ],
    tools: [
      {
        type: "function",
        function: {
          name: "get_current_weather",
          description: "Get the current weather for a given location",
          parameters: {
            type: "object",
            properties: {
              location: {
                type: "string",
                description: "The city and state, e.g. San Francisco, CA",
              },
              unit: {type: "string", enum: ["celsius", "fahrenheit"]},
            },
            required: ["location"],
          },
        },
      },
      {
        type: "function",
        function: {
          name: "get_distance",
          description: "Get the distance between one location to the other",
          parameters: {
            type: "object",
            properties: {
              source: {
                type: "string",
                description: "The destination location, city or state, e.g. Paris",
              },
              destination: {
                type: "string",
                description: "The destination location, city or state, e.g. Tokyo",
              },
            },
            required: ["source", "destination"],
          },
        },
      },
    ],
    // auto is default, but we'll be explicit
    tool_choice: "auto",
  });

  const responseMessage = response.choices[0].message;

  // Step 2: check if the model wanted to call a function
  const toolCalls = responseMessage.tool_calls;

  console.log('responseMessage.tool_calls', responseMessage.tool_calls)


  if (responseMessage.tool_calls) {
    // Step 3: call the function
    //The JSON response may not always be valid! handle errors
    const availableFunctions: Record<string, Function> = {
      // you cn add more functions here
      get_current_weather: getCurrentWeather,
      get_distance: getDistance,
    };

    // extend conversation with assistant's reply
    messages.push(responseMessage);

    for (const toolCall of toolCalls!) {
      const functionName = toolCall.function.name;


      const functionToCall = availableFunctions[functionName];
      const functionArgs = JSON.parse(toolCall.function.arguments);

      const functionResponse = functionToCall(
          functionArgs.location,
          functionArgs.unit
      );

      // extend conversation with function response
      messages.push({
        tool_call_id: toolCall.id,
        role: "tool",
        name: functionName,
        content: functionResponse,
      });
    }

    // get a new response from the model where it can see the function response
    const secondResponse = await openAi.chat.completions.create({
      model: "gpt-4o",
      messages: messages,
    });

    return secondResponse.choices[0].message.content;
  }

  return null;
}
```

- function to call the tools service within the `tools` route

__tools.post.ts__
```typescript
import { defineEventHandler, readBody } from 'h3';
import { startConversation } from'../services/tools'

export default defineEventHandler(async (event) => {
  const body = await readBody(event);

  // H3 will return 204 - no content
  if (!body.query) return null;

  return startConversation(body.query);
});
```

## implement the tools fronted

