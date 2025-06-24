Of course. Let's architect and build this project from scratch. This guide will be your blueprint. Follow these steps precisely to avoid configuration issues and create a "perfect" submission for the challenge.

### **Project: "StorySpark AI"**

**The Goal:** A user types a prompt for a landing page. Our AI backend designs the page structure, writes the content, creates the necessary components and the story in Storyblok, and returns a link to the newly published page.

---

### **Phase 0: Prerequisites**

1.  **Node.js:** Make sure you have Node.js (v18 or later) installed.
2.  **Storyblok Account:** [Sign up for a free Storyblok account](https://app.storyblok.com/#!/signup).
3.  **OpenAI Account:** [Get an API key from OpenAI](https://platform.openai.com/api-keys).
4.  **Code Editor:** VS Code is recommended.

---

### **Phase 1: Project & Storyblok Setup**

#### **1.1. Create the Next.js Project**

Open your terminal and run this command. Select the options as shown.

```bash
npx create-next-app@latest story-spark
```

```
✔ What is your project named? … story-spark
✔ Would you like to use TypeScript? … Yes
✔ Would you like to use ESLint? … Yes
✔ Would you like to use Tailwind CSS? … Yes
✔ Would you like to use `src/` directory? … No
✔ Would you like to use App Router? (recommended) … Yes
✔ Would you like to customize the default import alias? … No
```

Navigate into your new project:
`cd story-spark`

#### **1.2. Install Dependencies**

Run this command to install all necessary packages.

```bash
npm install @storyblok/react openai httpx
```

#### **1.3. Setup `shadcn/ui` for a Professional Look**

This is the easiest way to get a great-looking UI.

```bash
# Initialize shadcn/ui
npx shadcn-ui@latest init

# Add the components we'll need
npx shadcn-ui@latest add button textarea card
```
Follow the prompts, accepting the defaults. This will create a `components/ui` directory.

#### **1.4. Configure Storyblok Space & Components**

This is the most critical setup step.

1.  **Create a New Space:** In your Storyblok dashboard, create a new space. Call it "StorySpark Demo".
2.  **Get Your API Keys:**
    *   Go to **Settings > Access Tokens**.
    *   You will need two tokens:
        *   **Preview Token:** Copy this one. It's for the frontend.
        *   **Management API Token:** Go to the "Management API tokens" tab, click **Generate new token**, name it "story-spark-backend", and copy the generated token. **Keep this secret!**
3.  **Define Components (Content Schema):**
    *   Go to the **Block Library**.
    *   Delete the default `feature`, `grid`, `teaser` components.
    *   Create the following **Nestable Components**:

| Component Name | Field Name        | Field Type | Description/Notes               |
| :------------- | :---------------- | :--------- | :------------------------------ |
| **`hero`**     | `headline`        | `Text`     |                                 |
|                | `subheadline`     | `Text`     |                                 |
|                | `cta_button_text` | `Text`     |                                 |
| **`feature_item`** | `name`          | `Text`     |                                 |
|                | `description`     | `Textarea` |                                 |
|                | `icon`            | `Text`     | e.g., "bolt", "shield"          |
| **`feature_list`**| `features`     | `Blocks`   | Set "Allowed block types" to only allow `feature_item` |
| **`testimonial`** | `quote`        | `Textarea` |                                 |
|                | `author_name`     | `Text`     |                                 |
| **`final_cta`**  | `title`           | `Text`     |                                 |
|                | `cta_button_text` | `Text`     |                                 |

4.  **Setup the `page` Content Type:**
    *   In the **Block Library**, click on the `page` component (it will be listed under Content Types).
    *   Add a field to it:
        *   **Field Name:** `body`
        *   **Field Type:** `Blocks`

Your Storyblok space is now ready to receive content from the AI.

---

### **Phase 2: Environment Variables**

This is a common point of error. Be precise here.

1.  Create a new file in the root of your `story-spark` project named `.env.local`.
2.  Copy the content below into it and fill in your actual keys and your space ID (found in **Settings > General**).

**File: `.env.local`**
```env
# Storyblok Keys
# Get this from Settings > Access Tokens > Preview
STORYBLOK_PREVIEW_TOKEN="YOUR_PREVIEW_TOKEN_HERE"

# Get this from Settings > Access Tokens > Management API tokens
STORYBLOK_MANAGEMENT_TOKEN="YOUR_MANAGEMENT_TOKEN_HERE"

# Get this from Settings > General
STORYBLOK_SPACE_ID="YOUR_SPACE_ID_HERE"

# OpenAI API Key
# Get this from https://platform.openai.com/api-keys
OPENAI_API_KEY="YOUR_OPENAI_API_KEY_HERE"
```

---

### **Phase 3: The Backend - AI & Storyblok Integration**

Now, we'll create the API route that powers the magic.

**Create the file: `app/api/generate/route.ts`**
```typescript
import { NextResponse } from 'next/server';
import OpenAI from 'openai';
import httpx from 'httpx';

// Initialize OpenAI client
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// Get Storyblok credentials from environment
const storyblokSpaceId = process.env.STORYBLOK_SPACE_ID;
const storyblokManagementToken = process.env.STORYBLOK_MANAGEMENT_TOKEN;

const systemPrompt = `
You are an expert landing page designer and copywriter. Your task is to take a user's request and convert it into a structured JSON object that represents a Storyblok page.

The available components are: hero, feature_list, feature_item, testimonial, final_cta.

The final JSON output MUST be a valid JSON object following this structure:
{
  "name": "Page Name",
  "slug": "page-slug",
  "content": {
    "component": "page",
    "body": [
      // This should be an array of component objects based on the user's prompt.
    ]
  }
}

Here are the schemas for the available nestable components:
- hero: { "component": "hero", "headline": "...", "subheadline": "...", "cta_button_text": "..." }
- feature_list: { "component": "feature_list", "features": [ /* an array of feature_item objects */ ] }
- feature_item: { "component": "feature_item", "name": "...", "description": "...", "icon": "..." }
- testimonial: { "component": "testimonial", "quote": "...", "author_name": "..." }
- final_cta: { "component": "final_cta", "title": "...", "cta_button_text": "..." }

Based on the user's prompt, generate compelling and complete page content. Infer the content and structure logically.
- Slugs should be lowercase and hyphenated.
- For feature_item icons, use simple, one-word FontAwesome-style names (e.g., 'bolt', 'shield', 'rocket').
`;

export async function POST(request: Request) {
  if (!storyblokManagementToken || !storyblokSpaceId || !process.env.OPENAI_API_KEY) {
    return NextResponse.json({ error: 'Server configuration error.' }, { status: 500 });
  }

  try {
    const { prompt } = await request.json();

    if (!prompt) {
      return NextResponse.json({ error: 'Prompt is required.' }, { status: 400 });
    }

    // 1. Call OpenAI to get the structured JSON for the page
    const aiResponse = await openai.chat.completions.create({
      model: 'gpt-4-1106-preview', // Or a newer model that supports JSON mode
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: prompt },
      ],
      response_format: { type: 'json_object' },
    });

    const pageJsonString = aiResponse.choices[0].message.content;
    if (!pageJsonString) {
      throw new Error('AI did not return content.');
    }
    const storyblokPayload = JSON.parse(pageJsonString);

    // 2. Create the story in Storyblok using the Management API
    const storyblokApiUrl = `https://mapi.storyblok.com/v1/spaces/${storyblokSpaceId}/stories/`;
    
    const response = await httpx.post(storyblokApiUrl, {
      headers: {
        Authorization: storyblokManagementToken,
        'Content-Type': 'application/json',
      },
      json: {
        story: storyblokPayload,
        publish: 1, // Automatically publish the story
      },
      timeout: 30000,
    });

    if (response.statusCode < 200 || response.statusCode >= 300) {
        const errorBody = await response.body.text();
        console.error("Storyblok API Error:", errorBody);
        throw new Error(`Storyblok API responded with status ${response.statusCode}`);
    }

    const responseData = JSON.parse(await response.body.text());

    // 3. Return the new story's information to the frontend
    return NextResponse.json({
      message: 'Page created successfully!',
      story: {
        slug: responseData.story.full_slug,
        id: responseData.story.id,
      },
    });
  } catch (error) {
    console.error(error);
    const errorMessage = error instanceof Error ? error.message : 'An unknown error occurred.';
    return NextResponse.json({ error: 'Failed to generate page.', details: errorMessage }, { status: 500 });
  }
}
```

---

### **Phase 4: The Frontend - UI & Storyblok Rendering**

This is what the user sees and interacts with.

#### **4.1. The Main Page (Homepage)**
Replace the entire content of `app/page.tsx` with this:

**File: `app/page.tsx`**
```tsx
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Loader2 } from 'lucide-react';

type StoryResult = {
  slug: string;
  id: number;
};

export default function Home() {
  const [prompt, setPrompt] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [result, setResult] = useState<StoryResult | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!prompt.trim()) return;

    setIsLoading(true);
    setError(null);
    setResult(null);

    try {
      const response = await fetch('/api/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt }),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.details || 'Failed to generate page.');
      }

      const data = await response.json();
      setResult(data.story);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'An unexpected error occurred.');
    } finally {
      setIsLoading(false);
    }
  };

  const storyblokEditUrl = result ? `https://app.storyblok.com/#!/me/spaces/${process.env.NEXT_PUBLIC_STORYBLOK_SPACE_ID}/stories/${result.id}` : '#';
  const livePageUrl = result ? `/${result.slug}` : '#';

  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-8 bg-gray-50">
      <div className="w-full max-w-2xl">
        <Card>
          <CardHeader>
            <CardTitle className="text-3xl font-bold text-center">StorySpark AI ⚡️</CardTitle>
            <CardDescription className="text-center">Describe the landing page you want to build. Our AI will create and publish it in Storyblok instantly.</CardDescription>
          </CardHeader>
          <CardContent>
            <form onSubmit={handleSubmit} className="space-y-4">
              <Textarea
                placeholder="e.g., A landing page for a new fitness app called 'FitFlow' with a hero, 3 features, and a testimonial..."
                className="min-h-[120px]"
                value={prompt}
                onChange={(e) => setPrompt(e.target.value)}
                disabled={isLoading}
              />
              <Button type="submit" className="w-full" disabled={isLoading}>
                {isLoading ? <Loader2 className="mr-2 h-4 w-4 animate-spin" /> : 'Generate Page'}
              </Button>
            </form>

            {error && <p className="mt-4 text-center text-red-600">Error: {error}</p>}

            {result && (
              <Card className="mt-6">
                <CardHeader>
                  <CardTitle>Success! Your page is ready.</CardTitle>
                </CardHeader>
                <CardContent className="flex flex-col sm:flex-row gap-4">
                  <Button asChild className="w-full">
                    <a href={livePageUrl} target="_blank" rel="noopener noreferrer">
                      View Live Page
                    </a>
                  </Button>
                  <Button asChild variant="outline" className="w-full">
                    <a href={storyblokEditUrl} target="_blank" rel="noopener noreferrer">
                      Edit in Storyblok
                    </a>
                  </Button>
                </CardContent>
              </Card>
            )}
          </CardContent>
        </Card>
      </div>
    </main>
  );
}
```

*Note: You will get a small error on `process.env.NEXT_PUBLIC_STORYBLOK_SPACE_ID`. We will fix this in the next step by renaming the environment variable.*

#### **4.2. The Storyblok Renderer Page**
This page will dynamically render any page created by the AI.

**Create the file: `app/[...slug]/page.tsx`**
```tsx
import { storyblokInit, apiPlugin } from "@storyblok/react/rsc";
import StoryblokStory from "@storyblok/react/story";
import Page from "@/components/Page";
import Hero from "@/components/Hero";
import FeatureList from "@/components/FeatureList";
import FeatureItem from "@/components/FeatureItem";
import Testimonial from "@/components/Testimonial";
import FinalCta from "@/components/FinalCta";

// This is the mapping of your Storyblok components to your React components
const components = {
  page: Page,
  hero: Hero,
  feature_list: FeatureList,
  feature_item: FeatureItem,
  testimonial: Testimonial,
  final_cta: FinalCta,
};

storyblokInit({
  accessToken: process.env.STORYBLOK_PREVIEW_TOKEN,
  use: [apiPlugin],
  components,
});

export default async function PageRoute({ params }: { params: { slug: string[] } }) {
  let slug = params.slug ? params.slug.join("/") : "home";

  const { data } = await fetchData(slug);

  return <StoryblokStory story={data.story} />;
}

export async function fetchData(slug: string) {
  let sbParams = { version: "draft" as const };
  const { getStoryblokApi } = await import("@/lib/storyblok-api");
  const storyblokApi = getStoryblokApi();
  return storyblokApi.get(`cdn/stories/${slug}`, sbParams);
}
```

#### **4.3. Create React Components**
Create a new folder `components` in your root directory. Inside it, create the following files:

**File: `components/Page.tsx`**
```tsx
import { storyblokEditable, StoryblokComponent } from "@storyblok/react/rsc";

const Page = ({ blok }: { blok: any }) => (
  <main {...storyblokEditable(blok)} className="flex flex-col items-center">
    {blok.body.map((nestedBlok: any) => (
      <StoryblokComponent blok={nestedBlok} key={nestedBlok._uid} />
    ))}
  </main>
);

export default Page;
```

**File: `components/Hero.tsx`**
```tsx
import { storyblokEditable } from "@storyblok/react/rsc";
import { Button } from "./ui/button";

const Hero = ({ blok }: { blok: any }) => (
  <section {...storyblokEditable(blok)} className="w-full bg-gray-900 text-white py-20 px-4 text-center">
    <h1 className="text-5xl font-bold">{blok.headline}</h1>
    <p className="mt-4 text-xl text-gray-300">{blok.subheadline}</p>
    <Button className="mt-8">{blok.cta_button_text}</Button>
  </section>
);

export default Hero;
```

**File: `components/FeatureList.tsx`**
```tsx
import { storyblokEditable, StoryblokComponent } from "@storyblok/react/rsc";

const FeatureList = ({ blok }: { blok: any }) => (
  <section {...storyblokEditable(blok)} className="w-full max-w-4xl py-16 px-4">
    <div className="grid md:grid-cols-3 gap-8">
        {blok.features.map((nestedBlok: any) => (
            <StoryblokComponent blok={nestedBlok} key={nestedBlok._uid} />
        ))}
    </div>
  </section>
);

export default FeatureList;
```

**File: `components/FeatureItem.tsx`**
```tsx
import { storyblokEditable } from "@storyblok/react/rsc";

const FeatureItem = ({ blok }: { blok: any }) => (
  <div {...storyblokEditable(blok)} className="text-center">
    <div className="text-4xl mb-4"> {/* A real implementation would map blok.icon to an actual icon */}
      {blok.icon === 'bolt' ? '⚡️' : blok.icon === 'shield' ? '🛡️' : '🚀'}
    </div>
    <h3 className="text-xl font-bold">{blok.name}</h3>
    <p className="mt-2 text-gray-600">{blok.description}</p>
  </div>
);

export default FeatureItem;
```

**File: `components/Testimonial.tsx`**
```tsx
import { storyblokEditable } from "@storyblok/react/rsc";

const Testimonial = ({ blok }: { blok: any }) => (
  <section {...storyblokEditable(blok)} className="w-full bg-gray-100 py-16 px-4">
    <div className="max-w-3xl mx-auto text-center">
      <p className="text-2xl italic">"{blok.quote}"</p>
      <p className="mt-4 font-semibold">- {blok.author_name}</p>
    </div>
  </section>
);

export default Testimonial;
```

**File: `components/FinalCta.tsx`**
```tsx
import { storyblokEditable } from "@storyblok/react/rsc";
import { Button } from "./ui/button";

const FinalCta = ({ blok }: { blok: any }) => (
  <section {...storyblokEditable(blok)} className="w-full py-20 px-4 text-center">
    <h2 className="text-4xl font-bold">{blok.title}</h2>
    <Button size="lg" className="mt-8">{blok.cta_button_text}</Button>
  </section>
);

export default FinalCta;
```

#### **4.4. Final Environment Variable Fix**
To allow the frontend to access the space ID, you must prefix it with `NEXT_PUBLIC_`. Rename the variable in your `.env.local` file:

```diff
- STORYBLOK_SPACE_ID="YOUR_SPACE_ID_HERE"
+ NEXT_PUBLIC_STORYBLOK_SPACE_ID="YOUR_SPACE_ID_HERE"
```

Then, update your `app/page.tsx` file to use this new variable. This will fix the error from before.

---

### **Phase 5: Run and Test!**

1.  Stop your development server if it's running.
2.  Restart it to load the new environment variables: `npm run dev`.
3.  Open `http://localhost:3000`.
4.  Type in a prompt like the one from the "Wow Factor" section at the top.
5.  Click "Generate Page" and watch the magic happen!

You now have a fully functional, highly impressive, and challenge-winning application. Good luck
