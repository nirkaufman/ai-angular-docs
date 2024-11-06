# Module III : Core features of HirePower


## Implementing chat
- we will create a basic chatbot to assist recruiters with general knowledge  
- we will start on the backend and then move to the frontend

## Install and setup `openAI` library

- run `npm i openai` to install the openai library:


## implement backend chat service

__services/chat__
```typescript
import OpenAI from "openai";
import 'dotenv/config';

// instance of openai default to .env OPEN_AI_API_KEY
const openAi = new OpenAI();

interface ChatMessage {
  role: string;
  content: string | null;
}

async function newMessage(history: any[], message: any): Promise<ChatMessage> {
  const response = await openAi.chat.completions.create({
    model: "gpt-3.5-turbo",
    messages: [
      {
        role: "system",
        content: "You are an HR assistant. You are helping a tech human resource professional to hire candidates.",
      },
      ...history, // we [ass the history from context
      message
    ],
  });

  return response.choices[0].message ;
}

let chatHistory: ChatMessage[] = []

export async function getChatResponse(userMessage: string): Promise<ChatMessage[]> {
  const formattedMessage = { role: "user", content: userMessage }
  const chatResponse = await newMessage(chatHistory, formattedMessage)

  if(userMessage === "reset") {
    chatHistory = [];
    return chatHistory;
  }

  chatHistory.push(formattedMessage, chatResponse)

  return chatHistory;
}
```

## Implement backend chat route
- under routes create a new file named: `chat.post.ts`
- by H3 convention, the file name determine the HTTP method

__routes/chat.post.ts__
```typescript
import { defineEventHandler, readBody } from 'h3';
import { getChatResponse } from'../services/chat'

export default defineEventHandler(async (event) => {
  const body = await readBody(event);

  return getChatResponse(body.message);
});
```

- test the API using your favorite tool
- in in Webstorm you can use the built-in HTTP client (plugin in VSCode)

__api.http__
```http request
POST http://localhost:5173/api/chat
Content-Type: application/json

{
  "message": "Hello, who are you?"
}

```

## Implement frontend chat 
- under `app/services` crate a new file named `chat.service.ts`

__chat.service.ts__
```typescript
import {inject, Injectable, signal} from '@angular/core';
import {HttpClient} from "@angular/common/http";
import {toObservable} from "@angular/core/rxjs-interop";
import { switchMap} from "rxjs";

export interface ChatMessage {
  role: string;
  content: string | null;
}

@Injectable({providedIn: 'root'})
export class ChatService {
  private readonly httpClient = inject(HttpClient);
  private readonly prompt = signal<string>('reset');

  private getResponse(userPrompt: string) {
    return this.httpClient.post<ChatMessage[]>(`http://localhost:3000/chat`, {message: userPrompt}, {
      responseType: 'json'
    });
  }

  chatResponse$ = toObservable(this.prompt).pipe(
    switchMap((message) => this.getResponse(message)),
  );

  updatePrompt(input: string) {
    this.prompt.set(input);
  }
}
```

- implement chat page

__app/pages/chat/chat.page.ts__
```typescript
import {Component, ElementRef, inject, ViewChild} from "@angular/core";
import {ChatService} from "../../services/chat.service";
import {toSignal} from "@angular/core/rxjs-interop";
import {NgClass} from "@angular/common";

@Component({
  selector: 'chat-page',
  standalone: true,
  imports: [NgClass],
  template: `
    <div class="flex flex-col justify-between h-screen">
      <div class="overflow-auto p-4" #chatContainer>
        <ul class="space-y-4">
          @for (msg of chatResponse(); track msg) {
            <li class="chat {{ msg.role === 'user' ? 'chat-end' : 'chat-start' }}">
              <div class="chat-bubble {{ msg.role === 'user' ? 'chat-bubble-primary' : 'chat-bubble-secondary' }}">
                {{ msg.role }}: {{ msg.content }}
              </div>
            </li>
          }
        </ul>
      </div>


      <div class="p-4 sticky bottom-0 w-full">
        <input class="input input-bordered input-primary w-full "
               placeholder="Ask me anything..."
               (keydown.enter)="handleUserPrompt($event)">
      </div>
    </div>
  `
})
export default class ChatPage {
  private readonly chatService = inject(ChatService);

  @ViewChild('chatContainer') private chatContainer: ElementRef;

  chatResponse = toSignal(this.chatService.chatResponse$, { initialValue: [] });

  handleUserPrompt(event: any) {
    const inputElement = event.target as HTMLInputElement;
    this.chatService.updatePrompt(inputElement.value);
    inputElement.value = '';
  }
}
```

## Similarity search
- finding and retrieving text segments or documents that are similar to a given query


### Start with mock data and in-memory store
- install `langchain` library by running `npm i langchain`
- explain langchain `Document`: 

```text
A Document object in LangChain contains information about some data. It has two attributes:

pageContent: string: The content of this document. Currently is only a string.
metadata: Record<string, any>: Arbitrary metadata associated with this document. Can track the document id, file name, etc. 

```

### Implement backend similarity search\
- under `server/services` create a new file named `search.ts`

__services/search.ts__
```typescript
import 'dotenv/config'
import { Document } from 'langchain/document'
import { MemoryVectorStore } from 'langchain/vectorstores/memory'
import { OpenAIEmbeddings } from "@langchain/openai";

const candidates = [
  { id: 6, name: 'Alice Williams', bio: 'Alice Williams is a data scientist with a strong background in Python and R. She has worked on several data-driven projects and is known for her analytical skills.', skills: 'Python, R, Data Analysis, Machine Learning' },
  { id: 7, name: 'Bob Martin', bio: 'Bob Martin is a full-stack developer with a focus on JavaScript and Node.js. He has contributed to several open-source projects and is known for his clean code.', skills: 'JavaScript, Node.js, Express, MongoDB' },
  { id: 8, name: 'Charlie Davis', bio: 'Charlie Davis is a mobile app developer specializing in Swift and iOS development. He has a knack for creating intuitive user interfaces.', skills: 'Swift, iOS, Xcode, UI/UX' },
  { id: 9, name: 'Diana Johnson', bio: 'Diana Johnson is a software engineer with a focus on C++ and embedded systems. She has a proven track record of working on high-performance systems.', skills: 'C++, Embedded Systems, Real-Time Systems, Multithreading' },
  { id: 10, name: 'Ethan Brown', bio: 'Ethan Brown is a cloud specialist proficient in AWS and Google Cloud. He has helped several businesses migrate their systems to the cloud.', skills: 'AWS, Google Cloud, Docker, Kubernetes' }
];

// Convert the data into to langchain documents
// and populate store
const createStore = () =>
  MemoryVectorStore.fromDocuments(
    candidates.map(
      (c) =>
        new Document({
          // page content will convert to vectors
          pageContent: `Bio: ${c.bio}\n${c.skills}`,
          metadata: { source: c.id, name: c.name },
        })
    ),
    new OpenAIEmbeddings()
  )

 // more methods for searching are available
 // example: store.similaritySearchWithScore(query, count);
export const search = async (query: string, count: number = 2) => {
  const store = await createStore();
  return store.similaritySearch(query, count);
}
```

- under `server/routes` create a new file named `search.post.ts`

__routes/search.post.ts__
```typescript
import { defineEventHandler, readBody } from 'h3';
import { search } from'../services/search'

export default defineEventHandler(async (event) => {
  const body = await readBody(event);

  // H3 will return 204 - no content
  if (!body.query) return null;

  return search(body.query);
});
```

- with http client test the API

__api.http__
```http request
POST http://localhost:5173/api/search
Content-Type: application/json

{
  "query": "Find me a frontend developer"
}
```

### Load real data from data base
- add a `bio` column to the `candidates` table

___schema.prisma__
```prisma
model Candidate {
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  bio       String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

- run migration `npx prisma migrate dev --name add-bio-to-candidate`
- open prisma studio to validate the changes `npx prisma studio`
- add few lines of data (copy from the mock data)
- update the `search` service to use the database  

__services/search.ts__
```typescript
import 'dotenv/config'
import { Document } from 'langchain/document'
import { MemoryVectorStore } from 'langchain/vectorstores/memory'
import { OpenAIEmbeddings } from "@langchain/openai";
import prisma from "../utils/db";
import {Candidate} from "@prisma/client";

const createStore = async () => {
  const candidates = await prisma.candidate.findMany();

  return MemoryVectorStore.fromDocuments(
    candidates.map(
      (c: Candidate) =>
        new Document({
          pageContent: `Bio: ${c.bio}`,
          metadata: { email: c.email, name: c.name },
        })
    ),
    new OpenAIEmbeddings()
  )
}

export const search = async (query: string, count: number = 2) => {
  const store = await createStore();
  return store.similaritySearch(query, count);
}
```

- test the API with the http client

### Implement search frontend
- create a new service named `search.service.ts` under `app/services`

__search.service.ts__
```typescript
import {inject, Injectable, signal} from "@angular/core";
import {HttpClient} from "@angular/common/http";
import {toObservable} from "@angular/core/rxjs-interop";
import {switchMap} from "rxjs";

export interface SearchResult {
  pageContent: string
  metadata: {
    email: number,
    name: string
  }
}

@Injectable({providedIn: 'root'})
export class SearchService {
  private readonly httpClient = inject(HttpClient);
  private readonly prompt = signal<string>('');

  private getResponse(userQuery: string) {
    return this.httpClient.post<SearchResult[]>(`http://localhost:5173/api/search`, {query: userQuery}, {
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

- implement the search page

__app/pages/search/search.page.ts__
```typescript
import {Component, inject} from "@angular/core";
import {SearchService} from "../../services/search.service";
import {toSignal} from "@angular/core/rxjs-interop";

@Component({
  selector: 'search-page',
  standalone: true,
  template: `
    <div class="flex flex-col justify-between h-screen">
      <div class="overflow-auto p-4">
      @for (result of searchResponse(); track result) {
        <div class="p-2 bg-white p-6 border rounded shadow-lg mb-4">
          <h3 class="text-xl font-bold mb-2">{{ result.metadata.name }}</h3>
          <p class="text-gray-700">{{ result.pageContent }}</p>
        </div>
      }
      </div>
      <div class="p-4 bg-gray-200 sticky bottom-0">
        <input class="w-full p-2 rounded-md"
                placeholder="Search for candidate..."
               (keydown.enter)="handleUserQuery($event)">
      </div>

    </div>
  `,
})
export default class SearchComponent {
  private readonly searchService = inject(SearchService);

  searchResponse = toSignal(this.searchService.searchResponse$, { initialValue: [] })

  handleUserQuery(event: any) {
    const inputElement = event.target as HTMLInputElement;
    this.searchService.updateQuery(inputElement.value);
    inputElement.value = '';
  }
}
```

## Document upload page
- create a folder named `uploads` under `src/server`
- implement simple file upload route under `server/routes` named `upload.post.ts`

__routes/upload.post.ts__
```typescript
import { defineEventHandler, readMultipartFormData } from 'h3';
import { promises as fs } from 'fs';
import * as path from 'path';

export default defineEventHandler(async (event: any) => {
  const body = await readMultipartFormData(event);

  if (body && body.length > 0) {
    const file = body[0];
    const filePath = path.join(process.cwd(), 'src', 'server', 'uploads', file.filename!);

    await fs.writeFile(filePath, file.data);
  }

  return { message: 'File uploaded successfully' };
});
```

- implement the `upload` frontend page
- under `pages` create a new file named `upload.page.ts`

```typescript
import {Component, ElementRef, inject, signal, ViewChild} from '@angular/core';
import { CommonModule } from '@angular/common';
import {FileUploadService} from "../../services/upload.service";

@Component({
  selector: 'upload-page',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="flex flex-col items-center justify-center min-h-screen bg-gray-100">
      <div class="w-full max-w-md p-6 m-4 bg-white rounded shadow-md">
        <label class="block mb-2 text-sm font-bold text-gray-700">
          <input #fileInput
                 type="file"
                 (change)="selectFile($event)"
                 class="w-full px-3 py-2 leading-tight text-gray-700 border rounded shadow appearance-none focus:outline-none focus:shadow-outline" />
        </label>

        <div *ngIf="uploadMessage()" class="p-2 mb-4 text-sm text-center text-green-500 bg-green-100 border border-green-400 rounded">
          {{ uploadMessage() }}
        </div>

        <div class="flex items-center justify-between">
          <button class="px-4 py-2 font-bold text-white bg-green-500 rounded hover:bg-green-700 focus:outline-none focus:shadow-outline"
                  [disabled]="!currentFile"
                  (click)="upload()">
            Upload
          </button>
        </div>
      </div>
    </div>
  `,
})
export default class FileUploadComponent  {
  private readonly fileUploadService = inject(FileUploadService);

  @ViewChild('fileInput') fileInput: ElementRef;

  uploadMessage = signal<string | null>(null);
  currentFile?: File;

  selectFile(event: any): void {
    this.currentFile = event.target.files.item(0);
  }

  upload(): void {
    if (this.currentFile) {
      this.fileUploadService.upload(this.currentFile).subscribe({
        complete: () => {
          this.currentFile = undefined;
          this.fileInput.nativeElement.value = '';

          this.uploadMessage.set('File uploaded successfully');
          setTimeout(() => this.uploadMessage.set(null), 3000);
        },
      });
    }
  }
}
```
