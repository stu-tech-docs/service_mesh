# Service Mesh Architecture Documentation Site

This repository contains the architecture assessment, trade-off analysis, technology comparison, and implementation runbook for adopting a **Service Mesh on Red Hat OpenShift (OCP)** across 40 microservices.

The site is authored in **pure Markdown (`.md`)** and configured for GitHub Pages using the built-in **Jekyll** static site generator with the [`just-the-docs`](https://just-the-docs.github.io/just-the-docs/) documentation theme.

---

## 📚 Documentation Pages

| File | Page Title | Description |
| :--- | :--- | :--- |
| **[`index.md`](index.md)** | Overview & Baseline | Architecture baseline, application snapshot, and current operational pain points. |
| **[`benefits.md`](benefits.md)** | Benefits Analysis | Canary steering, retiring ConfigMap URLs, mTLS, and Kiali observability. |
| **[`challenges.md`](challenges.md)** | Risks & Complexities | Data plane overhead (120+ sidecars), latency hops, Redis TCP connection pool gotchas. |
| **[`mesh-comparison.md`](mesh-comparison.md)** | Envoy vs. Consul vs. OSSM | Clarifying data plane vs. control plane and why Red Hat OSSM is the best fit for OCP. |
| **[`canary-routing.md`](canary-routing.md)** | Pilot & Canary Routing | JWT claims, `HttpOnly` cookies, and downstream context propagation across 40 services. |
| **[`roadmap.md`](roadmap.md)** | Adoption Roadmap | 4-phase rollout strategy, verification runbook commands, and readiness checklists. |

---

## 🚀 How to Publish to GitHub Pages

1. Push this repository to your GitHub account (e.g., `main` branch).
2. In GitHub, navigate to **Settings** &rarr; **Pages**.
3. Under **Build and deployment**:
   * **Source:** Select `Deploy from a branch`.
   * **Branch:** Select `main` (or `master`) and folder `/ (root)`.
4. Click **Save**.
5. GitHub Pages will automatically run its built-in Jekyll static site builder. Within 1–2 minutes, your documentation site will be live at:
   ```
   https://<your-github-username>.github.io/<repository-name>/
   ```

---

## 🛠️ Local Preview (Optional)

If you have Ruby and Bundler installed locally, you can run:

```bash
gem install bundler jekyll
bundle init
bundle add jekyll-remote-theme
bundle exec jekyll serve
```

Then visit `http://localhost:4000` in your browser.
