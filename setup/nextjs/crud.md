# Setup

Setup bellow prompts.

## Install dependencies

```bash
pnpm add -D prisma
pnpm add @prisma/client

pnpm add zod
pnpm add react-hook-form
pnpm add @hookform/resolvers
pnpm add lucide-react
```

## Run scripts

```bash
pnpx prisma init
pnpx shadcn init
```

```bash
pnpx shadcn add form input button
```

## Write setting files

### ./package.json

Please add the following scripts to this file:

```json
{
  "scripts": {
    "postinstall": "prisma generate"
  }
}
```

### ./compose.yaml

```yaml
version: "3"
services:
  db:
    image: postgres
    ports:
      - 5432:5432
    environment:
      POSTGRES_PASSWORD: postgres
```

### ./.env

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/example"
```

### ./prisma/schema.prisma

```prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

// Looking for ways to speed up your queries, or scale easily with your serverless or edge functions?
// Try Prisma Accelerate: https://pris.ly/cli/accelerate-init

generator client {
  provider = "prisma-client-js"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Example {
  id        String   @id @default(uuid())
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

```bash
pnpx prisma generate
```

### ./tsconfig.json

Please add the following scripts to this file:

```json
{
  "compilerOptions": {
    "paths": {
      "@prisma/client": ["./src/generated/prisma"]
    }
  }
}
```

### .src/\_lib/db.ts

```typescript
import { PrismaClient } from "@prisma/client";

const prismaClientSingleton = () => {
  return new PrismaClient();
};

// biome-ignore lint/suspicious/noShadowRestrictedNames: This is a global variable for Prisma client
declare const globalThis: {
  prismaGlobal: ReturnType<typeof prismaClientSingleton>;
} & typeof global;

const prisma = globalThis.prismaGlobal ?? prismaClientSingleton();

export default prisma;

if (process.env.NODE_ENV !== "production") globalThis.prismaGlobal = prisma;
```

### ./src/\_schemas/domain/example.ts

```typescript
import { z } from "zod";

export const exampleZodSchema = z.object({
  id: z.string(),
  name: z.string(),
});

export const createZodSchema = z.object({
  name: z.string().min(1, "Name is required"),
});

export const updateZodSchema = z.object({
  name: z.string().min(1, "Name is required"),
});

export type exampleSchema = z.infer<typeof exampleZodSchema>;
export type createSchema = z.infer<typeof createZodSchema>;
export type updateSchema = z.infer<typeof updateZodSchema>;
```

### ./src/\_services/domain/example.ts

```typescript
import db from "@/_lib/db";

import type * as exampleSchema from "@/_schemas/domain/example";

export async function findMany() {
  const examples = await db.example.findMany({
    orderBy: { createdAt: "desc" },
  });

  return examples.map((example) => fromDb({ data: example }));
}

export async function findById({ id }: { id: string }) {
  const example = await db.example.findUniqueOrThrow({
    where: { id },
  });

  return fromDb({ data: example });
}

export async function create({ data }: { data: exampleSchema.createSchema }) {
  const example = await db.example.create({
    data,
  });

  return fromDb({ data: example });
}

export async function update({
  id,
  data,
}: {
  id: string;
  data: exampleSchema.updateSchema;
}) {
  const example = await db.example.update({
    where: { id },
    data,
  });

  return fromDb({ data: example });
}

export async function remove({ id }: { id: string }) {
  const example = await db.example.delete({
    where: { id },
  });

  return fromDb({ data: example });
}

export function fromDb({ data }: { data: exampleSchema.exampleSchema }) {
  return data;
}
```

### ./src/\_actions/domain/example.ts

```typescript
"use server";

import { revalidatePath } from "next/cache";

import type * as exampleSchema from "@/_schemas/domain/example";
import * as exampleService from "@/_services/domain/example";

export async function findMany() {
  const result = await exampleService.findMany();

  return result;
}

export async function findById({ id }: { id: string }) {
  const result = await exampleService.findById({ id });

  return result;
}

export async function create({ data }: { data: exampleSchema.createSchema }) {
  const result = await exampleService.create({ data });

  revalidatePath("/examples");

  return result;
}

export async function update({
  id,
  data,
}: {
  id: string;
  data: exampleSchema.updateSchema;
}) {
  const result = await exampleService.update({ id, data });

  revalidatePath("/examples");
  revalidatePath(`/examples/${id}`);

  return result;
}

export async function remove({ id }: { id: string }) {
  const result = await exampleService.remove({ id });

  revalidatePath("/examples");

  return result;
}
```

### .src/\_components/domain/example/form.tsx

```typescript
"use client";

import { zodResolver } from "@hookform/resolvers/zod";
import { useRouter } from "next/navigation";
import { useForm } from "react-hook-form";

import { cn } from "@/_lib/utils";

import { Button } from "@/_components/ui/button";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormMessage,
} from "@/_components/ui/form";
import { Input } from "@/_components/ui/input";

import * as exampleSchema from "@/_schemas/domain/example";
import * as exampleAction from "@/_actions/domain/example";

export default function Component({
  example,
  className,
  ...props
}: {
  example: exampleSchema.exampleSchema;
  className?: string;
} & React.HTMLAttributes<HTMLDivElement>) {
  const router = useRouter();

  const handleRemove = async () => {
    if (example?.id == null) return;

    const result = await exampleAction.remove({
      id: example.id,
    });

    console.log("Example removed:", result);

    router.push("/examples");
  };

  const handleSubmit = async ({
    data,
  }: {
    data: exampleSchema.createSchema;
  }) => {
    const result = await exampleAction.update({
      id: example.id,
      data: {
        name: data.name,
      },
    });

    console.log("Example updated:", result);

    form.reset({}, { keepValues: true });
  };

  const form = useForm<exampleSchema.createSchema>({
    resolver: zodResolver(exampleSchema.createZodSchema),
    defaultValues: {
      name: example?.name ?? "",
    },
  });

  return (
    <div className={cn("p-2", className)} {...props}>
      <Form {...form}>
        <form
          onSubmit={form.handleSubmit(async (data) => {
            await handleSubmit({
              data,
            });
          })}
          className="space-y-2"
        >
          <FormField
            control={form.control}
            name="name"
            render={({ field }) => (
              <FormItem>
                <FormControl>
                  <Input {...field} className="w-full" />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <Button type="submit" className="w-full">
            Submit
          </Button>
          <Button type="button" className="w-full" onClick={handleRemove}>
            Remove
          </Button>
        </form>
      </Form>
    </div>
  );
}
```

### .src/\_components/domain/example/list.tsx

```typescript
"use client";

import Link from "next/link";

import { cn } from "@/_lib/utils";

import { Button } from "@/_components/ui/button";

import type * as exampleSchema from "@/_schemas/domain/example.ts";
import * as exampleAction from "@/_actions/domain/example";

export default function Component({
  examples,
  className,
  ...props
}: {
  examples: exampleSchema.exampleSchema[];
  className?: string;
} & React.HTMLAttributes<HTMLDivElement>) {
  const handleCreate = async () => {
    await exampleAction.create({
      data: {
        name: "Untitled",
      },
    });
  };

  return (
    <div className={cn("p-2", className)} {...props}>
      <div className="space-y-2">
        <Button onClick={handleCreate} className="w-full">
          Create
        </Button>
        {examples.map((example) => {
          return (
            <div key={example.id}>
              <Link href={`/examples/${example.id}`}>
                <div>{example.name}</div>
              </Link>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### ./src/force-dynamic.ts

```typescript
export const dynamic = process.env.CI ? "force-dynamic" : "auto";
```

### ./src/app/examples/page.tsx

```typescript
import * as exampleAction from "@/_actions/domain/example";
import ExampleList from "@/_components/domain/example/list";

export { dynamic } from "@/force-dynamic";

export default async function Page() {
  const examples = await exampleAction.findMany();

  return <ExampleList examples={examples} />;
}
```

### ./src/app/examples/[id]/page.tsx

```typescript
import * as exampleAction from "@/_actions/domain/example";
import ExampleForm from "@/_components/domain/example/form";

export { dynamic } from "@/force-dynamic";

export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  const example = await exampleAction.findById({
    id,
  });

  return <ExampleForm example={example} />;
}
```

## Format

```bash
pnpm exec biome format --write
```
