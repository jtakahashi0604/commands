---
description: Generate CRUD for a specific model
argument-hint: <model name>
---

# Setup

Generate CRUD for [$ARGUMENTS] by following the example.

## Write files

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

export async function findMany({ teamId }: { teamId: string }) {
  const examples = await db.example.findMany({
    where: { teamId },
    orderBy: { createdAt: "desc" },
  });

  return examples.map((example) => fromDb({ data: example }));
}

export async function findById({ id, teamId }: { id: string; teamId: string }) {
  const example = await db.example.findUniqueOrThrow({
    where: { id, teamId },
  });

  return fromDb({ data: example });
}

export async function create({
  teamId,
  data,
}: {
  teamId: string;
  data: exampleSchema.createSchema;
}) {
  const example = await db.example.create({
    data: { ...data, teamId },
  });

  return fromDb({ data: example });
}

export async function update({
  id,
  teamId,
  data,
}: {
  id: string;
  teamId: string;
  data: exampleSchema.updateSchema;
}) {
  const example = await db.example.update({
    where: { id, teamId },
    data,
  });

  return fromDb({ data: example });
}

export async function remove({ id, teamId }: { id: string; teamId: string }) {
  const example = await db.example.delete({
    where: { id, teamId },
  });

  return fromDb({ data: example });
}

export function fromDb({ data }: { data: any }) {
  return data;
}
```

### ./src/\_services/domain/example.test.ts

```typescript
import { afterEach, beforeEach, describe, expect, it } from "vitest";

import db from "@/_lib/db";

import * as exampleService from "./example";

describe("Example Service", () => {
  const teamId1 = "test-team-1";
  const teamId2 = "test-team-2";

  let createdExampleIds: string[] = [];

  beforeEach(async () => {
    // Clean up any existing test data
    await db.example.deleteMany({
      where: {
        teamId: {
          in: [teamId1, teamId2],
        },
      },
    });
    createdExampleIds = [];
  });

  afterEach(async () => {
    // Clean up test data after each test
    if (createdExampleIds.length > 0) {
      await db.example.deleteMany({
        where: {
          id: {
            in: createdExampleIds,
          },
        },
      });
    }
  });

  describe("findMany", () => {
    it("should only return examples for the specified teamId", async () => {
      const src1 = await exampleService.create({
        teamId: teamId1,
        data: { name: "Team 1 Example" },
      });
      const src2 = await exampleService.create({
        teamId: teamId2,
        data: { name: "Team 2 Example" },
      });

      createdExampleIds.push(src1.id, src2.id);

      const team1Examples = await exampleService.findMany({ teamId: teamId1 });

      expect(team1Examples).toHaveLength(1);
      expect(team1Examples[0].teamId).toBe(teamId1);
      expect(team1Examples[0].name).toBe("Team 1 Example");

      const team2Examples = await exampleService.findMany({ teamId: teamId2 });

      expect(team2Examples).toHaveLength(1);
      expect(team2Examples[0].teamId).toBe(teamId2);
      expect(team2Examples[0].name).toBe("Team 2 Example");
    });
  });

  describe("findById", () => {
    it("should not find example from different team even with same id", async () => {
      const src = await exampleService.create({
        teamId: teamId1,
        data: { name: "example" },
      });
      createdExampleIds.push(src.id);

      await expect(
        exampleService.findById({ id: src.id, teamId: teamId2 })
      ).rejects.toThrow();
    });
  });

  describe("create", () => {
    it("should create examples with different teamIds", async () => {
      const src1 = await exampleService.create({
        teamId: teamId1,
        data: { name: "Team 1 Example" },
      });
      const src2 = await exampleService.create({
        teamId: teamId2,
        data: { name: "Team 2 Example" },
      });

      createdExampleIds.push(src1.id, src2.id);

      expect(src1.teamId).toBe(teamId1);
      expect(src2.teamId).toBe(teamId2);
    });
  });

  describe("update", () => {
    it("should not update example from different team", async () => {
      const src = await exampleService.create({
        teamId: teamId1,
        data: { name: "example" },
      });
      createdExampleIds.push(src.id);

      await expect(
        exampleService.update({
          id: src.id,
          teamId: teamId2,
          data: { name: "example" },
        })
      ).rejects.toThrow();
    });
  });

  describe("remove", () => {
    it("should not delete example from different team", async () => {
      const src = await exampleService.create({
        teamId: teamId1,
        data: { name: "example" },
      });
      createdExampleIds.push(src.id);

      await expect(
        exampleService.remove({ id: src.id, teamId: teamId2 })
      ).rejects.toThrow();

      const dst = await exampleService.findById({
        id: src.id,
        teamId: teamId1,
      });
      expect(dst.id).toBe(src.id);
    });
  });

  describe("Cross-team isolation verification", () => {
    it("should ensure complete isolation between teams", async () => {
      // Create multiple examples for each team
      const team1Example1 = await exampleService.create({
        teamId: teamId1,
        data: { name: "Team 1 Example 1" },
      });
      const team1Example2 = await exampleService.create({
        teamId: teamId1,
        data: { name: "Team 1 Example 2" },
      });
      const team2Example1 = await exampleService.create({
        teamId: teamId2,
        data: { name: "Team 2 Example 1" },
      });

      createdExampleIds.push(
        team1Example1.id,
        team1Example2.id,
        team2Example1.id
      );

      // Get examples for each team
      const team1Examples = await exampleService.findMany({ teamId: teamId1 });
      const team2Examples = await exampleService.findMany({ teamId: teamId2 });

      // Verify complete isolation
      expect(team1Examples).toHaveLength(2);
      expect(team2Examples).toHaveLength(1);

      // All team 1 examples should have teamId1
      team1Examples.forEach((example) => {
        expect(example.teamId).toBe(teamId1);
      });

      // All team 2 examples should have teamId2
      team2Examples.forEach((example) => {
        expect(example.teamId).toBe(teamId2);
      });

      // No overlap between teams
      const team1Ids = team1Examples.map((e) => e.id);
      const team2Ids = team2Examples.map((e) => e.id);
      expect(team1Ids).not.toEqual(expect.arrayContaining(team2Ids));
    });
  });
});
```

### ./src/\_actions/domain/example.ts

```typescript
"use server";

import { revalidatePath } from "next/cache";

import type * as exampleSchema from "@/_schemas/domain/example";
import * as authService from "@/_services/app/auth";
import * as exampleService from "@/_services/domain/example";

export async function findMany() {
  const teamId = await authService.getTeamId();

  const result = await exampleService.findMany({ teamId });

  return result;
}

export async function findById({ id }: { id: string }) {
  const teamId = await authService.getTeamId();

  const result = await exampleService.findById({ id, teamId });

  return result;
}

export async function create({ data }: { data: exampleSchema.createSchema }) {
  const teamId = await authService.getTeamId();

  const result = await exampleService.create({ data, teamId });

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
  const teamId = await authService.getTeamId();

  const result = await exampleService.update({ id, data, teamId });

  revalidatePath("/examples");
  revalidatePath(`/examples/${id}`);

  return result;
}

export async function remove({ id }: { id: string }) {
  const teamId = await authService.getTeamId();

  const result = await exampleService.remove({ id, teamId });

  revalidatePath("/examples");

  return result;
}
```

### ./src/\_components/domain/example/form.tsx

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

### ./src/\_components/domain/example/list.tsx

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
