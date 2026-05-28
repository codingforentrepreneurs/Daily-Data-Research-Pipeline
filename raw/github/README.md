# Raw GitHub data with GitHub CLI

Commands to run:


Download the github cli [here](https://cli.github.com/)
```bash
gh auth login
```

Login to github
```bash
gh auth login
```

Create the directory
```bash
mkdir -p ~/dev/delacruz/raw/github
cd ~/dev/delacruz
```

Get my starred repos
```bash
gh api --paginate user/starred > raw/github/stars.json
```

Get my repos
```bash
gh api --paginate user/repos > raw/github/repos.json
```

Get top repos
```bash
gh search repos "stars:>50000" --limit 1000 --json name,url,description,stargazersCount,language,updatedAt > raw/github/top_repos.json
```

Get trending repos from the last 7 days
```bash
gh search repos "created:>$(date -v-7d +%Y-%m-%d) stars:>50" --limit 200 --json name,url,description,stargazersCount,language,updatedAt > raw/github/trending_recent.json
```