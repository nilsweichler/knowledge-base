# VueJS Deployment to Netlify

## Initialize VueJS to a Git Repo
### Create a Repo

```bash
git init
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/your-name/your-repo.git
git push -u origin main
```
Repository created? Head to the next section!

## Add new site
### 1. Import an existing project
### 2. Connect to git provider
### 3. Choose the VueJS Repo
### Build Settings
Build command: `npm run build`

Publish directory: `dist`
### Some .env variables?
#### e.g. Coinflip Vue
Follow along:
- add **Key** `VUE_APP_API_URL` with **Value** `strapi.example.com`
- add **Key** `VUE_APP_PATH` with **Value** `/`

Congrats now you are done -> The Deployment will trigger automatically!