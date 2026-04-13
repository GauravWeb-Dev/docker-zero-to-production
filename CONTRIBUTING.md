# 🤝 Contributing to Docker-Notes

First off — thank you for taking the time to contribute! Every improvement, no matter how small, makes this resource better for the entire community.

---

## 📋 Ways to Contribute

- **Fix typos or grammar** — even small fixes are welcome
- **Improve explanations** — if something was confusing, help clarify it
- **Add examples** — real-world command examples are always valuable
- **Add new topics** — Docker Swarm, BuildKit, Kaniko, etc.
- **Report errors** — open an Issue if something is wrong
- **Translate** — help make this accessible in other languages

---

## 🔧 How to Contribute

### 1. Fork the Repository
Click the **Fork** button at the top-right of the repo page.

### 2. Clone Your Fork
```bash
git clone https://github.com/YOUR-USERNAME/Docker-Notes.git
cd Docker-Notes
```

### 3. Create a Branch
```bash
# Use a descriptive branch name
git checkout -b fix/typo-in-networking
git checkout -b feat/add-swarm-section
git checkout -b docs/improve-dockerfile-examples
```

### 4. Make Your Changes
- Follow the existing markdown formatting style
- Use code blocks with language identifiers (` ```bash `, ` ```yaml `, ` ```dockerfile `)
- Add emojis where appropriate (keeping it consistent with existing style)
- Test all commands before adding them

### 5. Commit
```bash
git add .
git commit -m "docs: fix typo in networking.md"

# Commit message format:
# docs: documentation changes
# fix: fixing an error
# feat: new content/section
# style: formatting only
```

### 6. Push and Open a PR
```bash
git push origin your-branch-name
```
Then open a Pull Request on GitHub with:
- A clear title
- Description of what you changed and why

---

## ✍️ Writing Style Guide

- **Language:** English primarily, Hinglish analogies welcome (matches existing style)
- **Tone:** Friendly, beginner-accessible, but technically accurate
- **Code blocks:** Always include realistic, runnable examples
- **Analogies:** Real-life analogies help beginners — use them
- **"Things to Remember"** sections at the end of each topic are encouraged

---

## 📁 Adding a New Topic

1. Create the folder: `mkdir -p XX-Topic-Name`
2. Create the file: `touch XX-Topic-Name/topic.md`
3. Follow the existing structure (concept → analogy → commands → use cases → tips)
4. Add it to the Table of Contents in `README.md`

---

## ❓ Questions?

Open a GitHub Issue with the `question` label. We're happy to help!

---

*Thank you for making this better for everyone. ⭐*
