# kernel-news-oracle

**The old books saw it coming.** [kernel](https://kernel.sh)'s cloud browsers read the last few days of news. Then a GraphRAG library of the books I've read answers each story with the two passages that best anticipated it.

- **Live page:** the static showcase in `public/`, deployed on Vercel
- **The library itself:** [onlychr.is/raglib](https://onlychr.is/raglib) (book links on the page open it at the cited passage)

## How it works

1. **Read.** `news_oracle.py` opens one stealth kernel browser and drives it with Playwright over CDP. It reads the front pages of eight outlets (The Guardian, NPR, AP, BBC, Wired, CNN, ProPublica, Politico) and follows story links. It keeps stories published in the last N days, using meta tags, JSON-LD, `<time>` or the URL date. robots.txt is respected.
2. **Match.** Each story is embedded with OpenAI `text-embedding-3-large` and compared against the literary passages in the library's ChromaDB (`passage_match.py`). It's also matched to the nearest Leiden community from the GraphRAG index.
3. **Pick.** `pick_diverse` takes the strongest matches, drops a story another outlet already covered, and gives each story two books. No book is cited twice on the page.
4. **Explain.** Claude writes one sentence per pairing on what the old passage anticipates, or what it reframes when the link is loose.

The core of the kernel integration is one line. Existing Playwright code points at a kernel browser instead of a local one:

```python
kb = kernel.browsers.create(stealth=True)
browser = await playwright.chromium.connect_over_cdp(kb.cdp_ws_url)
# ... ordinary Playwright from here ...
kernel.browsers.delete_by_id(kb.session_id)
```

## Also in this repo

| File | What it is |
|---|---|
| `discover.py` | Crawls literary magazines with kernel and ranks essays against my reading taste (the library's Leiden communities). `news_oracle.py` reuses its kernel client and robots.txt helpers. |
| `raglib_mcp.py` | An MCP server that lets Claude Code (or any MCP client) search the library, list its topics and find passages. |
| `kernel_app/` | A kernel App, "library vs. the live web": asks the library a question and has a kernel browser read the web's answer in parallel. Deploy with `kernel deploy main.py`. |
| `news_oracle_results.json` | The raw output of the last crawl. |
| `public/` | The showcase page and the data it renders (`news.json`, enriched by the library's `/api/news`). |

## Running it

The crawl needs the library's ChromaDB (about 400MB), which isn't in this repo. With a copy:

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env    # add KERNEL_API_KEY, OPENAI_API_KEY, ANTHROPIC_API_KEY, CHROMA_PATH
python news_oracle.py --days 3 --top 10
```

To run the MCP server in Claude Code, add it to `.mcp.json`:

```json
{ "mcpServers": { "raglib": { "command": "venv/bin/python", "args": ["raglib_mcp.py"] } } }
```

Built with Claude Code.
