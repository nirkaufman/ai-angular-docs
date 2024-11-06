# Module II : Infrastructure

## Create Analog project

- create new project: `npm create analog@latest`
- explain project structure
- clean up project styles, and HTML

__index.page.ts__
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-home',
  standalone: true,
  template: `
    <h1>Hire Power</h2>
    <h3>Your HR AI Assistant</h3>
  `,
})
export default class HomeComponent {}
```

__styles.css__
```css
/* Tailwind directives  */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## Install daisy UI library

```bash
  npm i -D daisyui@latest
```

__tailswind.config.js__
```javascript
module.exports = {
  content: ['./index.html', './src/**/*.{html,ts,md}'],
  theme: {
    extend: {},
  },
  plugins: [
    require('daisyui'),
  ],
};
```

## create a navbar component
- create a `components folder` under `src/app`

__navbar.component.ts__
```typescript
import {Component} from "@angular/core";
import {RouterLink} from "@angular/router";

@Component({
  selector: 'navbar',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div class="navbar bg-base-100">
      <div class="navbar-start">
        <a class="btn btn-ghost text-xl">HIRE POWER</a>
      </div>
      <div class="navbar-center lg:flex">
        <ul class="menu menu-horizontal px-1">
          <li><a [routerLink]="['/chat']" >Chat</a></li>
          <li><a [routerLink]="['/search']" >Search</a></li>
          <li><a [routerLink]="['/upload']" >Upload</a></li>
          <li><a [routerLink]="['/explore']" >Explore</a></li>
          <li><a [routerLink]="['/tools']" >Tools</a></li>
        </ul>
      </div>
    </div>
  `
})
export class NavbarComponent {}
```


## Create basic pages

- under `pages` create a file for each page: `chat.page.ts`, `search.page.ts`, `upload.page.ts`, `explore.page.ts`, `tools.page.ts`
- group pages by section: 
  - (chat): `chat.page.ts`, `search.page.ts`
  - (docs): `upload.page.ts`, `explore.page.ts`
  - (utils): `tools.page.ts`

- wrap each page in a `div` with a class `container px-7`


## Connect a database
- create free `postgres` instance on `render.com`
- install prisma: `npm i prisma -D`
- init prisma with `npx prisma init`
- replace `DATABASE_URL` in `.env` with the connection string from render
- create a model in `schema.prisma` for a candidate

__schema.prisma__
```prisma
model Candidate {
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

- migrate the model to a database

```bash
npx prisma migrate dev --name init
```

- inspect the database with `npx prisma studio`
- add one record with your name inside the studio

## Create a service to interact with the database
- under `src/server` create a folder named `utils`
- under `utils` create a file `db.ts`
- import `PrismaClient` from `@prisma/client` and create a new instance of it

__db.ts__
```typescript
import { PrismaClient } from '@prisma/client';

// create prisma instance
const prisma = new PrismaClient();

export default prisma;
```

## Create an API route
- under `src/server` create a folder named `routes`
- create a file name `ping.ts` to test the API
- api routes exposed under `/api` prefix by default

__server/routes/ping.ts__
```typescript
import { defineEventHandler } from 'h3';
import prisma from '../utils/db';

export default defineEventHandler( async () => {
  return prisma.candidate.findMany();
});
```

- navigate to `http://localhost:5173/api/ping` to test the API


## Create an OpenAI account
- go to `https://platform.openai.com/signup` and create an account
- which done, navigate to `https://platform.openai.com/api-keys` and create a new API key
- save the API key in a `.env` file under `OPENAI_API_KEY`


## Protect API routes
- in `vite.config` enable CORS for the API routes within the `analog` plugin
- 
```typescript
plugins: [analog({
  nitro: {
    routeRules: {
      '/api/**' : {
        cors: true,
      }
    }
  }
})]
```

