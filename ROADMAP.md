# Roadmap

## Now
- [ ] Win-back demo: n8n workflow live, site form connected, end-to-end test
- [ ] Export the win-back workflow into `n8n/workflows/`
- [ ] Deploy `site/` to Cloudflare Pages

## Next
- [ ] Shared front-end module so every demo uses the same form, loading and error handling
- [ ] SOP generator demo (page section + n8n workflow)
- [ ] Feedback miner demo
- [ ] Move rate limiting from n8n static data to a proper store (Sheet or KV)

## Later
- [ ] Cloudflare Pages Function as a thin proxy in front of n8n (hide the webhook URL, add a Turnstile check)
- [ ] Simple admin page: demo runs per day, top industries, conversion to enquiry
- [ ] Automated tests for the workflow JSON contracts (request/response shapes)
