# GitLab CI/CD on EC2 — Candidate Setup Guide

Oct 24, 2026 · @Satyam

Follow these steps in order. By the end, every `git push` to your default branch will build a Docker image, push it to Docker Hub, deploy it on your EC2 instance and check that it is running.

## Before you start

**What you will build:** `git push` → GitLab creates a pipeline → your EC2 runner runs 4 stages: **build → push → deploy → verify** → your app is live on port 9092.

**You need:**

- [ ] A GitLab.com project with your app code and a `Dockerfile` in the root
- [ ] An AWS EC2 instance running **Ubuntu 22.04 or 24.04**, with SSH access
- [ ] A Docker Hub account and a **personal access token** (Docker Hub → Account Settings → Personal access tokens → *Read & Write*). Do not use your password.
- [ ] Git installed on your laptop

**How to use this guide:** each step has **Do** (what to run or click) and **Check** (what you should see). Do not move to the next step until the check passes.

> Replace anything in `<ANGLE_BRACKETS>` with your own value.

## Step 1 — Prepare your GitLab project

**Do:**

1. Open your project and go to **Settings → Repository → Branch defaults**. Note the default branch name (`main` or `master`).
2. Go to **Settings → CI/CD → Runners** and turn **off** "Enable instance runners for this project".
3. Open your `Dockerfile` and note the port in the `EXPOSE` line (for example `EXPOSE 80`).

**Check:** write these down — you will need them later.

| Item | Your value |
| --- | --- |
| Default branch |  |
| Container port (`EXPOSE`) |  |
| EC2 public IP |  |
| Docker Hub username |  |

> Why turn off instance runners? GitLab's shared runners do not have access to your EC2, so a job that lands on them cannot deploy your app.

## Step 2 — Install Docker and GitLab Runner on EC2

SSH into your instance first:

```bash
ssh -i <your-key.pem> ubuntu@<EC2_PUBLIC_IP>
```

**Do — 2a. Install Docker:**

```bash
sudo apt update -y
sudo apt install -y docker.io git curl
sudo systemctl enable --now docker
```

**Do — 2b. Install GitLab Runner:**

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh -o script.deb.sh
sudo bash script.deb.sh
sudo apt install -y gitlab-runner
sudo systemctl enable --now gitlab-runner
```

**Do — 2c. Give Docker access to both users:**

```bash
sudo usermod -aG docker ubuntu
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
exit
```

Then SSH in again so your new group membership takes effect.

**Check:**

```bash
docker --version
gitlab-runner --version
docker run --rm hello-world
sudo -u gitlab-runner -H docker ps
```

- `hello-world` prints **Hello from Docker!**
- The last command prints an empty table header (`CONTAINER ID   IMAGE ...`) with **no** `permission denied` error.

> Always type `-aG`, never `-G` alone — without `-a` the user is removed from all other groups. Never run `chmod 666 /var/run/docker.sock`.

## Step 3 — Create a project runner in GitLab

**Do:**

1. Go to **Settings → CI/CD → Runners → New project runner**.
2. Fill in:
   - **Tags:** `ec2`
   - **Run untagged jobs:** leave **unchecked**
   - **Runner description:** `ec2-shell-runner`
3. Click **Create runner**.
4. On the next page choose **Linux** and copy the token that starts with `glrt-`.

**Check:** you have a token like `glrt-xxxxxxxxxxxx` copied.

> Treat the token like a password. Do not paste it in chat or commit it to the repo.

## Step 4 — Register the runner on EC2

**Do:** run this on your EC2 instance (paste your own token):

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com" \
  --token "<YOUR_GLRT_TOKEN>" \
  --executor "shell"
```

Do **not** add `--description` or `--tag-list` — those come from Step 3.

**Check:**

```bash
sudo gitlab-runner list
sudo gitlab-runner verify
```

- `list` shows your runner with `Executor=shell`.
- `verify` says the runner **is valid**.
- In GitLab, **Settings → CI/CD → Runners** shows your runner with a **green dot** (online) and the tag `ec2`.

## Step 5 — Add CI/CD variables

**Do:** go to **Settings → CI/CD → Variables → Add variable** and create:

| Key | Value | Visibility | Protect variable |
| --- | --- | --- | --- |
| `DOCKER_USER` | Your Docker Hub username | Visible | Yes |
| `DOCKER_PASS` | Your Docker Hub access token | Masked | Yes |

**Check:** both variables appear in the list, and `DOCKER_PASS` shows as masked.

> Never write your token in `.gitlab-ci.yml`. Anything committed stays in Git history even after you delete it.

> "Protect variable" means only pipelines on protected branches can read it. Your default branch is protected by default — if you changed that, the login in Step 6 will fail.

## Step 6 — Add the pipeline and run it

**Do — 6a.** In the root of your repo, create a file named exactly `.gitlab-ci.yml`. If your `Dockerfile` exposes a port other than 80, change `CONTAINER_PORT`.

```yaml
stages:
  - build
  - push
  - deploy
  - verify

workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

default:
  tags: [ec2]

variables:
  IMAGE: "$DOCKER_USER/notes-app"
  TAG: "$CI_COMMIT_SHORT_SHA"
  CONTAINER_NAME: "notes-app-container"
  APP_PORT: "9092"
  CONTAINER_PORT: "80"

build_image:
  stage: build
  script:
    - docker build -t $IMAGE:$TAG -t $IMAGE:latest .

push_to_dockerhub:
  stage: push
  script:
    - echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
    - docker push $IMAGE:$TAG
    - docker push $IMAGE:latest
  after_script:
    - docker logout

deploy_container:
  stage: deploy
  script:
    - docker rm -f $CONTAINER_NAME || true
    - docker run -d --name $CONTAINER_NAME --restart unless-stopped -p $APP_PORT:$CONTAINER_PORT $IMAGE:$TAG

verify_app:
  stage: verify
  script:
    - |
      for i in $(seq 1 10); do
        echo "Attempt $i..."
        if curl -sf http://localhost:$APP_PORT > /dev/null; then
          echo "App is up!"
          exit 0
        fi
        sleep 5
      done
      echo "App did not respond"
      docker logs --tail 50 $CONTAINER_NAME
      exit 1
```

**Do — 6b.** Use spaces for indentation, never tabs. Optionally paste the file into **Build → Pipeline editor** to validate it.

**Do — 6c.** Commit and push to your default branch:

```bash
git add .gitlab-ci.yml
git commit -m "Add CI/CD pipeline"
git push origin <DEFAULT_BRANCH>
```

**Check:**

- **Build → Pipelines** shows a new pipeline with 4 stages, all green.
- Open any job log: near the top it says `Running with gitlab-runner ... on ec2-shell-runner`.
- Docker Hub shows `<DOCKER_USER>/notes-app` with two tags: a commit ID (like `a1b2c3d4`) and `latest`.
- On EC2, `docker ps` shows `notes-app-container` with `0.0.0.0:9092->80/tcp`.

## Step 7 — Open the port and test in your browser

**Do:** in the AWS console, open your instance's **Security Group → Edit inbound rules** and make sure these rules exist:

| Type | Port | Source |
| --- | --- | --- |
| SSH | 22 | My IP |
| Custom TCP | 9092 | 0.0.0.0/0 |

**Check:**

```bash
curl -i http://localhost:9092             # on EC2
curl -i http://<EC2_PUBLIC_IP>:9092       # on your laptop
```

- Both return `HTTP/1.1 200 OK`.
- `http://<EC2_PUBLIC_IP>:9092` opens your app in the browser.

> If the app works on EC2 but not from your laptop, the Security Group rule is missing. Your public IP changes if you stop and start the instance.

## Troubleshooting

Read the last red line in the failed job's log first, then find it below.

| What you see | Likely cause | Fix |
| --- | --- | --- |
| No pipeline appears after push | Pushed to a non-default branch, or YAML error | Push to the default branch; check the file in **Build → Pipeline editor** |
| Job stuck in **pending** | Runner offline or tag mismatch | `sudo gitlab-runner verify`; confirm the runner has tag `ec2` |
| Job ran on a runner that is not yours | Instance runners still enabled | Step 1, item 2 |
| `permission denied ... /var/run/docker.sock` | `gitlab-runner` not in the `docker` group | Step 2c, then `sudo systemctl restart gitlab-runner` |
| `docker: command not found` | Docker not installed | Step 2a |
| `unauthorized: incorrect username or password` | Wrong `DOCKER_USER` / `DOCKER_PASS`, or branch not protected | Recreate the token; check Step 5 |
| `port is already allocated` | Something else uses 9092 | `docker ps -a` and `sudo ss -tlnp \| grep 9092` |
| `verify_app` fails with "App did not respond" | Wrong `CONTAINER_PORT`, or the app crashed | Check `docker logs notes-app-container`; match `EXPOSE` |
| Works on EC2, times out from laptop | Security Group port closed | Step 7 |

Still stuck? Share the **job log link** and the output of `docker ps -a` with your trainer.

## Graded exercises

Start these only after Step 7 passes.

### Exercise A — Check for a valid JSON response (10 marks)

**Do:**

1. Find a URL in your app that returns JSON (for example `/api/health`).
2. Add `HEALTH_PATH: "/api/health"` (your path) under `variables:`.
3. In `verify_app`, change the `curl` line to:

```bash
if curl -sf http://localhost:$APP_PORT$HEALTH_PATH | python3 -m json.tool > /dev/null; then
```

**Check:** the `verify_app` log prints `App is up!`. Then break the path on purpose (for example `/api/wrong`) and confirm the job fails.

### Exercise B — Slack alert on success and failure (10 marks)

**Do:**

1. In Slack, create an **Incoming Webhook** for a channel and copy its URL.
2. Add a GitLab variable `SLACK_WEBHOOK_URL` (Masked, Protected).
3. Add these two jobs at the end of `.gitlab-ci.yml`:

```yaml
notify_success:
  stage: .post
  when: on_success
  script:
    - >
      curl -sf -X POST -H 'Content-type: application/json'
      --data "{\"text\":\"Deployed $CI_PROJECT_NAME $TAG: $CI_PIPELINE_URL\"}"
      "$SLACK_WEBHOOK_URL"

notify_failure:
  stage: .post
  when: on_failure
  script:
    - >
      curl -sf -X POST -H 'Content-type: application/json'
      --data "{\"text\":\"Pipeline failed for $CI_PROJECT_NAME: $CI_PIPELINE_URL\"}"
      "$SLACK_WEBHOOK_URL"
```

**Check:** a normal push posts a success message. Then set `CONTAINER_PORT` to a wrong value, push, and confirm a failure message arrives. Set it back afterwards.

## Submission checklist

Submit screenshots or links for each item; total 60 marks.

| Task | Proof | Marks |
| --- | --- | --- |
| Runner registered and online | Runners page showing green dot and `ec2` tag | 10 |
| Full pipeline green, app opens in browser | Pipeline page + browser screenshot | 20 |
| Docker permission issue fixed | Failed job log + passing job log | 10 |
| JSON verify (Exercise A) | `verify_app` job log | 10 |
| Slack alerts (Exercise B) | Screenshot of both Slack messages | 10 |

**For the permission task:** run `sudo gpasswd -d gitlab-runner docker` and `sudo systemctl restart gitlab-runner`, push to see the failure, then fix it with Step 2c.

## Cleanup (after grading)

- [ ] Stop or terminate your EC2 instance
- [ ] Revoke your Docker Hub access token
- [ ] Delete the runner in **Settings → CI/CD → Runners**
- [ ] Delete the Slack webhook if you no longer need it
