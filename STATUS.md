# Status

This is the **formalia template repository**. No live work happens
here.

To start a new attempt:

```sh
gh repo create --template <upstream-owner>/formalia \
     <your-username>/<problem-name> --private --clone
cd <problem-name>

# One-time per-clone setup — fills in git identity and substitutes
# the GH_USERNAME / GIT_USER_NAME / GIT_USER_EMAIL placeholders.
make init

# In Claude Code:
/target
# Then (self-looping; bare = back-to-back, or /solve every 30m, n=5, once):
/solve
```

`/target` overwrites this file with the per-clone initial state.
