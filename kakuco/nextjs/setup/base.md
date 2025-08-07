# Setup

Setup bellow prompts.

## Install dependencies

```bash
pnpm add -D husky
pnpm add -D @biomejs/biome
pnpm add -D vitest
pnpm add -D vite-tsconfig-paths

pnpm add -D prisma
pnpm add @prisma/client

pnpm add zod

pnpm add react-hook-form
pnpm add @hookform/resolvers

pnpm add lucide-react

pnpm add @clerk/nextjs

pnpm add @next/third-parties

pnpm add vanilla-cookieconsent
```

## Run scripts

```bash
pnpm exec husky init
```

```bash
pnpx prisma init
pnpx shadcn init
```

```bash
pnpx shadcn add form input button accordion
```

```
Change dir and paths in config.

- Change `components` to `_components`
- Change `lib`        to `_lib`
```

## Write files

### ./tsconfig.json

Please add the following scripts to this file.

```json
{
  "compilerOptions": {
    "paths": {
      "@prisma/client": ["./src/generated/prisma"]
    }
  }
}
```

### ./package.json

Please add the following scripts to this file.

```json
{
  "packageManager": "pnpm@10.13.1"
}
```

```json
{
  "scripts": {
    "lint": "biome check .",
    "test": "vitest run --passWithNoTests"
  }
}
```

```json
{
  "scripts": {
    "postinstall": "prisma generate"
  }
}
```

### ./biome.json

```json
{
  "vcs": {
    "enabled": true,
    "clientKind": "git"
  },
  "assist": {
    "actions": {
      "source": {
        "organizeImports": {
          "level": "on",
          "options": {
            "groups": [
              ":NODE:",
              ":PACKAGE:",
              ":BLANK_LINE:",
              "@/_lib/**",
              ":BLANK_LINE:",
              "@/_schemas/**",
              "@/_services/**",
              "@/_actions/**",
              ":BLANK_LINE:",
              "@/_components/ui/**",
              "@/_components/domain/**"
            ]
          }
        }
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "nursery": {
        "useSortedClasses": "error"
      }
    }
  },
  "files": {
    "includes": [
      "**",
      "!.next/**",
      "!node_modules/**",
      "!src/_components/ui/**",
      "!src/generated/prisma/**"
    ]
  }
}
```

### ./.vscode/settings.json

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": "explicit"
  }
}
```

### ./.vscode/extensions.json

```json
{
  "recommendations": [
    "biomejs.biome",
    "bradlc.vscode-tailwindcss",
    "GitHub.copilot"
  ]
}
```

### ./vitest.config.ts

```typescript
import tsconfigPaths from "vite-tsconfig-paths";
import { defineConfig } from "vitest/config";

export default defineConfig({
  plugins: [tsconfigPaths()],
});
```

### ./src/example.test.ts

```typescript
import { expect, test } from "vitest";

import { sum } from "./example";

test("adds 1 + 2 to equal 3", () => {
  expect(sum({ a: 1, b: 2 })).toBe(3);
});
```

### ./src/example.ts

```typescript
export function sum({ a, b }: { a: number; b: number }): number {
  return a + b;
}
```

### ./.husky/pre-commit

```bash
pnpm lint
pnpm test
```

### ./.github/workflows/ci.yaml

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
        ports: ["5432:5432"]

    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      TEST_DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      WORK_DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: ${{ secrets.CLERK_PUBLISHABLE_KEY }}
      CLERK_PUBLISHABLE_KEY: ${{ secrets.CLERK_PUBLISHABLE_KEY }}
      CLERK_SECRET_KEY: ${{ secrets.CLERK_SECRET_KEY }}

    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          run_install: false
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install
      - run: pnpm build
      - run: pnpm lint
      - run: pnpm exec prisma db push --skip-generate --force-reset
      - run: pnpm test
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
WORK_DATABASE_URL="postgresql://postgres:postgres@localhost:5432/example-work"
TEST_DATABASE_URL="postgresql://postgres:postgres@localhost:5432/example-test"

CLERK_SECRET_KEY=""
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=""

NEXT_PUBLIC_CLERK_SIGN_IN_URL=/auth/sign-in
NEXT_PUBLIC_CLERK_SIGN_IN_FALLBACK_REDIRECT_URL=/examples
NEXT_PUBLIC_CLERK_SIGN_UP_FALLBACK_REDIRECT_URL=/examples

NEXT_PUBLIC_GA_ID=[example]
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
  teamId    String
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### ./src/middleware.ts

```typescript
import { NextResponse } from "next/server";

import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const isPublicRoute = createRouteMatcher(["/", "/auth/sign-in(.*)"]);

export default clerkMiddleware(async (auth, req) => {
  const { userId, orgId: teamId } = await auth();

  if (isPublicRoute(req) !== true) {
    await auth.protect();
  }

  if (userId && teamId == null && req.nextUrl.pathname !== "/auth/team") {
    return NextResponse.redirect(new URL("/auth/team", req.url));
  }
});

export const config = {
  matcher: [
    // Skip Next.js internals and all static files, unless found in search params
    "/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)",
    // Always run for API routes
    "/(api|trpc)(.*)",
  ],
};
```

### ./src/\_lib/db.ts

```typescript
import { PrismaClient } from "@prisma/client";

const prismaClientSingleton = () => {
  const databaseUrl =
    process.env.NODE_ENV === "test"
      ? process.env.TEST_DATABASE_URL
      : process.env.WORK_DATABASE_URL;

  return new PrismaClient({
    datasources: {
      db: {
        url: databaseUrl,
      },
    },
  });
};

// biome-ignore lint/suspicious/noShadowRestrictedNames: This is a global variable for Prisma client
declare const globalThis: {
  prismaGlobal: ReturnType<typeof prismaClientSingleton>;
} & typeof global;

const prisma = globalThis.prismaGlobal ?? prismaClientSingleton();

export default prisma;

if (process.env.NODE_ENV !== "production") globalThis.prismaGlobal = prisma;
```

### ./src/\_services/app/auth.ts

```typescript
import { auth } from "@clerk/nextjs/server";

export async function getTeamId() {
  const { userId, orgId: teamId } = await auth();

  if (userId == null || teamId == null) {
    throw new Error("Unauthorized");
  }

  return teamId;
}
```

### ./src/\_components/app/tracking.tsx

```typescript
"use client";

import { GoogleAnalytics } from "@next/third-parties/google";
import { useEffect, useState } from "react";

import "vanilla-cookieconsent/dist/cookieconsent.css";
import * as CookieConsent from "vanilla-cookieconsent";

export default function Tracking({ gaId }: { gaId: string | undefined }) {
  const [acceptedAnalytics, setAcceptedAnalytics] = useState(false);

  useEffect(() => {
    CookieConsent.run({
      categories: {
        necessary: { enabled: true, readOnly: true },
        analytics: {},
      },
      language: {
        default: "en",
        translations: {
          en: {
            consentModal: {
              title: "We use cookies",
              description: "Cookie modal description",
              acceptAllBtn: "Accept all",
              acceptNecessaryBtn: "Reject all",
              showPreferencesBtn: "Manage Individual preferences",
            },
            preferencesModal: {
              title: "Manage cookie preferences",
              acceptAllBtn: "Accept all",
              acceptNecessaryBtn: "Reject all",
              savePreferencesBtn: "Accept current selection",
              closeIconLabel: "Close modal",
              sections: [
                {
                  title: "Somebody said ... cookies?",
                  description: "I want one!",
                },
                {
                  title: "Strictly Necessary cookies",
                  description:
                    "These cookies are essential for the proper functioning of the website and cannot be disabled.",

                  //this field will generate a toggle linked to the 'necessary' category
                  linkedCategory: "necessary",
                },
                {
                  title: "Performance and Analytics",
                  description:
                    "These cookies collect information about how you use our website. All of the data is anonymized and cannot be used to identify you.",
                  linkedCategory: "analytics",
                },
                {
                  title: "More information",
                  description:
                    'For any queries in relation to my policy on cookies and your choices, please <a href="#contact-page">contact us</a>',
                },
              ],
            },
          },
        },
      },
      onConsent() {
        setAcceptedAnalytics(CookieConsent.acceptedCategory("analytics"));
      },
    });
  }, []);

  if (gaId == null) {
    return;
  }

  return acceptedAnalytics ? <GoogleAnalytics gaId={gaId} /> : null;
}
```

### ./src/\_components/app/auth/team/list.tsx

```typescript
"use client";

import { OrganizationList } from "@clerk/nextjs";

import { cn } from "@/_lib/utils";

export default function Component({
  className,
  ...props
}: {
  className?: string;
} & React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div className={cn("space-y-4", className)} {...props}>
      <OrganizationList hidePersonal={true} hideSlug={true} />
    </div>
  );
}
```

### ./src/app/auth/sign-in/[[...sign-in]]/page.tsx

```typescript
import { SignIn, SignedIn, SignedOut, UserButton } from "@clerk/nextjs";

export default function Page() {
  return (
    <div>
      <SignedOut>
        <SignIn />
      </SignedOut>
      <SignedIn>
        <UserButton />
      </SignedIn>
    </div>
  );
}
```

### ./src/app/auth/team/page.tsx

```typescript
import AuthTeamList from "@/_components/app/auth/team/list";

export default async function Page() {
  return <AuthTeamList />;
}
```

### ./src/app/layout.tsx

```typescript
import { ClerkProvider } from "@clerk/nextjs";
import type { Metadata } from "next";
import { Geist, Geist_Mono } from "next/font/google";
import "./globals.css";

import Tracking from "@/_components/app/tracking";

const geistSans = Geist({
  variable: "--font-geist-sans",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

export const metadata: Metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body
          className={`${geistSans.variable} ${geistMono.variable} antialiased`}
        >
          {children}
        </body>
        <Tracking gaId={process.env.NEXT_PUBLIC_GA_ID} />
      </html>
    </ClerkProvider>
  );
}
```

### ./src/app/page.tsx

```typescript
import { SiX } from "@icons-pack/react-simple-icons";
import { BotIcon, PenSquareIcon, SettingsIcon } from "lucide-react";
import Image from "next/image";
import Link from "next/link";

import {
  Accordion,
  AccordionContent,
  AccordionItem,
  AccordionTrigger,
} from "@/_components/ui/accordion";
import { Button } from "@/_components/ui/button";

function SectionHeader({
  headline,
  title,
  description,
}: {
  headline: string;
  title: string;
  description: string;
}) {
  return (
    <div className="space-y-2">
      <div className="text-xs">{headline}</div>
      <h3 className="font-bold text-xl">{title}</h3>
      <div>{description}</div>
    </div>
  );
}

function FeatureCard({
  title,
  description,
  icon,
}: {
  title: string;
  description: string;
  icon: React.ReactNode;
}) {
  return (
    <div className="grid gap-4 bg-gray-100 p-4">
      <div className="grid grid-flow-col justify-center">{icon}</div>
      <div>{title}</div>
      <div className="text-xs">{description}</div>
    </div>
  );
}

function IntegrationCard({ icon }: { icon: React.ReactNode }) {
  return (
    <div className="grid gap-4 p-4">
      <div className="grid grid-flow-col justify-center">{icon}</div>
    </div>
  );
}

function PlanCard({
  name,
  price,
  features,
  variant,
}: {
  name: string;
  price: string;
  features: string[];
  variant?: "default" | "secondary";
}) {
  return (
    <div className="w-fit space-y-8 rounded-lg border px-8 py-12">
      <div className="space-y-2">
        <div className="font-bold text-xl">{name}</div>
        <div className="text-lg">{price}</div>
      </div>
      <ul className="list-disc pl-4">
        {features.map((feature, index) => (
          // biome-ignore lint/suspicious/noArrayIndexKey: <explanation>
          <li key={index}>{feature}</li>
        ))}
      </ul>
      <Button className="rounded-full" variant={variant}>
        Get started today
      </Button>
    </div>
  );
}

function DeveloperCard({
  name,
  title,
  imageUrl,
  xLink,
}: {
  name: string;
  title: string;
  imageUrl: string;
  xLink: string;
}) {
  return (
    <div className="grid w-fit gap-2">
      <div className="grid grid-flow-col items-center gap-4">
        <Image
          src={imageUrl}
          alt={name}
          width={64}
          height={64}
          className="rounded-full"
        />
        <div className="space-y-2">
          <div>
            <div className="font-bold">{name}</div>
            <div className="text-xs">{title}</div>
          </div>
          <div>
            <Link href={xLink} target="_blank" rel="noopener noreferrer">
              <SiX size={12} />
            </Link>
          </div>
        </div>
      </div>
    </div>
  );
}

export default function Page() {
  return (
    <div className="m-auto max-w-3xl">
      <header className="p-8 font-mono">
        <div className="grid grid-flow-col grid-cols-[auto_1fr]">
          <div>
            <BotIcon size={32} />
          </div>
          <div className="grid grid-flow-col items-center justify-end gap-2">
            <Button className="rounded-full">Sign In</Button>
          </div>
        </div>
      </header>
      <main className="space-y-16 p-8 py-32">
        <section className="space-y-8">
          <h1 className="font-bold text-4xl">
            Example is a awesome product for life and work with AI
          </h1>
          <h2 className="font-bold">We support many integrations</h2>
        </section>
        <section>
          <Image
            src="https://placeholdit.com/1024x768/dddddd/999999"
            alt="A product image"
            width={1024}
            height={768}
            className="rounded-lg"
          />
        </section>
        <section className="space-y-8">
          <SectionHeader
            headline="Features"
            title="Features - title"
            description="Features - description"
          />
          <div className="grid grid-cols-3 grid-rows-2 gap-4">
            <FeatureCard
              title="Feature 1"
              description="Description of feature 1."
              icon={<SettingsIcon />}
            />
            <FeatureCard
              title="Feature 2"
              description="Description of feature 2."
              icon={<SettingsIcon />}
            />
            <FeatureCard
              title="Feature 3"
              description="Description of feature 3."
              icon={<SettingsIcon />}
            />
            <FeatureCard
              title="Feature 4"
              description="Description of feature 4."
              icon={<SettingsIcon />}
            />
            <FeatureCard
              title="Feature 5"
              description="Description of feature 5."
              icon={<SettingsIcon />}
            />
            <FeatureCard
              title="Feature 6"
              description="Description of feature 6."
              icon={<SettingsIcon />}
            />
          </div>
        </section>
        <section className="space-y-8">
          <SectionHeader
            headline="Feature spotlight"
            title="Feature spotlight - title"
            description="Feature spotlight - description"
          />
          <div>
            <Image
              src="https://placeholdit.com/1024x768/dddddd/999999"
              alt="A feature spotlight image"
              width={1024}
              height={768}
              className="rounded-lg"
            />
          </div>
        </section>
        <section className="space-y-8">
          <SectionHeader
            headline="Integrations"
            title="Integrations - title"
            description="Integrations - description"
          />
          <div className="grid grid-cols-4 grid-rows-3 gap-4">
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
            <IntegrationCard icon={<PenSquareIcon size={16} />} />
          </div>
        </section>
        <section className="space-y-8">
          <SectionHeader
            headline="Pricing"
            title="Pricing - title"
            description="Pricing - description"
          />
          <div className="grid grid-flow-col justify-center gap-4">
            <PlanCard
              name="Free Plan"
              price="Free"
              features={["Feature 1", "Feature 2", "Feature 3"]}
              variant="secondary"
            />
            <PlanCard
              name="Free Plan"
              price="Free"
              features={["Feature 1", "Feature 2", "Feature 3"]}
              variant="default"
            />
          </div>
        </section>
        <section className="space-y-8">
          <SectionHeader
            headline="FAQ"
            title="FAQ - title"
            description="FAQ - description"
          />
          <div>
            <Accordion type="single" collapsible>
              <AccordionItem value="faq-1">
                <AccordionTrigger>Question 1</AccordionTrigger>
                <AccordionContent>Answer to question 1.</AccordionContent>
              </AccordionItem>
              <AccordionItem value="faq-2">
                <AccordionTrigger>Question 2</AccordionTrigger>
                <AccordionContent>Answer to question 2.</AccordionContent>
              </AccordionItem>
              <AccordionItem value="faq-3">
                <AccordionTrigger>Question 3</AccordionTrigger>
                <AccordionContent>Answer to question 3.</AccordionContent>
              </AccordionItem>
            </Accordion>
          </div>
        </section>
        <section className="space-y-8">
          <SectionHeader
            headline="About us"
            title="About us - title"
            description="About us - description"
          />
          <div className="grid justify-center gap-4">
            <DeveloperCard
              imageUrl="https://placeholdit.com/64x64/dddddd/999999"
              name="[example]"
              title="Product Engineer"
              xLink="https://x.com/[example]"
            />
          </div>
        </section>
      </main>
      <footer className="p-8 font-mono text-xs">
        <div className="grid grid-flow-col grid-cols-[auto_1fr]">
          <div className="grid grid-flow-col gap-4">
            <div>Privacy Policy</div>
            <div>Terms of Service</div>
          </div>
          <div className="grid grid-flow-col justify-end gap-4">
            <div>
              <SiX size={12} />
            </div>
          </div>
        </div>
      </footer>
    </div>
  );
}
```

---

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

## Run scripts

### Format

```bash
pnpm exec biome format --write
```

### Remove

```bash
rm -rf src/app/favicon.ico
rm -rf src/app/page.tsx
rm -rf public/*.svg
```

### Generate

```bash
pnpx prisma generate
```
