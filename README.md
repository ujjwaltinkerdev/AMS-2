# AMS-2: Prior Authorization Multi-Agent System

Complete architectural documentation for the PA system using RAG (Retrieval-Augmented Generation) with state laws, payer policies, and historical denial/appeal outcomes.

## 📚 Documentation

- **[State Laws + Payer Policies Training Architecture](docs/README.md)** — Core technical design covering fine-tune vs RAG decision, knowledge base architecture, and deployment strategy

## 🚀 Quick Start

This project uses Docsify for documentation. To view locally:

```bash
npm install -g docsify-cli
docsify serve docs
```

Then open `http://localhost:3000`

## 🌐 Deployed Site

The documentation is deployed to Netlify:
- **URL:** `https://your-domain.netlify.app/` (after deployment)
- **Build Command:** None required (static site)
- **Publish Directory:** `docs/`

## 📋 Files

```
docs/
├── index.html          # Docsify bootstrap
├── README.md           # Main documentation (converted from MD)
└── _sidebar.md         # Navigation sidebar

.gitignore             # Git ignore rules
README.md              # This file
```

## 🔧 Technologies

- **Docsify** — Zero-build markdown documentation
- **Netlify** — Free static site hosting
- **GitHub** — Version control

## 📖 Document Sections

1. **Pilot Scope** — Michigan + Texas, 2-state pilot, scalable to 50 states
2. **The Problem** — Four variability dimensions (state law, payer policy, patient circumstance, past outcomes)
3. **Fine-Tune vs RAG** — Why RAG wins (9-1 comparison)
4. **Architecture** — Three knowledge bases, metadata filtering, query flow
5. **Variability Handling** — How each dimension is handled via retrieval
6. **Appeal/Re-Appeal Workflow** — Using all three KBs for appeal suggestions
7. **Ingestion Pipeline** — How data flows into the system
8. **Cost Analysis** — RAG vs Fine-Tuning (~500× cheaper)
9. **Best Practices** — Citation enforcement, metadata filtering, drift monitoring
10. **Summary** — Architectural verdict and implementation plan

## 🛠️ Setup Instructions

### Step 1: Create GitHub Repository

```bash
git remote add origin https://github.com/YOUR_USERNAME/AMS-2.git
git branch -M main
git add .
git commit -m "Initial commit: PA system architecture documentation"
git push -u origin main
```

### Step 2: Deploy to Netlify

1. Go to [netlify.com](https://netlify.com)
2. Click **"New site from Git"**
3. Select your GitHub repository
4. **Build command:** (leave empty)
5. **Publish directory:** `docs`
6. Click **Deploy**

Your site will be live at `https://YOUR-SITE.netlify.app`

### Step 3: Custom Domain (Optional)

In Netlify Settings → Domain Management, add your custom domain.

## 📝 License

Internal documentation — AMS-2 Project

## 👤 Author

Generated for AMS billing & prior authorization system.

---

**Last Updated:** 2026-05-14
