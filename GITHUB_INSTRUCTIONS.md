# GitHub Instructions for Claude

## The Problem

In Claude's computer environment, `git clone` and `git push` **do not work** due to a proxy/tunnel issue:

```
fatal: unable to access 'https://github.com/...': CONNECT tunnel failed, response 401
```

This happens even for public repos and even with a valid token.

## The Solution: Use the GitHub API

The GitHub REST API works via `curl`. You can read, create, update, and delete files this way.

---

## Repository Info

- **Repo**: `dsodol/claude_c_sharp_stuff`
- **URL**: https://github.com/dsodol/claude_c_sharp_stuff

---

## Required: Personal Access Token (PAT)

Ask the user for a GitHub Personal Access Token with **Contents: Read and write** permission.

- **Fine-grained token** (recommended): Select the specific repo and grant "Contents" read/write
- **Classic token**: Needs `repo` scope

---

## API Operations

### 1. Check if repo exists and get info

```bash
curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff"
```

### 2. List files in repo

```bash
curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents"
```

### 3. Get a specific file's content

```bash
curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/FILENAME"
```

The content is base64 encoded. Decode it:

```bash
curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/FILENAME" | jq -r '.content' | base64 -d
```

### 4. Create/Update a file (requires token)

```bash
# First, base64 encode the file content
CONTENT=$(base64 -w 0 /path/to/local/file)

# Then push via API
curl -X PUT \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Accept: application/vnd.github+json" \
  -d "{\"message\":\"Your commit message\",\"content\":\"$CONTENT\"}" \
  "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/FILENAME"
```

### 5. Update an existing file (requires SHA)

First get the file's current SHA:

```bash
SHA=$(curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/FILENAME" | jq -r '.sha')
```

Then update:

```bash
CONTENT=$(base64 -w 0 /path/to/local/file)

curl -X PUT \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Accept: application/vnd.github+json" \
  -d "{\"message\":\"Update commit message\",\"content\":\"$CONTENT\",\"sha\":\"$SHA\"}" \
  "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/FILENAME"
```

### 6. Delete a file (requires SHA and token)

```bash
# Get the SHA first
SHA=$(curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/FILENAME" | jq -r '.sha')

# Delete
curl -X DELETE \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Accept: application/vnd.github+json" \
  -d "{\"message\":\"Delete FILENAME\",\"sha\":\"$SHA\"}" \
  "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/FILENAME"
```

---

## Downloading All Files (Simulating Clone)

To "clone" the repo, fetch the file list and download each:

```bash
# Get list of files
FILES=$(curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents" | jq -r '.[].path')

# Download each file
mkdir -p /home/claude/claude_c_sharp_stuff
for FILE in $FILES; do
  curl -sL "https://api.github.com/repos/dsodol/claude_c_sharp_stuff/contents/$FILE" | jq -r '.content' | base64 -d > "/home/claude/claude_c_sharp_stuff/$FILE"
done
```

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `CONNECT tunnel failed, response 401` | Proxy blocks git protocol | Use GitHub API instead |
| `Resource not accessible by personal access token` | Token lacks write permission | User needs to regenerate token with Contents read/write |
| `Not Found` (404) | Repo is empty or file doesn't exist | Check repo contents first |
| `Validation Failed` / `sha missing` | Updating file without SHA | Fetch current SHA first |

---

## Security Reminder

Always remind the user to **revoke their token** after the session, since it was shared in the conversation.
