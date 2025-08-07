# Setup

Setup bellow prompts.

## Install dependencies

```bash
pnpm add stripe
```

## Run scripts

## Write files

### ./.env

Please add the following scripts to this file.

```env
STRIPE_SECRET_KEY=sk_test_[example]
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_[example]
STRIPE_WEBHOOK_SECRET=whsec_[example]

NEXT_PUBLIC_STRIPE_PAYMENT_URL=https://buy.stripe.com/test_[example]
NEXT_PUBLIC_STRIPE_SETTING_URL=https://billing.stripe.com/p/login/test_[example]
```

### ./src/\_services/app/subscription.ts

```typescript
import Stripe from "stripe";

import db from "@/_lib/db";

const ACTIVE_SUBSCRIPTION_STATUSES = ["trialing", "active"];

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY as string, {
  apiVersion: "2025-06-30.basil",
});

export async function create({
  clientReferenceId,
  subscriptionId,
}: {
  clientReferenceId: string;
  subscriptionId: string;
}) {
  await db.subscription.create({
    data: {
      clientReferenceId,
      subscriptionId,
    },
  });
}

export async function getPaymentUrl({
  clientReferenceId,
}: {
  clientReferenceId: string;
}) {
  return `${process.env.NEXT_PUBLIC_STRIPE_PAYMENT_URL}?client_reference_id=${clientReferenceId}`;
}

export async function getSettingUrl({
  clientReferenceId,
}: {
  clientReferenceId: string;
}) {
  const subscription = await db.subscription.findFirst({
    where: { clientReferenceId },
    orderBy: { createdAt: "desc" },
  });

  if (subscription == null) {
    return null;
  }

  return `${process.env.NEXT_PUBLIC_STRIPE_SETTING_URL}`;
}

export async function getSubscriptionStatus({
  clientReferenceId,
}: {
  clientReferenceId: string;
}) {
  const subscription = await db.subscription.findFirst({
    where: { clientReferenceId },
    orderBy: { createdAt: "desc" },
  });

  if (subscription == null) {
    return null;
  }

  const stripeSubscription = await stripe.subscriptions.retrieve(
    subscription.subscriptionId
  );

  return stripeSubscription.status;
}

export async function isActive({
  clientReferenceId,
}: {
  clientReferenceId: string;
}): Promise<boolean> {
  const status = await getSubscriptionStatus({
    clientReferenceId,
  });

  if (status === null) {
    return false;
  }

  return ACTIVE_SUBSCRIPTION_STATUSES.includes(status);
}
```

## ./src/app/webhook/stripe/route.ts

```typescript
import { NextResponse } from "next/server";
import Stripe from "stripe";

import * as subscriptionService from "@/_services/app/subscription";

const STRIPE_WEBHOOK_SECRET = process.env.STRIPE_WEBHOOK_SECRET;

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY as string, {
  apiVersion: "2025-06-30.basil",
});

export async function POST(request: Request) {
  if (STRIPE_WEBHOOK_SECRET == null) {
    return NextResponse.json(
      {
        message: "Bad request",
      },
      {
        status: 400,
      }
    );
  }

  const sign = request.headers.get("stripe-signature");

  if (sign == null) {
    return NextResponse.json(
      {
        message: "Bad request",
      },
      {
        status: 400,
      }
    );
  }

  try {
    const body = await request.arrayBuffer();
    const buff = Buffer.from(body);

    const event = stripe.webhooks.constructEvent(
      buff,
      sign,
      STRIPE_WEBHOOK_SECRET
    );

    if (event.type !== "checkout.session.completed") {
      console.error(event.data.object);
      throw new Error("Invalid event type");
    }

    const clientReferenceId = event.data.object.client_reference_id;
    const subscriptionId = event.data.object.subscription?.toString();

    if (clientReferenceId == null || subscriptionId == null) {
      console.error(event.data.object);
      throw new Error("Invalid event data");
    }

    await subscriptionService.create({
      clientReferenceId,
      subscriptionId,
    });
  } catch (error) {
    const message = `${(error as Error).message}`;

    console.error(message);

    return NextResponse.json(
      {
        message: "Bad request",
      },
      {
        status: 400,
      }
    );
  }

  return NextResponse.json(
    {
      message: "Success",
    },
    {
      status: 200,
    }
  );
}

export const dynamic = "force-dynamic";
export const dynamicParams = true;
export const revalidate = 0;
export const fetchCache = "force-no-store";
```

### ./src/\_components/app/account/setting/subscription/form.tsx

```typescript
"use client";

import { cn } from "@/_lib/utils";

const ACTIVE_SUBSCRIPTION_STATUSES = ["trialing", "active"];

export default function Component({
  subscriptionStatus,
  paymentUrl,
  settingUrl,
  className,
  ...props
}: {
  subscriptionStatus: string | null;
  paymentUrl: string;
  settingUrl: string | null;
  className?: string;
} & React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div className={cn("space-y-4", className)} {...props}>
      {subscriptionStatus !== null &&
      settingUrl !== null &&
      ACTIVE_SUBSCRIPTION_STATUSES.includes(subscriptionStatus) ? (
        <div>
          <p>Plan: plus</p>
          <a href={settingUrl} className="hover:underline">
            Manage Subscription
          </a>
        </div>
      ) : (
        <div>
          <p>Plan: free</p>
          <a href={paymentUrl} className="hover:underline">
            Manage Subscription
          </a>
        </div>
      )}
    </div>
  );
}
```

### ./src/app/account/setting/page.tsx

```typescript
import * as authService from "@/_services/app/auth";
import * as subscriptionService from "@/_services/app/subscription";

import SubscriptionForm from "@/_components/app/account/setting/subscription/form";
import AuthTeamList from "@/_components/app/account/setting/team/list";

export default async function Page() {
  const teamId = await authService.getTeamId();

  const subscriptionStatus = await subscriptionService.getSubscriptionStatus({
    clientReferenceId: teamId,
  });
  const paymentUrl = await subscriptionService.getPaymentUrl({
    clientReferenceId: teamId,
  });
  const settingUrl = await subscriptionService.getSettingUrl({
    clientReferenceId: teamId,
  });

  return (
    <div className="space-y-4">
      <AuthTeamList />
      <SubscriptionForm
        subscriptionStatus={subscriptionStatus}
        paymentUrl={paymentUrl}
        settingUrl={settingUrl}
      />
    </div>
  );
}
```

## Run scripts

### Format

```bash
pnpm exec biome format --write
```
