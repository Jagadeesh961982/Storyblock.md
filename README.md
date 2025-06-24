Of course! Thank you for the clarification. It's much more exciting to build a project from the ground up. The reference code is still an excellent model for what a high-quality, comprehensive solution looks like, and we can use its architectural patterns as inspiration.

Based on the challenge prompt, your goal is to create a "digital experience" that is creative, easy to use, and deeply integrated with Storyblok, with a high chance of winning the **"Amazing AI"** category.

Let's design a project that hits all those marks.

### The Winning Idea: "StorySpark AI" - From Prompt to Published Page

**The Concept:** A simple, elegant web application where a user types a prompt describing a landing page they want to create. The AI then designs the page structure, writes all the content, and **builds the page directly in the user's Storyblok space** by creating the necessary story and components.

**The "Wow" Factor & User Story:**
Imagine a marketer needs a landing page for a new "AI-Powered Cat Toy". Instead of fumbling with components and Lorem Ipsum, they go to your app and type:

> *"Create a landing page for 'PurrfectPlay', our new smart cat toy. It should have a bold hero section with a call to action, a section listing 3 key features with icons (self-driving, laser pointer, catnip dispenser), a testimonial from a happy cat owner named Mittens, and a final CTA to 'Buy Now'."*

Your app shows a loading spinner, and 60 seconds later, it returns:
1.  "✅ Your page 'PurrfectPlay' has been created in Storyblok!"
2.  A link to **edit the new page in the Storyblok Visual Editor**.
3.  A link to the **live, publicly accessible page**, rendered with the new content.

This is a complete, magical experience that directly solves a real problem and perfectly showcases the power of AI combined with Storyblok's structured content capabilities.

---

### Step-by-Step Game Plan to Build "StorySpark AI"

This plan breaks the project into manageable phases.

#### Phase 1: The Foundation (Storyblok & Project Setup)

This is the most crucial phase. Your AI needs a defined set of "Lego bricks" to build with.

1.  **Sign up for Storyblok:** Get a free community account.
2.  **Define Your Components in Storyblok:** This is your content schema. Create a small but versatile set of components. Don't go overboard.
    *   **`page` (Content Type):** This will be the main container for all pages. It should have one field:
        *   `body` (Type: `Blocks`)
    *   **`hero` (Nestable Component):**
        *   `headline` (Type: `Text`)
        *   `subheadline` (Type: `Text`)
        *   `cta_button_text` (Type: `Text`)
    *   **`feature_list` (Nestable Component):**
        *   `features` (Type: `Blocks`) - This will allow you to nest `feature_item` components inside.
    *   **`feature_item` (Nestable Component):**
        *   `name` (Type: `Text`)
        *   `description` (Type: `Textarea`)
        *   `icon` (Type: `Text`, e.g., for a simple icon library name like "bolt" or "shield")
    *   **`testimonial` (Nestable Component):**
        *   `quote` (Type: `Textarea`)
        *   `author_name` (Type: `Text`)
    *   **`final_cta` (Nestable Component):**
        *   `title` (Type: `Text`)
        *   `cta_button_text` (Type: `Text`)

3.  **Project Setup (The Tech Stack):**
    *   **Framework:** Use **Next.js** (App Router). It's perfect for this, as it handles frontend rendering and backend API routes in one project.
    *   **Styling:** Use **Tailwind CSS** and **shadcn/ui**. This will give you a beautiful, professional-looking UI with very little effort.
    *   **Storyblok Integration:**
        *   `@storyblok/react`: For fetching and rendering the generated pages on the front end.
        *   `httpx`: For making secure calls to the Storyblok Management API from your backend. **Do not use the Management API token on the client-side.**
    *   **AI:** Use the **OpenAI API** (or Claude's API). You'll need an API key.

#### Phase 2: The AI Core (The Backend API Route)

This is the brain of your application. Create a Next.js API route (e.g., `app/api/generate/route.ts`).

1.  **The Endpoint:** This route will accept a `POST` request containing the user's prompt.
2.  **The AI Prompting:** This is the secret sauce. You will send a detailed prompt to the LLM (e.g., GPT-4). Your prompt needs to instruct the AI to return a **structured JSON object** that matches the Storyblok components you defined.

    **Example System Prompt for the LLM:**
    ```
    You are an expert landing page designer and copywriter. Your task is to take a user's request and convert it into a structured JSON object that represents a Storyblok page.

    The available components are: `hero`, `feature_list`, `feature_item`, `testimonial`, `final_cta`.

    The final JSON output MUST follow this structure:
    {
      "name": "Page Name",
      "slug": "page-slug",
      "content": {
        "component": "page",
        "body": [
          // Array of component objects
        ]
      }
    }

    Here is an example of a component object for the 'hero' component:
    { "component": "hero", "headline": "...", "subheadline": "...", "cta_button_text": "..." }

    Based on the user's prompt, generate a compelling and complete page. Infer the content and structure. Keep slugs lowercase and hyphenated.
    ```
3.  **The Logic Flow:**
    *   Get the user's prompt from the request body.
    *   Call the OpenAI API with your system prompt and the user's prompt.
    *   Parse the JSON response from the AI. If it's not valid JSON, handle the error.
    *   Use `httpx` to make a `POST` request to the Storyblok Management API's `/v1/spaces/YOUR_SPACE_ID/stories/` endpoint. The body of this request will be the JSON you received from the AI.
    *   If the Storyblok API call is successful, return the new story's slug and Storyblok URL to the frontend.

#### Phase 3: The Digital Experience (The Frontend UI)

This is what the judges will see and use. Keep it clean and focused.

1.  **The Main Page:**
    *   A large, inviting text area for the user's prompt.
    *   A "Generate Page" button.
    *   A loading state (e.g., a spinner and some fun text like "Sparking creativity...").
2.  **The Results Display:**
    *   When the backend returns a success, display a confirmation message.
    *   Show two prominent buttons: "View Live Page" and "Edit in Storyblok".
    *   The "View Live Page" link will go to a dynamic route in your Next.js app (e.g., `/[slug]`) which uses `@storyblok/react` to render the page.

#### Phase 4: Polish and Submission

This phase turns a good project into a winning one.

*   **Error Handling:** What if the AI returns bad JSON? What if the Storyblok API fails? Show user-friendly error messages.
*   **Example Prompts:** Display 3-4 example prompts on the homepage so users know what's possible.
*   **A "Refine" Feature:** (Stretch Goal) After a page is generated, show the content in the UI and allow the user to give a follow-up prompt like "Make the headline more exciting" which triggers another AI call to just update that specific field.

### How This Project Wins the Challenge

*   **Use of Storyblok (10/10):** You are programmatically creating stories and nested components, which is a deep, meaningful use of Storyblok's core strength: structured content. You're also providing a link directly to the Visual Editor, highlighting a key feature.
*   **Creativity (10/10):** This is a highly creative and innovative use of AI. It goes beyond simple "text-to-text" and becomes "idea-to-structured-content."
*   **Complexity (9/10):** This involves a multi-step AI chain (prompt -> structured JSON), multiple API integrations (Your Frontend -> Your Backend -> OpenAI -> Storyblok), and a full-stack application.
*   **Ease of Use (10/10):** The user experience is the definition of simplicity. It abstracts away all the complexity of a CMS into a single text box.
*   **Amazing AI (Winner):** This is the quintessential "Amazing AI" project. It's not a gimmick; it's a genuinely useful tool that leverages AI to make a content creator's life drastically easier.

This project is ambitious but very achievable. Start with Phase 1, get your components and basic app running, then build the core AI logic in Phase 2. The rest is about making the experience shine. Good luck
