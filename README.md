# Multi-Account Backblaze B2 Cloudflare Worker (With CORS)

A Cloudflare Worker for accessing multiple Backblaze B2 accounts through a single endpoint using the S3-compatible API. Account credentials are stored securely in Cloudflare Worker secrets, while cache settings can be configured using environment variables.

## Features

- **Multiple B2 accounts:** Configure as many Backblaze B2 accounts as required.
- **S3-compatible access:** Connect to Backblaze B2 using S3-compatible endpoints and application keys.
- **Dynamic account configuration:** Store all account configurations in one `B2_ACCOUNTS` secret.
- **CORS support:** Cross-origin response handling implemented by the Worker.
- **Cloudflare edge caching:** Configure cache duration using `CACHE_TTL`.
- **Secure credential storage:** Keep real account credentials out of the source code and public GitHub repository.
- **Google Colab deployment:** Clone, configure, and deploy the Worker using Google Colab and Wrangler.
- **Flexible configuration:** Upload a private `.json` or `.txt` account file through the Colab workflow.

## Requirements

- A Cloudflare account with Workers access.
- One or more Backblaze B2 accounts.
- An S3-compatible application key for each account.
- A GitHub repository containing this project.
- Google Colab or a local environment with Node.js and Wrangler.
- A Cloudflare API token with the permissions required to deploy Workers and manage Worker secrets.

## Project Structure

```text
b2-multi-account-worker/
├── index.js
├── wrangler.toml
├── README.md
├── .gitignore
├── .dev.vars.example
├── b2_accounts.example.json
└── b2-multi-account-worker.ipynb
```

The notebook filename is an example. Use the actual filename if yours differs.

## Configuration

### 1. Configure `wrangler.toml`

Example:

```toml
name = "b2-multi-account-worker"
main = "index.js"
compatibility_date = "2026-10-15"

[vars]
CACHE_TTL = "86400"

[secrets]
required = ["B2_ACCOUNTS"]
```

Change the Worker name to the name you want to deploy.

`CACHE_TTL` is measured in seconds.

| Value | Cache duration |
|---:|---|
| `3600` | 1 hour |
| `21600` | 6 hours |
| `43200` | 12 hours |
| `86400` | 24 hours |
| `604800` | 7 days |
| `0` | Disables caching in the Worker logic |

If `CACHE_TTL` is not configured, the current Worker code uses `86400` seconds as its default.

**Important:** Keep the TTL in `wrangler.toml` consistent with the value you intend to deploy. A subsequent Wrangler deployment can change settings managed through the deployment configuration.

### 2. Configure B2 accounts

The Worker reads account configuration from the `B2_ACCOUNTS` secret.

The endpoint must use the format expected by this project's `index.js`. In the current example configuration, use the hostname **without `https://`**.

For example:

```json
{
  "account01": {
    "endpoint": "s3.us-west-004.backblazeb2.com",
    "region": "us-west-004",
    "keyId": "YOUR_KEY_ID_1",
    "applicationKey": "YOUR_APPLICATION_KEY_1",
    "label": "Account 01"
  },
  "account02": {
    "endpoint": "s3.us-west-004.backblazeb2.com",
    "region": "us-west-004",
    "keyId": "YOUR_KEY_ID_2",
    "applicationKey": "YOUR_APPLICATION_KEY_2",
    "label": "Account 02"
  }
}
```

The repository's `b2_accounts.example.json` provides a template for ten accounts.

Each account needs a unique account name, such as `account01`, `account02`, and `account03`. The account name is used in the Worker URL.

Replace the placeholder credentials with valid credentials for each account. Use the endpoint and region associated with that account's Backblaze B2 S3-compatible configuration.

**Security:** Never commit real application keys, account credentials, or the contents of your private `B2_ACCOUNTS` configuration to GitHub.

### 3. Upload your account configuration using Google Colab

You can prepare your private configuration as either:

- `b2_accounts.json`
- `my_accounts.json`
- `b2_accounts.txt`

A `.txt` file must still contain valid JSON.

The Colab workflow can:

1. Ask you to upload one `.json` or `.txt` file.
2. Rename the uploaded file to `b2_accounts.json`.
3. Validate the JSON structure and required account fields.
4. Upload the complete configuration to Cloudflare as the `B2_ACCOUNTS` secret.

Use your private configuration, not the public sample file with placeholder credentials.

Updating `B2_ACCOUNTS` replaces its previous value. Include every account you want to retain.

### 4. Set the `B2_ACCOUNTS` secret manually

Alternatively, use Wrangler from your project directory:

```bash
npx wrangler secret put B2_ACCOUNTS
```

Paste the complete JSON configuration when prompted.

The secret must belong to the same Worker that you intend to use. If you change the Worker name and deploy a different Worker, configure the secret for that Worker too.

### 5. Configure your Cloudflare API token

In Google Colab, enter your token without displaying it:

```python
import os
import getpass

os.environ["CLOUDFLARE_API_TOKEN"] = getpass.getpass(
    "Enter Cloudflare API Token: "
)
```

Do not print the token or commit it to GitHub. Use a token with only the permissions needed for your workflow.

## Deployment

### Deploy with Wrangler

Install Wrangler if necessary:

```bash
npm install -g wrangler
```

From the project directory, deploy the Worker:

```bash
wrangler deploy
```

Ensure that the `B2_ACCOUNTS` secret is configured for the intended Worker.

### Deploy from Google Colab

A typical workflow is:

1. Remove the previous local project directory if you want a clean clone.
2. Clone this GitHub repository.
3. Enter the project directory.
4. Optionally change `CACHE_TTL`.
5. Upload the private account configuration and update `B2_ACCOUNTS`.
6. Deploy with Wrangler.

Example clone commands:

```python
!rm -rf /content/b2-multi-account-worker
!git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git b2-multi-account-worker
%cd /content/b2-multi-account-worker
!ls -la
```

Replace the example URL with your repository URL.

**Warning:** The `rm -rf` command permanently deletes the named local directory in the current Colab runtime. It does not delete the GitHub repository or Cloudflare Worker.

## Changing the Cache TTL

The Worker reads the configured cache duration from:

```javascript
env.CACHE_TTL
```

The fallback in the current code is:

```javascript
env.CACHE_TTL || "86400"
```

For example, setting `CACHE_TTL` to `21600` configures a six-hour TTL.

You can change the value in either of these ways:

- **Cloudflare Dashboard:** Open Workers & Pages, select the Worker, and go to Settings → Variables and Secrets. Add or edit `CACHE_TTL` under Variables.
- **Wrangler:** Edit `CACHE_TTL` in the `[vars]` section of `wrangler.toml`, then deploy again.

You do not need to modify `index.js` just to change the TTL.

Setting `CACHE_TTL` to `0` disables caching through the TTL check in the current Worker logic. Actual cache behaviour can also depend on request type, response headers, and Cloudflare's caching rules.

## URL Structure

The Worker uses account-based routes. The following examples describe the expected route pattern; available operations depend on the implementation in `index.js`.

```text
https://YOUR_WORKER_DOMAIN/account01/
https://YOUR_WORKER_DOMAIN/account01/BUCKET_NAME/
https://YOUR_WORKER_DOMAIN/account01/BUCKET_NAME/path/to/file.mp4
```

Examples:

- `/account01/` — account-specific bucket-listing route.
- `/account01/BUCKET_NAME/` — bucket object-listing route.
- `/account01/BUCKET_NAME/path/to/file.mp4` — object-access route.

Replace `YOUR_WORKER_DOMAIN`, account names, bucket names, and object paths with your actual values.

## Caching Behaviour

The current Worker implementation uses Cloudflare's Cache API for eligible object GET requests when caching is enabled.

- Eligible successful object responses can be cached.
- Cache duration is controlled by `CACHE_TTL`.
- Conditional requests and range requests bypass the Worker cache in the current implementation.
- Bucket listings are not cached by this Worker logic.
- Setting `CACHE_TTL` to `0` disables caching through the TTL check.

Caching can also depend on response headers and Cloudflare platform rules. Do not assume every request is cached.

## GitHub and `.gitignore`

Use a `.gitignore` similar to this:

```gitignore
node_modules/
.dev.vars
.env
.wrangler/

# Private B2 account credentials
b2_accounts.json
b2_accounts.txt
```

Keep `b2_accounts.example.json` out of `.gitignore` so that the sample template remains visible in your public repository.

The sample file must contain placeholder credentials only. Do not put real credentials in the README, sample JSON, or notebook code.

Before publishing your Colab notebook, clear outputs that might contain tokens, credentials, or sensitive response data. `.gitignore` does not protect secrets embedded inside a notebook that you commit.

Adding a filename to `.gitignore` does not remove it if it was already committed to Git.

## Security Best Practices

- Store `B2_ACCOUNTS` as an encrypted Cloudflare Worker secret.
- Never commit real B2 application keys or Cloudflare API tokens.
- Keep `.dev.vars`, `.env`, `b2_accounts.json`, and `b2_accounts.txt` private.
- Use a Cloudflare API token with only the permissions necessary for your workflow.
- Rotate credentials if they are accidentally exposed.
- Verify B2 application key permissions and bucket access.
- Review your routes and access controls before exposing private files publicly.

## Troubleshooting

### Missing `B2_ACCOUNTS`

Confirm that the secret exists for the Worker configured in `wrangler.toml`.

### Incorrect account or bucket

Check the account identifier in the URL, bucket name, endpoint, region, and application key.

### HTTP 403 or authentication errors

Check the actual HTTP response and error body. Verify the endpoint format, account credentials, and B2 application key permissions. An HTTP 403 alone does not establish the exact cause.

### XML parsing error in the browser

Inspect the HTTP status and raw response body. The browser may be displaying an XML or error response that does not contain valid XML. The parsing message alone is not enough to identify the underlying problem.

### Cache duration does not change

Confirm that `CACHE_TTL` has the intended numeric value, that it is configured for the correct Worker, and that the latest deployment uses the expected configuration.

### Changes are not reflected after deployment

Confirm that you are in the correct project directory and that the Worker name in `wrangler.toml` matches the intended deployment.

## License

Choose a license before distributing this project. If you want others to use, modify, and redistribute the code, add an appropriate open-source license file.
