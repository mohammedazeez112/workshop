# 🎓 Student-AI-Hub — Product Requirements Document

> A Next.js AI-powered student productivity platform with flashcard generation, PDF summarization, placement insights, and an AI chatbot.

---

## 📐 Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 14 (App Router) |
| Database & Storage | Supabase (Postgres + Storage Bucket) |
| AI Models | Hugging Face Inference API |
| Dataset | Kaggle Student Placement Dataset |
| Chatbot LLM | Groq API (llama3-8b-8192 or mixtral-8x7b) |
| Styling | Tailwind CSS + shadcn/ui |
| Language | TypeScript |

---

## 🗂️ Project Structure

```
student-ai-hub/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                    # Landing / Login page
│   ├── dashboard/
│   │   └── page.tsx                # Main dashboard after login
│   ├── flashcards/
│   │   └── page.tsx                # Flashcard generator
│   ├── summarizer/
│   │   └── page.tsx                # PDF summarizer
│   ├── placement/
│   │   └── page.tsx                # Placement insights (Kaggle dataset)
│   └── chatbot/
│       └── page.tsx                # AI Chatbot
├── components/
│   ├── Navbar.tsx
│   ├── Sidebar.tsx
│   ├── FlashcardDeck.tsx
│   ├── PDFUploader.tsx
│   ├── PlacementChart.tsx
│   └── ChatWindow.tsx
├── lib/
│   ├── supabaseClient.ts           # Supabase client init
│   ├── huggingface.ts              # HuggingFace API helpers
│   ├── groq.ts                     # Groq API helper
│   └── placementData.ts            # Kaggle dataset loader/parser
├── public/
│   └── placement_data.csv          # Kaggle dataset (static import)
├── .env.local
└── README.md
```

---

## 🔐 Environment Variables (`.env.local`)

```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
HUGGINGFACE_API_KEY=your_huggingface_api_key
GROQ_API_KEY=your_groq_api_key
```

---

## 🗃️ Supabase Setup

### Table: `users`
> Simple login table — no Supabase Auth, just a basic username/password lookup.

```sql
create table users (
  id uuid default gen_random_uuid() primary key,
  username text unique not null,
  password text not null,         -- store plain or hashed (bcrypt recommended)
  created_at timestamp default now()
);
```

### Table: `uploaded_pdfs`
> Tracks PDFs uploaded by each user to the storage bucket.

```sql
create table uploaded_pdfs (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references users(id),
  file_name text not null,
  storage_path text not null,     -- path inside Supabase bucket
  uploaded_at timestamp default now()
);
```

### Storage Bucket: `student-pdfs`
- Bucket name: `student-pdfs`
- Access: **Public** (or authenticated via anon key)
- Accepted types: `application/pdf`
- Store path pattern: `{user_id}/{timestamp}_{filename}.pdf`

---

## 🔑 Feature 1 — Login / Register Page (`/`)

### Description
Simple login and register page. No Supabase Auth — just a direct table lookup.

### UI
- Centered card with app logo and name **"Student AI Hub"**
- Two tabs: **Login** | **Register**
- Fields: `Username`, `Password`
- On login: query `users` table → match username + password → store user in `localStorage` or React context
- On register: insert new row into `users` table
- On success: redirect to `/dashboard`

### Logic
```ts
// Login
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('username', username)
  .eq('password', password)
  .single();

if (data) { /* save to context, redirect */ }

// Register
await supabase.from('users').insert({ username, password });
```

---

## 🏠 Feature 2 — Dashboard (`/dashboard`)

### Description
Main hub after login. Shows overview cards with navigation to all features.

### UI
- Top Navbar with app name + logged-in username + logout button
- Left Sidebar with links: Dashboard, Flashcards, Summarizer, Placement Insights, Chatbot
- Main area: 4 feature cards with icon, title, short description, and a "Go" button
  - 📇 **Flashcard Generator** — Generate flashcards from any topic
  - 📄 **PDF Summarizer** — Upload PDF and get summary
  - 📊 **Placement Insights** — Explore student placement data
  - 🤖 **AI Chatbot** — Ask anything, powered by Groq

---

## 📇 Feature 3 — Flashcard Generator (`/flashcards`)

### Description
User types a topic or pastes text → AI generates flashcards (question + answer pairs).

### Hugging Face Models to use
- Primary: `google/flan-t5-large` — instruction-tuned, great for Q&A generation
- Alternative: `declare-lab/flan-alpaca-large`

### UI
- Text area: "Enter topic or paste notes"
- Dropdown: Select number of flashcards (5, 10, 15, 20)
- Button: **"Generate Flashcards"**
- Output: Flip cards — front shows Question, back shows Answer
- Cards animate on flip (CSS 3D transform)
- Navigation: Previous / Next buttons

### API Call (HuggingFace)
```ts
// lib/huggingface.ts
export async function generateFlashcards(text: string, count: number) {
  const prompt = `Generate ${count} flashcard question and answer pairs from the following text. 
  Format each as: Q: [question] A: [answer]
  Text: ${text}`;

  const response = await fetch(
    "https://api-inference.huggingface.co/models/google/flan-t5-large",
    {
      method: "POST",
      headers: { Authorization: `Bearer ${process.env.HUGGINGFACE_API_KEY}` },
      body: JSON.stringify({ inputs: prompt }),
    }
  );
  return response.json();
}
```

### Parsing Output
- Split response by newlines
- Match pattern `Q: ... A: ...`
- Render each as a card component

---

## 📄 Feature 4 — PDF Summarizer (`/summarizer`)

### Description
User uploads a PDF → it's stored in Supabase bucket → text is extracted → HuggingFace summarization model generates a summary.

### Hugging Face Models to use
- Primary: `facebook/bart-large-cnn` — best for summarization
- Alternative: `sshleifer/distilbart-cnn-12-6` (faster, lighter)

### UI
- Drag-and-drop / click-to-upload PDF zone
- Upload button → uploads to Supabase bucket → saves record to `uploaded_pdfs` table
- List of previously uploaded PDFs (fetched from `uploaded_pdfs` for current user)
- Click any PDF → "Summarize" button appears
- Summary displayed below in a scrollable card
- Copy to clipboard button

### Logic Flow
1. User selects PDF file
2. Upload to Supabase bucket at path `{user_id}/{timestamp}_{filename}.pdf`
3. Insert record into `uploaded_pdfs`
4. Extract text from PDF on the client using `pdfjs-dist` library
5. Send extracted text to HuggingFace summarization endpoint
6. Display summary

### PDF Upload to Supabase
```ts
const { data, error } = await supabase.storage
  .from('student-pdfs')
  .upload(`${userId}/${Date.now()}_${file.name}`, file);
```

### Summarization API Call
```ts
export async function summarizePDF(text: string) {
  const response = await fetch(
    "https://api-inference.huggingface.co/models/facebook/bart-large-cnn",
    {
      method: "POST",
      headers: { Authorization: `Bearer ${process.env.HUGGINGFACE_API_KEY}` },
      body: JSON.stringify({ inputs: text.slice(0, 1024) }), // BART token limit
    }
  );
  return response.json();
}
```

---

## 📊 Feature 5 — Placement Insights (`/placement`)

### Description
Visualize and explore the Kaggle student placement dataset. Students can see trends, filter data, and get a predicted placement probability for their own profile.

### Dataset
- Source: [Kaggle — Campus Recruitment Dataset](https://www.kaggle.com/datasets/benroshan/factors-affecting-campus-placement)
- File: `public/placement_data.csv`
- Key columns: `gender`, `ssc_p` (10th %), `hsc_p` (12th %), `degree_p`, `etest_p`, `mba_p`, `status` (Placed/Not Placed), `salary`

### UI
- **Section 1: Overview Stats**
  - Total students, % placed, average salary (from CSV data)
  - Shown as stat cards

- **Section 2: Charts** (use `recharts` or `chart.js`)
  - Bar chart: Placement rate by degree specialization
  - Pie chart: Gender distribution among placed students
  - Line/Scatter chart: MBA % vs Salary

- **Section 3: Predict My Placement**
  - Form inputs: 10th %, 12th %, Degree %, MBA %, Work Experience (Yes/No)
  - Button: "Predict"
  - Logic: Simple rule-based scoring OR call HuggingFace text classification model
  - Output: "High chance of placement 🎉" / "Moderate chance" / "Needs improvement"

### Dataset Loader
```ts
// lib/placementData.ts
import Papa from 'papaparse';

export async function loadPlacementData() {
  const res = await fetch('/placement_data.csv');
  const text = await res.text();
  const { data } = Papa.parse(text, { header: true, skipEmptyLines: true });
  return data;
}
```

---

## 🤖 Feature 6 — AI Chatbot (`/chatbot`)

### Description
A general-purpose AI chatbot for students — can answer academic questions, help with explanations, summarize topics, etc. Powered by Groq's fast inference API.

### Groq Model
- `llama3-8b-8192` (fast + capable)
- Fallback: `mixtral-8x7b-32768`

### System Prompt
```
You are a helpful AI study assistant for students. 
You help with academic questions, explain concepts clearly, 
assist with exam preparation, and provide study tips.
Keep responses concise and student-friendly.
```

### UI
- Full-height chat window
- Message bubbles: user (right, colored) and AI (left, white/grey)
- Input box at bottom with send button
- Loading indicator (typing animation) while waiting for response
- Clear chat button
- Chat history preserved in component state during session

### API Route (`app/api/chat/route.ts`)
```ts
import Groq from 'groq-sdk';

const groq = new Groq({ apiKey: process.env.GROQ_API_KEY });

export async function POST(req: Request) {
  const { messages } = await req.json();

  const completion = await groq.chat.completions.create({
    model: 'llama3-8b-8192',
    messages: [
      {
        role: 'system',
        content: 'You are a helpful AI study assistant for students...',
      },
      ...messages,
    ],
    max_tokens: 1024,
  });

  return Response.json({ 
    reply: completion.choices[0].message.content 
  });
}
```

### Frontend Chat Logic
```ts
const sendMessage = async (userMsg: string) => {
  const newMessages = [...messages, { role: 'user', content: userMsg }];
  setMessages(newMessages);
  setLoading(true);

  const res = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({ messages: newMessages }),
  });
  const { reply } = await res.json();
  setMessages([...newMessages, { role: 'assistant', content: reply }]);
  setLoading(false);
};
```

---

## 🎨 UI / Design Guidelines

- **Color Palette**: Deep navy `#0F172A` background, electric blue `#3B82F6` accent, white text
- **Font**: `Geist` or `Inter` for body, `Cal Sans` or `Sora` for headings
- **Components**: Use `shadcn/ui` for cards, buttons, inputs, tabs, dialogs
- **Layout**: Sidebar (fixed left 240px) + main content area
- **Responsive**: Mobile-friendly with collapsible sidebar
- **Dark Mode**: Default dark theme throughout

---

## 📦 NPM Packages to Install

```bash
npm install @supabase/supabase-js
npm install groq-sdk
npm install pdfjs-dist
npm install papaparse
npm install recharts
npm install shadcn/ui
npm install @radix-ui/react-tabs @radix-ui/react-dialog
npm install lucide-react
npm install clsx tailwind-merge
```

---

## 🚀 Page Routes Summary

| Route | Page |
|-------|------|
| `/` | Login / Register |
| `/dashboard` | Main Dashboard |
| `/flashcards` | Flashcard Generator |
| `/summarizer` | PDF Summarizer |
| `/placement` | Placement Insights |
| `/chatbot` | AI Chatbot |
| `/api/chat` | Groq Chatbot API Route |

---

## ✅ Build Order (Recommended for Vibe Coding)

1. Setup Next.js project + Tailwind + shadcn/ui
2. Setup Supabase client + create tables + storage bucket
3. Build Login/Register page (Feature 1)
4. Build layout: Navbar + Sidebar + Dashboard (Feature 2)
5. Build Chatbot page + API route (Feature 6) — easiest to test
6. Build Flashcard Generator (Feature 3)
7. Build PDF Summarizer with upload (Feature 4)
8. Build Placement Insights with CSV + charts (Feature 5)

---

## 🔒 Auth Guard (Simple)

Since there's no Supabase Auth, protect routes by checking localStorage:

```ts
// In each protected page
useEffect(() => {
  const user = localStorage.getItem('student_hub_user');
  if (!user) router.push('/');
}, []);
```

---

*Built with ❤️ for students, by students.*
